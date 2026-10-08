# test-box

A centralized CI/CD infrastructure project that runs tests for multiple application repositories on a self-hosted Oracle Linux runner using ephemeral Docker containers.

Instead of writing and maintaining separate CI pipelines and runner configurations across every project, `test-box` acts as a central hub. Application repositories (spokes) only need a minimal trigger workflow and a Dockerfile. All build, execution, and cleanup logic lives here.

## Architecture

This project uses a hub-and-spoke model:

- **The Hub (`test-box`):** Contains the reusable GitHub Actions workflow (`.github/workflows/central-test.yml`) and the Ansible playbooks used to provision the execution node.
- **The Execution Node (Oracle Linux VM):** An Oracle Linux instance running Docker and self-hosted GitHub Actions Runner services managed by systemd.
- **The Spokes (Application Repositories):** Standard application repositories that contain only application code, a `Dockerfile`, and a trigger workflow that calls the central test workflow.

### Security and Isolation

- All test suites run inside ephemeral Docker containers (`docker run --rm`). Application code never runs directly on the host operating system.
- Docker images are tagged with the caller repository's commit SHA (`test-box-${{ github.sha }}`) and removed immediately after the test run finishes (`docker rmi`).
- Runner services run under the `docker` user group to allow container execution without requiring `sudo`.

## Requirements

Before provisioning the runner, ensure you have:

- An Oracle Linux 9 instance (x86_64 or ARM64 Ampere A1) with SSH access.
- Ansible installed on your local machine.
- The `monolithprojects.github_actions_runner` Ansible role installed locally:
    ```bash
    ansible-galaxy install monolithprojects.github_actions_runner
    ```
- A GitHub Personal Access Token (PAT) with `Administration: Read and Write` permissions for all repositories where runners will be registered.

## Repository Setup

### 1. Configure Inventory

Copy the example inventory file and add your server details:

```bash
cp hosts.example.yaml hosts.yaml
```

Edit `hosts.yaml` with your server IP, SSH user (default for Oracle Linux is `opc`), and SSH private key path.

### 2. Configure Secrets

Copy the secrets template:

```bash
cp secrets.template.yaml secrets.yaml
```

Edit `secrets.yaml` with your GitHub account name and Personal Access Token:

```yaml
access_token: "YOUR_GITHUB_PAT"
github_account: "ACCOUNT_NAME"
```

You can optionally encrypt this file using Ansible Vault:

```bash
ansible-vault encrypt secrets.yaml
```

### 3. Configure Target Repositories

Edit `repositories.yml` to list the repositories that should have runners deployed on this machine:

```yaml
repositories:
    - "REPOSITORY_NAME"
    - "REPOSITORY_NAME"
    - "REPOSITORY_NAME"
```

### 4. Run the Provisioning Playbooks

First, update the host packages:

```bash
ansible-playbook -i hosts.yaml 0-update-security.yaml
```

Next, install Docker, configure user permissions, and deploy the runner services:

```bash
ansible-playbook -i hosts.yaml 1-setup-test-box.yaml
```

Each repository listed in `repositories.yml` will get an isolated runner instance deployed under `/opt/actions-runner/<repo-name>` with its own systemd service.

## Connecting an Application Repository

To connect a new or existing repository to the test-box system:

### 1. Add the Repository to Ansible

Add the repository name to `repositories.yml` and run `1-setup-test-box.yaml` to register and start its runner. Ensure your GitHub PAT has access to the new repository.

### 2. Add the Spoke Trigger Workflow

In your application repository, create `.github/workflows/test-box.yml`:

```yaml
name: Test-Box

on:
    push:
        branches: [master, main]
    pull_request:
        branches: [master, main]

jobs:
    Call-Test-Box:
        uses: ACCOUNT_NAME/test-box/.github/workflows/central-test.yml@master
```

### 3. Add a Dockerfile

In the root of your application repository, create a `Dockerfile` that defines your runtime environment, copies your application files, and runs your test framework:

```dockerfile
FROM python:3.12-slim

WORKDIR /opt/app

COPY . .

RUN pip install pytest

CMD ["python", "-m", "pytest"]
```

optionally, make the docker file for both production and testing by adding/changing the following to the docker file:

```dockerfile
ENV TEST="false"
CMD if [ "$TEST" = "true" ]; then ["python", "-m", "pytest"]; fi
```

When code is pushed or a pull request is opened in the application repository:

1. GitHub triggers the spoke workflow.
2. The job routes to the self-hosted runner on the Oracle VM.
3. The runner checks out the calling repository code at the triggering commit SHA.
4. The runner builds the Docker image.
5. The runner executes the container.
6. The test framework exits with `0` (pass) or `1` (fail), which reports back to GitHub.
7. The runner removes the Docker image.
