# docker-runner
	1. Request docker desktop installation with service now
	2. Install Vscode :IDE https://code.visualstudio.com/download#
	3. Install Vscode -Extensions Docker,YAML
Follow the sections in 

https://dev.to/pwd9000/create-a-docker-based-self-hosted-github-runner-linux-container-48dh![image](https://user-images.githubusercontent.com/19489833/212598988-778cdd87-5cc6-4b00-802b-d2b728e7b7cd.png)

		Open VSCode, you can clone the repo found on my GitHub project docker-github-runner-linux which contains all the files or simply follow along with the following steps. 
		We will prepare a script that will be needed as part of our docker image creation.
		Create a root folder called docker-github-runner-linux and then another sub folder called scripts. Inside of the scripts folder you can create the following script:
		
		start.sh
		This script will be used as our 'ENTRYPOINT' script and will be used to bootstrap our docker container when we start/run a container from the image we will be creating. The main purpose of this script is to register a new self hosted GitHub runner instance on the repo we pass into the docker environment each time a new container is spun up or scaled up from the image.
		
		
		#!/bin/bash
		GH_OWNER=$GH_OWNER
		GH_REPOSITORY=$GH_REPOSITORY
		GH_TOKEN=$GH_TOKEN
		RUNNER_SUFFIX=$(cat /dev/urandom | tr -dc 'a-z0-9' | fold -w 5 | head -n 1)
		RUNNER_NAME="dockerNode-${RUNNER_SUFFIX}"
		REG_TOKEN=$(curl -sX POST -H "Accept: application/vnd.github.v3+json" -H "Authorization: token ${GH_TOKEN}" https://api.github.com/repos/${GH_OWNER}/${GH_REPOSITORY}/actions/runners/registration-token | jq .token --raw-output)
		cd /home/docker/actions-runner
		./config.sh --unattended --url https://github.com/${GH_OWNER}/${GH_REPOSITORY} --token ${REG_TOKEN} --name ${RUNNER_NAME}
		cleanup() {
		    echo "Removing runner..."
		    ./config.sh remove --unattended --token ${REG_TOKEN}
		}
		trap 'cleanup; exit 130' INT
		trap 'cleanup; exit 143' TERM
		./run.sh & wait $!
		
		##Prepare dockerfile to build image (Linux)
		Now with our scripts ready, we can get to the fun part... Building the linux docker image. Navigate back to the root folder and create a file called: dockerfile:
		
		Let's take a closer look and see what this docker build file will actually do, step by step:
		
		
		# base image
		FROM ubuntu:20.04
		#input GitHub runner version argument
		ARG RUNNER_VERSION
		ENV DEBIAN_FRONTEND=noninteractive
		LABEL Author="Dileep"
		LABEL Email="pwd9000@hotmail.co.uk"
		LABEL GitHub="https://github.com/Dileep-github"
		LABEL BaseImage="ubuntu:20.04"
		LABEL RunnerVersion=${RUNNER_VERSION}
		# update the base packages + add a non-sudo user
		RUN apt-get update -y && apt-get upgrade -y && useradd -m docker
		# install the packages and dependencies along with jq so we can parse JSON (add additional packages as necessary)
		RUN apt-get install -y --no-install-recommends \
		    curl nodejs wget unzip vim git azure-cli jq build-essential libssl-dev libffi-dev python3 python3-venv python3-dev python3-pip
		# cd into the user directory, download and unzip the github actions runner
		RUN cd /home/docker && mkdir actions-runner && cd actions-runner \
		    && curl -O -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
		    && tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
		# install some additional dependencies
		RUN chown -R docker ~docker && /home/docker/actions-runner/bin/installdependencies.sh
		# add over the start.sh script
		ADD scripts/start.sh start.sh
		# make the script executable
		RUN chmod +x start.sh
		# set the user to "docker" so all subsequent commands are run as the docker user
		USER docker
		# set the entrypoint to the start.sh script
		ENTRYPOINT ["./start.sh"]
		
		
		# base image
		FROM ubuntu:20.04
		
		The 'FROM' instruction will tell our docker build to fetch and use an Ubuntu 20.04 OS base image. We will add additional configuration to this base image next.
		
		
		#input GitHub runner version argument
		ARG RUNNER_VERSION
		ENV DEBIAN_FRONTEND=noninteractive
		LABEL Author="Dileep"
		LABEL Email="pwd9000@hotmail.co.uk"
		LABEL GitHub="https://github.com/Dileep-github"
		LABEL BaseImage="ubuntu:20.04"
		LABEL RunnerVersion=${RUNNER_VERSION}
		
		We define an input argument using 'ARG'. This is so that we can instruct the docker build command to load a specific version of the GitHub runner agent into the image when building the image. Because we are using a linux container, 'ARG' will create a system variable $RUNNER_VERSION which will be accessible to Bash inside the container.
		We also set an Environment Variable called DEBIAN_FRONTEND to noninteractive with 'ENV', this is so that we can run commands later on in unattended mode.
		In addition we can also label our image with some metadata using 'LABEL' to add more information about the image. You can change these values as necessary.
		NOTE: 'LABEL RunnerVersion=${RUNNER_VERSION}', this label is dynamically updated from the build argument we will be passing into the docker build command later.
		
		
		
		
		# update the base packages + add a non-sudo user
		RUN apt-get update -y && apt-get upgrade -y && useradd -m docker
		# install the packages and dependencies along with jq so we can parse JSON (add additional packages as necessary)
		RUN apt-get install -y --no-install-recommends \
		    curl nodejs wget unzip vim git azure-cli jq build-essential libssl-dev libffi-dev python3 python3-venv python3-dev python3-pip
		
		The first 'RUN' instruction will update the base packages on the Ubuntu 20.04 image and add a non-sudo user called docker.
		The second 'RUN' will install packages and dependencies such as git, Azure-CLI, python along with jq so we can parse JSON for the token in our ENTRYPOINT script.
		NOTE: You can add additional packages as necessary at this stage, but try not to install too many packages at build time to keep the image as lean, compact and re-usable as possible. You can always use a GitHub Action later in a workflow when running the container and use actions to install more tooling.
		
		
		
		
		
		# cd into the user directory, download and unzip the github actions runner
		RUN cd /home/docker && mkdir actions-runner && cd actions-runner \
		    && curl -O -L https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz \
		    && tar xzf ./actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
		
		
		
		The next 'RUN' instruction will create a new folder called actions-runner and download and extract a specific version of the GitHub runner binaries based on the build argument 'ARG' value passed into the container build process that sets the environment variable: $RUNNER_VERSION as described earlier. A few more additional dependencies are also installed from the extracted GitHub runner files.
		
		
		# add over the start.sh script
		ADD scripts/start.sh start.sh
		# make the script executable
		RUN chmod +x start.sh
		# set the user to "docker" so all subsequent commands are run as the docker user
		USER docker
		# set the entrypoint to the start.sh script
		ENTRYPOINT ["./start.sh"]
		
		The last section will 'ADD' the 'ENTRYPOINT' script named start.sh to the image. The entrypoint script will run each time a new container is created. It acts as a bootstrapper that will, based on specific environment variables we pass into the Docker Run command, such as, $GH_OWNER, $GH_REPOSITORY and $GH_TOKEN to register the containers self hosted runner agent against a specific repository in the GitHub organisation we specify.
		Now that we have our scripts as well as our dockerfile ready we can build our image.
		
		
		
		
		In VSCode terminal or a PowerShell session, navigate to the root folder containing the docker file and run the following command. Remember we need to pass in a build argument to tell docker what version of the GitHub runner agent to use in the image creation.
		
		docker build --build-arg RUNNER_VERSION=2.292.0 --tag docker-github-runner-lin .
		
		Once the process is complete, you will see the new image in Docker Desktop for Windows under images:
		
		
		
		Run the Docker Image - Docker Desktop (Linux)
		To run and provision a new self hosted GitHub runner linux container from the image we just created, run the following command. We have to pass in some environment variables using the '-e' option to specify the PAT (Personal Access Token), GitHub Organisation and Repository to register the runner against.
		
		
		
		See creating a personal access token on how to create a GitHub PAT token. PAT tokens are only displayed once and are sensitive, so ensure they are kept safe.
		The minimum permission scopes required on the PAT token to register a self hosted runner are: "repo", "read:org":
		
		
		
		
		
		
		docker run -e GH_TOKEN=value' -e GH_OWNER='Dileep-github' -e GH_REPOSITORY='learning_notebooks' -d docker-github-runner-lin
		
		
		After running this command, under the GitHub repository settings, you will see a new self hosted GitHub runner. (This is our docker container):
		
		
		You will also be able to see the running container under Docker Desktop for Windows under Containers:
		
		
		
		
		
		
		Lets test our new docker container self hosted GitHub runner by creating a GitHub workflow to run a few GitHub Actions by installing Terraform on the running container.
		
		Create a configuration file under .github/workflows
		
		
		Screen clipping taken: 1/14/2023 5:28 PM
		
		
		
		
		name: CI
		on: [push]
		jobs:
		  build:
		    runs-on: [self-hosted]
		    
		    steps:
		    - uses: actions/checkout@v1
		    - name: Run a one-line script
		      run: echo Hello, world!
		    - name: Run a multi-line script
		      run: |
		        echo Add other actions to build,
		        echo test, and deploy your project.
		
![image](https://user-images.githubusercontent.com/19489833/212599530-06ef0aad-b7cc-47a5-9275-8eb722ca5965.png)


