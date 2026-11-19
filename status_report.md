At the time of writing this, I realized I am actually commiting this to my personal copy of the project instead of the NCSU one I explicelty made for this, I do appologize. As proof this is mine, my name is Lliam Rankins, this is for the DevOps CSC 519 project.

## Statement on Progress

![Picture of the Test-Box CI/CD pipeline](media\Test-Box_Pipeline.png)

Current progress on the Test-Box application is going well. The distributed pipeline is able to handle linting, testing, and publishing of all projects its connected to.

I am however having issues with deployment as most options seem too fragile or convoluted due to the universal nature of the CI/CD pipeline. However I will be spending the rest of this and next week working on it to ensure that everything works as intended.

## Link to Commits

Lliam Rankins - [Main Commit with Lint/Test/Publish implimentation](https://github.com/Lliam-Rankins/test-box/commit/8f09acb970fa979ebd67bbb070c4c345c2902ef8)

I was responsible for implimenting the distributed/universal logic for a central CI/CD pipeline. With the goal of easily hooking in other projects on a github account to quickly add a fully fleshed out pipeline to each one. So far the first three portions are complete, successfully linting, testing, and publishing.

## Blockers

The main blocker at the moment is just finding a simple/easy way to not just switch traffic from one spun up docker container to another(I plan on using nginx, but likely still with ansible to impliment changes), but automatically getting this setup when a new project is added to the test-box pipeline. I'm having trouble with a straight forward solution here as all the ones I've found seem quite fragile in their structure. This likely just needs more research and time.

## Project Modifications

The only modification is that the only deployment that will be supported at this time is for web applications. Considering the issues with fragile deployment structures as is, expanding it to be more than web applications would make it even more so. To be clear though, Test-Box will still be able to lint, test, and publish for any type of project(assuming it is a public repository).
