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
| `Node Exporter`    | `:9100` |
| `Grafana`          | `:3000` |
| `K8s deploy`       | `:30007`|


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
#### Introduced Jenkins and install necessary plugins in Jenkins
- Note: Before using Jenkins, we should install **Java** first
1. Install Java (if not exist on your server) and Jenkins on EC2 instance to automate deployment

   ```
   # Install Java first
   sudo apt update
   sudo apt install fontconfig openjdk-17-jre
   java -version
   openjdk version "17.0.8" 2023-07-18
   OpenJDK Runtime Environment (build 17.0.8+7-Debian-1deb12u1)
   OpenJDK 64-Bit Server VM (build 17.0.8+7-Debian-1deb12u1, mixed mode, sharing)

   # Then install Jenkins
   sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
   https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
   echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
   https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
   /etc/apt/sources.list.d/jenkins.list > /dev/null
   sudo apt-get update
   sudo apt-get install jenkins
   sudo systemctl start jenkins
   sudo systemctl enable jenkins
   ```

2. Command: `sudo service jenkins status` and open *PublicIP:8080* to access Jenkins
    - After accessing to the signin page, use **path** to get the password. Type `sudo cat [path]` , then get the password → continue
3. Getting Started with Jenkins: choose install suggested plugins
    - Then we need to install some necessary plugins: Manage Jenkins → Plugins → Available Plugins:
        - Eclipse Temurin Installer 
        - SonarQube Scanner
        - NodeJs Plugin
        - Email Extension Plugin
4. Configure Java and NodeJs in Tool Configuration
    - Manage Jenkins → Tools → Install **JDK(17) and NodeJs(16)** → Apply and Save
#### Setting up SonarQube
1. Back to SonarQube dashboard(PublicIP:9000), then click Administration → Security → Users → Tokens for create token and copy token
2. In Jenkins → Credentials → system → global credentials → new credential
    - Kind: Secret text
    - Scope: Global
    - Secret: paste your token
    - ID/ description: sonar-token
3. Manage Jenkins → System → Setting up SonarQube servers → Apply and Save
    - Name: sonar-server
    - Server URL: PublicIP:9000
    - choose sonar-token
4. Setting up in Tools: Manage Jenkins → Tools → Set up SonarQube Scanner → Apply
    - **So our pipeline is ready to deploy an application now**
5. Click "New Item" and select Pipeline and paste below pipeline script

```groovy

pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/N4si/DevSecOps-Project.git'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token' 
                }
            } 
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix nasi101/netflix:latest "
                       sh "docker push nasi101/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image nasi101/netflix:latest > trivyimage.txt" 
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 nasi101/netflix:latest'
            }
        }
    }
}

```

6. Add Docker and Dependency-Check Plugins and tools in Jenkins:
    1. Install OWasp(dependency-check) and Docker: Jenkins dashboard → Manage Jenkins → Manage Plugins → Search and Install
        - OWASP Dependency-Check
        - Docker/ Docker Commons/ Docker Pipeline/ Docker API/ docker-build-step 
    2. Add DockerHub Credentials:  Manage Jenkins → Manage Credentials → System → Global Credentials → Add Credentials
        - Kind: Username with password
        - Scope: Global
        - Username/ Password: Your DockerHub Username and Password
        - ID: Set up your ID
    - Whenever you push an image, you should be able to log in with this credential
7. Configure Dependency-Check Tool: Manage Jenkins → Tools → Dependency-Check installations & Docker installations → Apply
    - Name: DP-Check(same as our later process)
    - Choose "Install from github.com"
    - Name: Docker
    - Choose "download from [docker.com](http://docker.com) " with latest version
8. Add a pipeline which will be going to scan images through Trivy, check dependency through OWasp and also build and push the image to our DockerHub using the commands

