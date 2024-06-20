# Netflix Clone Cloud Pipeline

### Project Introduction
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

2. Inside server:
    1. Update all packages using command: `sudo apt update -y`
    2. Clone the github repo: git clone https://github.com/Reneechang17/Netflix-CloudPipeline
    3. Install the Docker and running the app using a container
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
    4. Get API Key from TMBD
        1. search TMDB(The movie Database), then Login or create an account
        2. Once you Login → Profile → Settings → API
        3. Create a new Key and accept the terms, fill out the required info and get your TMDB API Key
    5. At this time, you can recreate your Docker image with your TMDB API key using: `docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .`
        - You might need delete previous one:
        ```
        docker stop <containerid>
        docker rmi -f netflix
        ```















### Step 2: Integrated SonarQube & Trivy for Security
- Used a service registry(Eureka) which enabled services like Job and Review to dynamically discover each other for inter-service communication.
![Eureka](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress2/Service%20Registry%20Flow.png)

- Zipkin is used for distributed tracing, helping to pinpoint failures or bottlenecks across microservices by tracing requests from end to end.
![Zipkin](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress2/Distributed%20Tracing%20with%20Zipkin.png)

  - Below diagram shows how trace and span IDs are used in Zipkin to track request.
  ![Zipkin tracing progress](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress2/Tracing%20with%20Trace%20and%20Span%20IDs.png)

- Integrated Config Server allows centralized management of configurations across microservices which fetches configurations from a Git repository.
![Config Server](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress2/Spring%20Cloud%20Config%20Server.png)

- Services running at(local):

| Service            | PORT    |
| ------------------ | ------- |
| `eureka`           | `:8761` |
| `zipkin`           | `:9411` |
| `configserver`     | `:8080` |

### Step 3: Using Jenkins for CI/CD build and deploy
- Spring Cloud Gateway routes external requests to appropriate microservices, providing a single entry point for the system.
![API Gateway](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress3/API%20Gateway.jpg)

- Resilience4J uses a circuit breaker to monitor service health and manage traffic with a rate limiter, preventing overload and maintaining service availability.
![Resilience4J](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress3/%20Resilience4J%20.jpg)

- RabbitMQ use for decoupled communication between services. 
  - The Review service acts as a producer, sending rating information to the Company service.
  - And Company service acts as a consumer. 
![RabbitMQ](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress3/rabbitmq.jpg)

- Services running at(local):

| Service            | PORT    |
| ------------------ | ------- |
| `gateway`          | `:8084` |
| `rabbitmq`         | `:15672`|

### Step 4: Adding Prometheus & Grafana for monitoring(EC2 and Jenkins and K8s)
- Use Docker to containerize services for easy deployment and scalability across different systems. 
![Docker for project](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress4/Docker%20for%20Project.jpg)

- Docker images
  - company-service: renee6177/companyms
  - job-service: renee6177/jobms
  - review-service: renee6177/reviewms
  - config-service: renee6177/configserver
  - gateway: renee6177/gateway
  - server-registry: renee6177/servicereg
  ![Docker images](https://github.com/Reneechang17/Spring-MicroJobHub/blob/main/static/progress4/docker%20images.jpg)

- Kubernetes for companyms, jobms and reviewms: Orchestrates container deployment, scaling, and management for efficient and consistent service operation across a cluster, providing automatic load balancing, service discovery, and self-healing capabilities to ensure optimal performance and reliability.

- Kubernetes for Postgres, Zipkin and RabbitMQ
  - Postgres: Stores application configuration in a centralized manner, allowing services to retrieve and apply configurations dynamically without service restarts.
  - Zipkin & RabbitMQ: Deploys Zipkin for distributed tracing capabilities, RabbitMQ for messaging system to handle communication between services.

- By deploying the services to Kubernetes cluster, we can access the Job application using internal IP and Nodeport.

- If use k8s, our project do not need API Gateway and Eureka server.

### Step 5: Email Notification

### Step 6: AWS EKS → final deployment 