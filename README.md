# Netflix Clone Cloud Pipeline

## Project Introduction
- This project involves setting up a comprehensive **CI/CD** pipeline starting from a single server where the application code is cloned and deployed locally in a container. We enhanced security with **SonarQube and Trivy**, integrated **Jenkins** for continuous integration and deployment, and utilized **Prometheus and Grafana** for monitoring. Ultimately, the application was deployed on **EKS** using **ArgoCD**, with **Helm** managing the monitoring setup, exemplifying a robust, scalable, and secure software deployment lifecycle.

### Technologies Used
- EC2, Docker, SonarQube, Trivy, OWasp, Jenkins, Prometheus, Grafana, ArgoCD, Helm, AWS EKS

### Project Overview
![Diagram](Replace Link )

- Service port(you need to set up in your security group)

| Service            | PORT    |
| ------------------ | ------- |
| `SSH`              | `:22` |
| `application`      | `:8081` |
| `Jenkins`          | `:8080` |
| `SonarQube`        | `:9000` |
| `Prometheus`       | `:9090` |


## Project Instruction 
- ‼ Note: This project involves the use of **paid** AWS services. Please be aware that charges will apply.
- Before starting, you need an AWS account. (Search: AWS Management console and register one with free)

### Step 1: Setting up EC2 for local running app with cloud service

#### Launch T2 large EC2 instance(Ubuntu)
1. EC2 instance Setting:
    1. Choose T2 large**(which is not in free tier). Because we are going to do a lot of stuff like different plugins so we need a big server
    2. Operating system: Ubuntu
    3. Create key pair(for mac using .ppk and for Linux using .pem)
    4. Network: using default VPC and subnet
    5. Firewall: allows SSH, HTTP and HTTPs traffic
    6. Storage: change to **25GB**
    7. Create instance
    - Note: Before running the instance, we have to create the **ElasticIP** for later break or etc, which make sure that our IP will not be changed
    8. Network & Security: allocate ElasticIP and make it *associate* with current instance and named it
    9. Then connect the EC2 instance (make sure that the port 22 in security group is open)
#### Update and Clone the github repo: 
   ```
   sudo apt update -y
   git clone https://github.com/Reneechang17/Netflix-CloudPipeline
   ```
#### Install the Docker and running the app using a container
1. Set up Docker on the EC2 instance
    ```
    sudo apt-get install docker.io -y
    sudo usermod -aG docker [your system user name]
    newgrp docker
    sudo chmod 777 /var/run/docker.sock
    ```
2. Build and run application using Docker containers
    ```
    docker build -t netflix .
    docker run -d --name netflix -p 8081:80 netflix:latest
    ```
3. You will get error because we need API Key
#### Get API Key from TMBD
1. search TMDB(The movie Database), then Login or create an account
2. Once you Login → Profile → Settings → API
3. Create a new Key and accept the terms, fill out the required info and get your TMDB API Key

#### Recreate your Docker image with your TMDB API key
- `docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .`
 - You might need delete previous one:
    ```
    docker stop <containerid>
    docker rmi -f netflix
    ```
#### Therefore, you can use ***PublicIP:8081*** port and access the Application on browser

### Step 2: Integrated SonarQube & Trivy for Security
#### Install SonarQube and Trivy on server
-  For SonarQube: `docker run -d --name sonar -p 9000:9000 sonarqube:lts-community` (using port ***PublicIP:9000*** to access it, default username and pwd is admin)
-  For Trivy: 
   ```
   sudo apt-get install wget apt-transport-https gnupg lsb-release
   wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
   echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
   sudo apt-get update
   sudo apt-get install trivy
   ```
-  You can type `trivy` to see all the options we can use
-  To scan correct one: `trivy fs .`
-  To scan image using Trivy: `trivy image <IMAGE ID>`
    - Once scanning by Trivy, we will get the report of what is the problem


### Step 3: Using Jenkins for CI/CD build and deploy


### Step 4: Adding Prometheus & Grafana for monitoring(EC2 and Jenkins and K8s)


### Step 5: Email Notification

### Step 6: AWS EKS → final deployment 