```groovy

pipeline{
    agent any
    tools{
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME=tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace'){
            steps{
                cleanWs()
            }
        }
        stage('Checkout from Git'){
            steps{
                git branch: 'main', url: 'https://github.com/Reneechang17/Netflix-CloudPipeline'
            }
        }
        stage("Sonarqube Analysis "){
            steps{
                withSonarQubeEnv('sonar-server') {
                    sh ''' $SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix '''
                }
            }
        }
        stage("quality gate"){
           steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage('OWASP FS SCAN') {
            steps {
                dependencyCheck additionalArguments: '--scan ./ --disableYarnAudit --disableNodeAudit', odcInstallation: 'DP-Check'
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('TRIVY FS SCAN') {
            steps {
                sh "trivy fs . > trivyfs.txt"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){
                       sh "docker build --build-arg TMDB_V3_API_KEY=<yourapikey> -t netflix ."
                       sh "docker tag netflix nasi101/netflix:latest "
                       sh "docker push nasi101/netflix:latest "
                    }
                }
            }
        }
        stage("TRIVY"){
            steps{
                sh "trivy image nasi101/netflix:latest > trivyimage.txt"
            }
        }
        stage('Deploy to container'){
            steps{
                sh 'docker run -d --name netflix -p 8081:80 nasi101/netflix:latest'
            }
        }
    }
}

```

- Note: If your pipeline failure with "Docker LogIn Failed", you can command below and try to solve(make sure your credentials and port no problems):

```
sudo su 
sudo usermod -aG docker jenkins
sudo systemctl restart jenkins 
```

### Step 4: Adding Prometheus & Grafana for monitoring(EC2 and Jenkins and K8s)
#### Launch T2 medium EC2 instance for Monitoring server
- Before: why we don’t use the same server with T2 large?? Because the server run so slowly so we separate the server.
1. Operating system: Ubuntu
    - The minimum requirements for Prometheus Server is: 2 CPU cores, 4 GB of memory and 20 GB of free disk space
2. Use the same key pair 
3. Make sure allows HTTPS traffic
4. Setting storage with 20GB
5. Create instance
6. Also need to set up ElasticIP to avoid IP change and *associate* it with monitoring server(How to set you can see Step 1)
7. Then we run the server and do the initialize update first: `sudo apt update -y`

#### Install Prometheus
1. Add a user named prometheus and install prometheus from official github repo (It will download the zip folder, we need to unzip it later)
    
    ```
    sudo useradd --system --no-create-home --shell /bin/false prometheus
    wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
    ```
    
