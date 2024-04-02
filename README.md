![action-git](https://github.com/Eric-Kay/gitHub_Action_Netflix/assets/126447235/7b23854c-5dc7-4d30-8264-1c803aca98c1)

# Ansible – GitHub Actions: Netflix Deployment Powered by DevSecOps
From Continuous Integration (CI) and Continuous Deployment (CD) to code quality assurance and security scanning, GitHub Actions brings automation to every aspect of the development process. With custom workflows, enhanced collaboration, and release management, this tool empowers developers to be more efficient, reliable, and productive. Discover how GitHub Actions is not just a concept but a transformative solution in the daily lives of developers.

## __STEP1:__ Launch an Ec2 Instance
To launch an AWS EC2 instance with Ubuntu 22.04 using the AWS Management Console, sign in to your AWS account, access the EC2 dashboard, and click “Launch Instances.

## __STEP2A:__ Install Docker and Run Sonarqube Container

Connect to your Ec2 instance using Putty, Mobaxtreme or Git bash and install docker on it.
```bash
sudo apt-get update
sudo apt install docker.io -y
sudo usermod -aG docker ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
```

Pull the SonarQube Docker image and run it.

After the docker installation, we will create a Sonarqube container (Remember to add 9000 ports in the security group).

```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

Now copy the IP address of the ec2 instance
```bash
<ec2-public-ip:9000>
```

Provide Login and password
```bash
login admin
password admin
```

## __STEP2B:__ Integrating SonarQube with GitHub Actions
+ Integrating SonarQube with GitHub Actions allows you to automatically analyze your code for quality and security as part of your continuous integration pipeline.
+ We already have Sonarqube up and running. On Sonarqube Dashboard click on Manually.
+ Next, provide a name for your project and provide a Branch name and click on setup
+ On the next page click on With GitHub actions and it will generate an overview of the Project and provide some instructions to integrate
+ Open your GitHub and select your repository, In my case it is gitHub_Action_Netflix and Click on Settings
+ Search for Secrets and variables and click on and again click on actions
+ Click on New Repository secret
+ Use the informations from sonaqube to create your gitHub secret and workflow.

Let’s add our workflow
```bash
.github/workflows/build.yml  #you can use any name iam using sonar.yml
```
Copy content and add it to the file

```bash
name: Build,Analyze,scan
on:
  push:
    branches:
      - main
jobs:
  build-analyze-scan:
    name: Build
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v2
        with:
          fetch-depth: 0  # Shallow clones should be disabled for a better relevancy of analysis
      - name: Build and analyze with SonarQube
        uses: sonarsource/sonarqube-scan-action@master
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}
```

Click on commit changes then the workflow is created. Click on Actions and it’s automatically started the workflow

## __STEP3:__ Let’s scan files using Trivy
Add this code to your build.yml (I mean workflow)

```bash
- name: install trivy
  run: |
    #install trivy
    sudo apt-get install wget apt-transport-https gnupg lsb-release -y
    wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
    echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
    sudo apt-get update
    sudo apt-get install trivy -y
    #command to scan files
    trivy fs .
```
GitHub Actions workflow step that installs Trivy, a popular open-source vulnerability scanner for containers, and then uses it to scan the files.
Click on commit changes then the workflow is created. Click on Actions and it’s automatically started the workflow. This will install Trivy and scan all files for security checks.

## __STEP4A:__ Docker build and push to Dockerhub
Create a Personal Access token for your Dockerhub account

+ Go to docker hub and click on your profile –> Account settings –> security –> New access token. generate your token.
+ Now Go to GitHub again and click on settings and search for Secrets and variables and click on and again click on actions.
+ Click on New Repository secret and add your Dockerhub username and token as:

```bash
DOCKERHUB_USERNAME   #use your dockerhub username
```
```bash
DOCKERHUB_TOKEN
```

## __STEP4B:__ Create a TMDB API Key
+ Open a new tab in the Browser and search for TMDB
+ Click on the first result, you will see this page
+ Click on the Login on the top right. Create account if you dont have.
+ Create an API key, By clicking on your profile and clicking settings then API
+ Click on Developer and accecpt terms and conditions and then click on submit and you will get your API key.

Add the below steps to your workflow build.yml
```bash
- name: Docker build and push
  run: |
    #run commands to build and push docker images
    docker build --build-arg TMDB_V3_API_KEY=ff0b559baaea5add32ada10b92749108 -t netflix .
    docker tag netflix erickay/netflix:latest
    docker login -u ${{ secrets.DOCKERHUB_USERNAME }} -p ${{ secrets.DOCKERHUB_TOKEN }}
    docker push erickay/netflix:latest
  env:
    DOCKER_CLI_ACI: 1
```
Click on commit changes then the workflow is created. Click on Actions and it’s automatically started the workflow

## __STEP5A:__ Add a self-hosted runner to Ec2

+ Go to GitHub and click on Settings –> Actions –> Runners
+ Click on New self-hosted runner and select Linux and Architecture X64

Use the below commands to add a self-hosted runner
```bash
mkdir actions-runner && cd actions-runner
```
```bash
curl -o actions-runner-linux-x64-2.310.2.tar.gz -L https://github.com/actions/runner/releases/download/v2.310.2/actions-runner-linux-x64-2.310.2.tar.gz
```
```bash
echo "fb28a1c3715e0a6c5051af0e6eeff9c255009e2eec6fb08bc2708277fbb49f93  actions-runner-linux-x64-2.310.2.tar.gz" | shasum -a 256 -c
```
```bash
tar xzf ./actions-runner-linux-x64-2.310.2.tar.gz
```
```bash
./config.sh --url https://github.com/Aj7Ay/Netflix-clone --token A2MXW4323ALGB72GGLH34NLFGI2T4
```
```bash
./run.sh
```

## __STEP5B:__ Final workflow to run the container

Let’s add a deployment workflow build.yml
```bash
deploy:
    needs: build-analyze-scan
    runs-on: [aws-netflix]
    steps:
      - name: Pull the docker image
        run: docker pull erickay/netflix:latest
      - name: Trivy image scan
        run: trivy image erickay/netflix:latest
      - name: Run the container netflix
        run: docker run -d --name netflix -p 8081:80 erickay/netflix:latest
```
Click on commit changes then the workflow is created. Click on Actions and it’s automatically started the workflow
```bash
![build-deploy](https://github.com/Eric-Kay/gitHub_Action_Netflix/assets/126447235/5ac6cfeb-bac9-4e35-a0a8-e7d6277430de)
```
On GitHub, you will see this. the build succeeded
```bash
![Screenshot 2024-04-02 140630](https://github.com/Eric-Kay/gitHub_Action_Netflix/assets/126447235/7bce0ff6-7254-4b8b-be48-dab4ce97790a)
```
Now copy your ec2 instance ip and go to the browser
```bash
<Ec2-instance-ip:8081>
```
