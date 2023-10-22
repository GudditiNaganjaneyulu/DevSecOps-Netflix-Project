# Netflix Clone

<br />

<div align="center">
  <img src="./public/assets/home-page.png" alt="Logo" width="100%" height="100%">
  <p align="center">Home Page</p>
</div>


# Deploy Netflix Clone on Cloud using Jenkins - DevSecOps Project!

### **Phase 1: Initial Setup and Deployment**

**Step 1: Launch EC2 (Ubuntu 22.04):**

- Provision an EC2 instance on AWS with Ubuntu 22.04.
- Connect to the instance using SSH.

**Step 2: Clone the Code:**

- Update all the packages and then clone the code.
- Clone your application's code repository onto the EC2 instance:
    
    ```bash
    git clone https://github.com/GudditiNaganjaneyulu/DevSecOps-Netflix-Project.git
    ```
    

**Step 3: Install Docker and Run the App Using a Container:**

- Set up Docker on the EC2 instance:
    
    ```bash
    
    sudo apt-get update
    sudo apt-get install docker.io -y
    sudo usermod -aG docker $USER  # Replace with your system's username, e.g., 'ubuntu'
    newgrp docker
    sudo chmod 777 /var/run/docker.sock
    ```
    
- Build and run your application using Docker containers:
    
    ```bash
    docker build -t netflix .
    docker run  --name netflix -p 8081:80 netflix:latest
    
    #to delete
    CTRL+C
    ```

It will show an error cause you need API key

**Step 4: Get the API Key:**

- Open a web browser and navigate to TMDB (The Movie Database) website. https://developer.themoviedb.org/reference/intro/getting-started
- Click on "Login" and create an account.
- Once logged in, go to your profile and select "Settings."
- Click on "API" from the left-side panel.
- Create a new API key by clicking "Create" and accepting the terms and conditions.
- Provide the required basic details and click "Submit."
- You will receive your TMDB API key.

Now recreate the Docker image with your api key:
```
docker build --build-arg TMDB_V3_API_KEY=<your-api-key> -t netflix .
```

**Phase 2: Security**

1. **SonarQube and Trivy:**
    - I'm using git actions for this .
    - Create free sonar cloud account https://sonarcloud.io/ and generate token or you can install it in docker .

        sonarqube
        ```
        # Git Actions 
      - name: Checkout
        uses: actions/checkout@v4
      - name: SonarCloud Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:  
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

          (or) 
        docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
        ```
        
        
        To access: 
        
        publicIP:9000 (by default username & password is admin)
        
        Trivy:
        ```
        #GitActions
      - name: Build an image from Dockerfile
        run: |
          docker build --build-arg TMDB_V3_API_KEY=${{ secrets.TMDB }} -t gudditi/netflix .
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'gudditi/netflix'
          format: 'table'
          exit-code: '1'
          ignore-unfixed: true
          vuln-type: 'os,library'
          severity: 'CRITICAL,HIGH'
        
        ```
        
        to scan image using trivy
        ```
        It scans when workflow triggered
        ```
        
        


**Phase 3: CI/CD Setup**

1. **Git Actions **

    
    ```Yaml
        name: DevSecOps-Netflix-Project

        on:
        push:
            branches:
            - main
        workflow_dispatch:


        jobs:
        sonar-scan:
            runs-on: ubuntu-latest
            steps:
            - name: Checkout
                uses: actions/checkout@v4
            - name: SonarCloud Scan
                uses: SonarSource/sonarcloud-github-action@master
                env:  
                SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

        docker-build:
            name: Build
            runs-on: ubuntu-latest
            needs:  sonar-scan
            steps:
            - name: Checkout code
                uses: actions/checkout@v4
            - name: Build an image from Dockerfile
                run: |
                docker build --build-arg TMDB_V3_API_KEY=${{ secrets.TMDB }} -t gudditi/netflix .
            - name: Run Trivy vulnerability scanner
                uses: aquasecurity/trivy-action@master
                with:
                image-ref: 'gudditi/netflix'
                format: 'table'
                exit-code: '1'
                ignore-unfixed: true
                vuln-type: 'os,library'
                severity: 'CRITICAL,HIGH'
            - name: docker login and push
                run: |
                docker login -u ${{secrets.DOCKER_USERNAME}} -p ${{secrets.DOCKER_TOKEN}}
                docker push gudditi/netflix
    ```
    
    - Store all credentials in secrets 
    ```
    SONAR_TOKEN
    TMDB
    DOCKER_USERNAME
    DOCKER_TOKEN
    ```

2. **ArgoCD**
    - Create kuberenetes cluster using *minikube* or use *DockerDesktop*
    - install helm : https://helm.sh/docs/intro/install/
    - official documentation : https://argo-cd.readthedocs.io/en/stable/ 
    ### Installation:

    ```        #Add repository
    helm repo add argo https://argoproj.github.io/argo-helm
    #Install chart
    helm install my-argo-cd argo/argo-cd --version 5.46.7
    #port-forward
    kubectl port-forward service/my-argo-cd-argocd-server -n default 8080:443
    #passcode
    kubectl -n default get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
    ```
    ### Confuguration:
    - NewApp -> Edit As YAML Paste the fallowing: -> click on **CREATE**
    ```
        project: default
        source:
        repoURL: 'https://github.com/GudditiNaganjaneyulu/DevSecOps-Netflix-Project'
        path: Kubernetes
        targetRevision: HEAD
        destination:
        server: 'https://kubernetes.default.svc'
        syncPolicy:
        automated: {}
        syncOptions:
            - CreateNamespace=true
    
    ```


    ### ACCESSING :
    - Go inside application configured and findout the status .
    - Access application from `http://<IPAddress>:30007/browse`