2. Unzip the folder and go inside folder, then create a dictionary named as data and prometheus, move all necessary files inside local bin

    ```
    tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
    cd prometheus-2.47.1.linux-amd64/
    sudo mkdir -p /data /etc/prometheus
    sudo mv prometheus promtool /usr/local/bin/
    sudo mv consoles/ console_libraries/ /etc/prometheus/
    sudo mv prometheus.yml /etc/prometheus/prometheus.yml
    ```

    - After doing this, you can see under */usr/local/bin/*, there are prometheus promtool, and under */etc/prometheus/*, there are consoles/ console_libraries prometheus.yml
3. Change the permission for above files
    - The initial owner is Ubuntu, we need to change it to the Prometheus user we created
    `sudo chown -R prometheus:prometheus /etc/prometheus/ /data/` 
4. Create Prometheus systemd unit config: `sudo nano /etc/systemd/system/prometheus.service`
    - this command will open nano tab, and we need to paste below(service configuration) to the tab:

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```  

5. Then we can enable and start the Prometheus

    ```
    sudo systemctl enable prometheus
    sudo systemctl start prometheus
    ```

6. Verify Status: `sudo systemctl status prometheus`
7. Finally, you can use *PublicIP:9000* to access the web browser. Make sure setting up the port 9090 for Prometheus in Security group

#### Install Node Exporter
1. Create a user named node_exporter and install node exporter from official github repo
    
    ```
    sudo useradd --system --no-create-home --shell /bin/false node_exporter
    wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
    ```
    
2. Unzip the folder and move necessary file to particular folder and clean up
    
    ```
    tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
    sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
    rm -rf node_exporter*
    ```
    
    - After doing this, we can see under /usr/local/bin/, there are **node_exporter** prometheus promtool
3. Create Node Exporter systemd unit config: `sudo nano /etc/systemd/system/node_exporter.service`
    - this command will open nano tab, and we need to paste below service configuration to the tab: 

    ```
    [Unit]
    Description=Node Exporter
    Wants=network-online.target
    After=network-online.target
    
    StartLimitIntervalSec=500
    StartLimitBurst=5
    
    [Service]
    User=node_exporter
    Group=node_exporter
    Type=simple
    Restart=on-failure
    RestartSec=5s
    ExecStart=/usr/local/bin/node_exporter --collector.logind
    
    [Install]
    WantedBy=multi-user.target
    ```
        
4. Enable and start Node Exporter:
    
    ```
    sudo systemctl enable node_exporter
    sudo systemctl start node_exporter
    ```
    
5. Verify the status: `sudo systemctl status node_exporter`
6. Finally, you can access Node Exporter metrics in Prometheus

#### Integrate Node Exporter with Jenkins (modify the prometheus.yml file)
1. First cd to /etc/prometheus/
2. Whenever you want to monitor something, you should edit it → use `cat prometheus.yml` to see the job now exist and going to add node exporter and Jenkins
  - How to edit?? command: `sudo nano prometheus.yml`
    - and use the below template to add(Make sure your Jenkins IP and port are correct:
    
    ```
    global:
      scrape_interval: 15s
    
    scrape_configs:
      - job_name: 'node_exporter'static_configs:
          - targets: ['localhost:9100']
    ```
    
3. Check the validity: `promtool check config /etc/prometheus/prometheus.yml`
4. Reload: `curl -X POST http://localhost:9090/-/reload`
5. And you can access Prometheus targets and see node_exporter: *PublicIP:9090/targets*
    - Make sure your port 9100 is open

#### Install Grafana and set it work with Prometheus
1. Install all necessary dependencies for Grafana and set port 3000 port with Grafana in security group

    ```
    sudo apt-get update
    sudo apt-get install -y apt-transport-https software-properties-common
    ```
    
2. Add GPG key for Grafana, and it will output “OK”
    
    ```
    wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
    ```
    
3. Add Grafana repo
    
    ```
    echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
    ```
    
4. Update and install Grafana
    
    ```
    sudo apt-get update
    sudo apt-get -y install grafana
    ```
    
5. Enable and start Grafana, also you can check the status

    ```
    sudo systemctl enable grafana-server`
    sudo systemctl start grafana-server` 
    sudo systemctl status grafana-server`
    ```

6. Then you can open the *PublicIP:3000* to see Grafana with browser. Default username and password is "admin"
7. Add Prometheus data source in Grafana:
    - Add data source option → choose "Prometheus" →Put your Prometheus server URL in box(In my case is PublicIP: 9090) → Scroll down and click Save and Test
    
8. Back to Grafana home page and import a dashboard:
    - Click "+" sign → import a dashboard → Enter your template ID(search for "node exporter grafana dashboard" on website then you can get it) → Select "Prometheus" as data source → import and you will see a nice dashboard show us CPU, RAM and more info..

#### Integrate Jenkins with Prometheus and monitor Jenkins using Grafana
1. Integrate Jenkins with Prometheus
    1. Back to Jenkins dashboard → Manage Jenkins → Plugins → Available plugins → search for “Prometheus metrics” → install
        - Note: If you restart it you should make sure your password is ready since you will need to log in with your password:
          `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` 
    
    2. Sign in with Jenkins → Manage Jenkins → System(might wait some time) → check “Prometheus”(nothing to do actually) → Save 
    3. Add Jenkins into your prometheus.yml(use nano)
    
    ```
     scrape_configs:
    	 - job_name: 'jenkins'
    		 metrics_path: '/prometheus'
    		 static_configs:
    	     - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
    ```
    
    4. Reload: `curl -X POST http://localhost:9090/-/reload` 
    5. And you can access Prometheus targets and see Jenkins using *PublicIP:9090/targets*

2. Monitor Jenkins using Grafana
    - 1. Back to Grafana home page and import a dashboard:
    - 2. Click "+" sign → import a dashboard → Enter your template ID(search for "node exporter grafana dashboard" on website then you can get it) → Select "Prometheus" as data source → import and you will see a nice dashboard show data about Jenkins

### Step 5: Email Notification
1. First, you need a gmail account
2. Manage your Google Account → Security → Check you have 2fa enable → search "App Passwords" → Enter your google account password → Create an App and set name: Netflix → Create and get your password
- Using for Jenkins to send the email notifications every time your pipeline has been running and get the reports
    
3. Back to Jenkins dashboard →  Manage Jenkins → System → Scroll down to find E-mail Notification → 
    - SMTP: [smtp.gamil.com](http://smtp.gamil.com) 
    - Default user-email: put your email
    - For Advanced, select Use SMTP (fill out username and password which you got from App Passwords) and Use SSL, SMTP port is 465,
    - Select "Test Configuration" and put your email → Save → And you will receive a test mail 
4. And find "Extended E-mail Notification", you need to enter same thing → 
    - For credentials, select the one just created, and select Use SSL 
    - For Default Content Type: HTML 
    - For Trigger: select "Always" and "Failure - Any" → Apply and Save
5. Back to Jenkins → Pipeline Netflix → Configure → Add below to your script → Approve script → Apply

```
post {
 always {
    emailext attachLog: true,
        subject: "'${currentBuild.result}'",
        body: "Project: ${env.JOB_NAME}<br/>" +
            "Build Number: ${env.BUILD_NUMBER}<br/>" +
            "URL: ${env.BUILD_URL}<br/>",
        to: '[your email]',                                
        attachmentsPattern: 'trivyfs.txt,trivyimage.txt'
    }
}
```

### Step 6: AWS EKS → final deployment 
1. Setting Up EKS : default cluster service role → Remove your personal subnet → default security group → Next and Next → default add-ons → Next and review and create
2. Create your Nodes: EKS → Clusters → Netflix → Add Node group → and we set 1 instance (node) running → next and create
3. Before go ahead, you need to install **Helm** on your machine.
4. Install **ArgoCD**, you can simply follow: https://archive.eksworkshop.com/intermediate/290_argocd/install/
After doing so, you might need to wait for some time, because this service is a load balancer so it will create a load balancer in our AWS dashboard, once it created, we can get the DNS and log to ArgoCD to connect our repo and deploy it
5. Install Node Exporter using Helm:
    1. Add Prometheus community Helm repo: `helm repo add prometheus-community https://prometheus-community.github.io/helm-charts`
    2. Create a k8s namespace for the node exporter: `kubectl create namespace prometheus-node-exporter`
    3. Install node exporter using helm: `helm install prometheus-node-exporter prometheus-community/prometheus-node-exporter --namespace prometheus-node-exporter`
6. Expose ArgoCD-server: https://archive.eksworkshop.com/intermediate/290_argocd/configure/
    - the DNS will stored in ARGOCD_SERVER, and we can use: `echo $ARGOCD_SERVER` to get the endpoint and open it then follow the instruction to sign in
7. In your ArgoCD dashboard → Settings → Repositories → Connect repo using HTTPS → paste your github repo URL → Click "Connect" → Back and click "New APP" → set your application and project name → for Sync Policy, choose: Automatic, and paste your github URL on source and Destination, then the path is "Kubernetes", and Namespace is "default" → Save → Click "Sync" → Select "Force" → Click "Synchronize" and "OK" 
    - name: Set the name for your application
    - destination: Define the destination where your application should be deployed
    - project: Specify the project the application belongs to
    - source: Set the source of your application, including the GitHub repository URL, revision, and the path to the application within the repository
    - syncPolicy: Configure the sync policy, including automatic syncing, pruning, and self-healing
  - This application actually using node port service and working on port 30007
8. Access port 3007: Make sure that port 30007 is open in security group and open *PublicIP:30007* on browser, your application should be running.
9. Monitoring and add it in Prometheus
    - add following to your prometheus.yml file using nano, and make sure open port 9100 in security group, and you can using PublicIP:9100 to access Node Exporter and check the metrics for k8s cluster

    ```
      - job_name: 'Netflix'
        metrics_path: '/metrics'
        static_configs:
          - targets: ['node1Ip:9100']
    ```

    - and we can also add K8s on your targets section
    
    ```
      - job_name: 'K8s'
        metrics_path: '/metrics'
        static_configs:
          - targets: ['node1Ip:9100']

    ```
