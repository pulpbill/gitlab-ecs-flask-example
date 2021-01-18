# Gitlab CI/CD for Docker, Python Flask, AWS ECS Dev Ops | Gitlab CI/CD Docker Deployment on AWS ECS
### Source: These notes were the steps I had to do in order to follow Ankit Malhotra's YT tutorial: https://www.youtube.com/watch?v=vb7w7jnkD2s

- **Note:** I did some minor changes: Python 3 - Added a docker login line to gitlab-ci file - Gitlab variables for the pipelines

In order to follow this tutorial, you'll need:

- An AWS account

- A Docker hub account

- A Linux distro (I use WSL2 + ubuntu 20.04)

- Docker engine (I use docker desktop for Windows 10)

#### Python app:

- Setting up a python virtual env:
```
python3 -m venv env
```

- Read and execute the environment you've just created:
```
source env/bin/activate
```

- Install Flask
```
pip install flask
```

- Writting the flask app (you must have flask install as shown on the previous step) (use the same app.py in this repo)
```
vim app.py 
```

- Testing the app (browse on your computer: http://localhost:5000)
```
python app.py
```

- Outpur your installed packages
```
pip freeze > requirements.txt 
```

#### Dockerfile app:

- Create your Dockerfile (use the same Dockerfile in this repo)
```
vim Dockerfile
```
###### Layers:
- Set base image and first commands to execute (FROM - RUN)

- Copy the requirements (COPY)

- Define the working directory (WORKDIR)

- Install requirements (RUN)

- Copy files to working directory (COPY)

- Setup the entrypoint (ENTRYPOINT)

- Setup the command (CMD)

- Build the app (image) (based on the Dockerfile and set the image name (flaskapp2020)):
```
docker build -t flaskapp2020 .
```

- Confirm your created image (look for "flaskapp2020"):
```
docker images
```

#### Dockerhub:

- Create a public repository at your account: https://hub.docker.com/repository/create

- Copy the namespace `<your-username>`/flaskapp2020

- Log into dockerhub
```
docker login --username=<your-username>
```

- Tag your docker image (I'll use "flaskapp2020")
```
docker tag flaskapp <your-username>/flaskapp2020:latest
```

- Push your tagged image
```
docker image push <your-username>/flaskapp2020:latest 
```

#### Gitlab Project repository:
- Create a public Project via GUI: https://docs.gitlab.com/ee/gitlab-basics/create-project.html

- Setup your git context and upload your files (these steps are the same as shown when you create your repo)
```
git config --global user.name <your-username>
git config --global user.email <your-email>
git init
git remote add origin git@gitlab.com:<your-username>/flaskapp.git
git add .
git commit -m "Initial commit"
git push -u origin master 
```

#### Setup a Gitlab Runner:

- Download the package (In my case (Ubuntu) I'll use a Debian package)
```
curl -LJO -i https://gitlab-runner-downloads.s3.amazonaws.com/latest/deb/gitlab-runner_amd64.deb
```

- Install it
```
sudo dpkg -i gitlab-runner_amd64.deb
```

- Start it
```
sudo gitlab-runner start
```

- Run a multi runner service
```
sudo gitlab-runner run
```

- Register the runner 
```
sudo gitlab-runner register
```

- In order to get the token and configure the Runner, go to: https://gitlab.com/`<youruser>`/flaskapp/-/settings/ci_cd and complete as follow:
    - coordinator: https://gitlab.com/
    - token: `<your-project-token>`
    - description: `<what-ever-you-want>`
    - tag: dockerflasktest
    - executor: shell

- Verify the installation and configuration
```
sudo gitlab-runner verify 
```

#### Create an AWS ECS cluster:

- Login into your AWS console, go to ECS services and setup a cluster as follow: 
    - Choose "custom: configure"
        - Container name: flaskapp
        - Image: `<your-dockerhub-username>`/flaskapp2020 # (the one that you previously pushed to dockerhub)
        - Port Mappings: 5000 # Default TCP port for flask app
        - Load Balancer Type: ALB
        - Clustername: flaskcluster
        - Create

#### Gitlab CI/CD:

- Create your gitlab CI file (use the one in this repo):
```
vim .gitlab-ci.yml
```

- This file has/will: 
    - Your build jobs and stages 
        - tags: dockerflasktest # This tag must match the one that you used for the runner
    - Your scripts (this will execute your builds, test, etc.)
    - Build the image (docker build)
    - Push the image to dockerhub (docker image push)
    - Update ECS service/cluster
        - force will update our existing ECS cluster

- **Note:** The ECS cluster must be setup beforhand, this will update it (previous step)

#### Validate:

- Go to your AWS console, at Load Balancers section, look for the DNS name (auto generated endpoint by AWS) and access it on port 5000

#### Test your CI/CD:

- For example: Change the return msg at app.py file, to something like this (it just adds "V2"):
    - return `"<h1>Demo Flask App V2</h1>"`

- Push your changes

- Validate at gitlab console: Go to: CI/CD -> Pipelines -> Jobs section, you will see the runner (that's running on your PC executing the scripts)

- Repeat the "Validate" step and refresh the DNS endpoint, you should see "V2" now

- **Note:** Remember to leave your gitlab-runner running in order to test this

### Other useful commands and personal notes:

Confirm your registry
```
docker info | grep Registry
```

- Variables that I used on gitlab:

CI_REGISTRY_USER = `<your-dockerhub-username>`

CI_REGISTRY_PASSWORD = `<your-dockerhub-password>`

CI_REGISTRY = docker.io

CI_REGISTRY_IMAGE = flaskapp2020

CI_PROJECT_PATH = `<your-dockerhub-username>`/flaskapp2020

AWS_ACCESS_KEY_ID = `<your-aws-acces-key>`

AWS_SECRET_ACCESS_KEY = `<you-aws-secret-key>`

AWS_DEFAULT_REGION = `<your-aws-prefered-region>` # I used the cheapeast> us-east-1

- **Note:** Remember to mask your credentials: https://docs.gitlab.com/ee/ci/variables/README.html#custom-environment-variables
    - Masked values are hidden at job logs