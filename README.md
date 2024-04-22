# Deploy Netflix Clone on Kubernetes
<h4>  ------------------- Launch EC2 instance of t3.large for jenkins, sonarqube anaylysis --------------------------- </h4>

login via ssh on cmd, clone the repository <br>

  Step 1 : Install Git <br>
       <pre> sudo yum install git </pre>

  Step 2 : Clone Repository <br>
       <pre> git clone https://github.com/AnukratiRawal14/Netflix-Clone-on-Kubernetes.git </pre>
       
  Step 3 : Install Docker <br>
      <pre>
         sudo yum install docker -y
         sudo systemctl start docker
         sudo usermod -aG docker $USER
         newgrp docker
         sudo chmod 777 /var/run/docker.sock
      </pre>

  Step 4 : Setup Jenkins pipeline for CI integration (to install jenkins, java is required) <br>
  
  Install java <br>
     <pre> sudo dnf install java-17-amazon-corretto -y </pre>
  
  Verify java verion
     <pre> java -version </pre>

Step 5 : Add the Jenkins Repository <br> 
<pre> sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo </pre>

Step 6 : Import a key file from Jenkins-CI to enable installation from the package <br>
<pre> sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key </pre>

Step 7 : Install jenkins, enable, and start jenkins
<pre>
    sudo dnf install jenkins -y
    sudo systemctl enable jenkins
    sudo systemctl start jenkins
</pre>

Step 8 : Access jenkins
<pre> http://your_amazon_linux_instance_ip:8080 </pre>

Step 9 : Retrieve the password
<pre> sudo cat /var/lib/jenkins/secrets/initialAdminPassword </pre>

Step 10 : SonarQube installation <br>
<pre>docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
(to access: publicIP:9000 (by default username & password is admin))
</pre>
Now, Integrate SonarQube with your CI(jenkins) pipeline.
Configure SonarQube to analyze code for quality and security issues.

Step 10 : Go to Manage Jenkins → Plugins → Available Plugins →

Install below plugins
<pre>
1. Eclipse Temurin Installer (Install without restart)<br>
2. SonarQube Scanner (Install without restart)<br>
3. NodeJs Plugin (Install Without restart)<br>
4. Docker
5. Docker Commons
6. Docker Pipeline
7. Docker API
8. docker-build-step
9. Prometheus Plugin Integration
</pre>

<h5> Configure Java and Nodejs in Global Tool Configuration </h5>
<pre>
  Goto Manage Jenkins → Tools → Install JDK(17) and NodeJs(16)→ Click on Apply and Save
   SonarQube
     Create the token
     Goto Jenkins Dashboard → Manage Jenkins → Credentials → Add Secret Text. It should look like this
     After adding sonar token - Click on Apply and Save
     The Configure System option is used in Jenkins to configure different server
     Global Tool Configuration is used to configure different tools that we install using Plugins
     We will install a sonar scanner in the tools.
  </pre>
 Docker
  Add DockerHub Credentials:
  <pre>
    To securely handle DockerHub credentials in your Jenkins pipeline, follow these steps:
      Go to "Dashboard" → "Manage Jenkins" → "Manage Credentials."
      Click on "System" and then "Global credentials (unrestricted)."
      Click on "Add Credentials" on the left side.
      Choose "Secret text" as the kind of credentials.
      Enter your DockerHub credentials (Username and Password) and give the credentials an ID (e.g., "docker").
      Click "OK" to save your DockerHub credentials.
  </pre>
 
<h5> Create a Jenkins Pipeline </h5>
Configure CI/CD Pipeline in Jenkins:
  Create a CI/CD pipeline in Jenkins to automate your application deployment.
  <pre>
    pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
    }
    stages {
        stage('clean workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main', url: 'https://github.com/AnukratiRawal14/Netflix-Clone-on-Kubernetes.git'
            }
        }
        stage("Sonarqube Analysis") {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner -Dsonar.projectName=Netflix \
                    -Dsonar.projectKey=Netflix'''
                }
            }
        }
        // stage("quality gate") {
        //     steps {
        //         script {
        //             waitForQualityGate abortPipeline: false, credentialsId: 'Sonar-token'
        //         }
        //     }
        // }
        stage('Install Dependencies') {
            steps {
                sh "npm install"
            }
        }
        stage("Docker Build & Push"){
            steps{
                script{
                   withDockerRegistry(credentialsId: 'docker', toolName: 'docker'){   
                       sh "docker build --build-arg TMDB_V3_API_KEY=5b3fde1c84ecd659a492b5565303a13b -t netflix ."
                       sh "docker tag netflix akrawal/netflix:latest "
                       sh "docker push akrawal/netflix:latest"
                    }
                }
            }
        }
        // commented, created only to test on linux server 
        // stage('Deploy to container'){
        //     steps{
        //         sh 'docker run -d --name netflix -p 8081:80 akrawal/netflix:latest'
        //     }
        // }
    }
}
</pre>

<h4>  ------------------- Launch EC2 instance of t2.micro for monitoring - Prometheus and Grafana --------------------------- </h4>
 
Step 1 : Install Prometheus
<pre>
  sudo useradd --system --no-create-home --shell /bin/false prometheus
  wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
  tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
  cd prometheus-2.47.1.linux-amd64/
  sudo mkdir -p /data /etc/prometheus
  sudo mv prometheus promtool /usr/local/bin/
  sudo mv consoles/ console_libraries/ /etc/prometheus/
  sudo mv prometheus.yml /etc/prometheus/prometheus.yml
  sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
  sudo nano /etc/systemd/system/prometheus.service
</pre>
Add the following content to the prometheus.service file:
<pre>
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
</pre>

Here's a brief explanation of the key parts in this prometheus.service file:
User and Group specify the Linux user and group under which Prometheus will run.
ExecStart is where you specify the Prometheus binary path, the location of the configuration file (prometheus.yml), the storage directory, and other settings.
web.listen-address configures Prometheus to listen on all network interfaces on port 9090.
web.enable-lifecycle allows for management of Prometheus through API calls.

Enable and start prometheus
<pre>sudo systemctl enable prometheus
sudo systemctl start prometheus</pre>
You can access Prometheus in a web browser using your server's IP and port 9090:
http://<your-server-ip>:9090

Step 2 : Installing Node Exporter
<pre>
  sudo useradd --system --no-create-home --shell /bin/false node_exporter
  wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
  tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
  sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
  rm -rf node_exporter*
  sudo nano /etc/systemd/system/node_exporter.service
</pre>

Add the following content to the node_exporter.service file:
<pre>
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
</pre>
  
Step 3 : Integrate Jenkins with Prometheus to monitor the CI/CD pipeline. <br>
Prometheus Configuration:  <br>
To configure Prometheus to scrape metrics from Node Exporter and Jenkins, you need to modify the prometheus.yml file.
Here is an example prometheus.yml configuration for your setup:
<pre>
 global:
  scrape_interval: 15s
  
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']

  - job_name: 'jenkins'
    metrics_path: '/prometheus'
    static_configs:
      - targets: ['<your-jenkins-ip>:<your-jenkins-port>']
</pre>

Verify prometheus.yml file
<pre>
  promtool check config /etc/prometheus/prometheus.yml
  curl -X POST http://localhost:9090/-/reload
</pre>

Step 4 : Install Grafana
<pre> 
sudo nano /etc/yum.repos.d/grafana.repo
--------------------------
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
--------------------------
</pre>

<pre>
  sudo yum install grafana -y
  sudo systemctl daemon-reload
  sudo systemctl start grafana-server
  sudo systemctl status grafana-server
  sudo systemctl enable grafana-server.service
</pre>

Step 5 : Grab our public IPv4 and add :3000 at the end, ie. 10.90.80.10:3000. This will bring you to a login screen and our Username and Password will be admin.
Import a Dashboard:
<pre>
To make it easier to view metrics, you can import a pre-configured dashboard. Follow these steps:
Click on the "+" (plus) icon in the left sidebar to open the "Create" menu.
Select "Dashboard."
Click on the "Import" dashboard option.
Enter the dashboard code you want to import (e.g., code 1860).
Click the "Load" button.
Select the data source you added (Prometheus) from the dropdown.
Click on the "Import" button.
You should now have a Grafana dashboard set up to visualize metrics from Prometheus.
</pre>

Step 6 :
Create cluster and node groups
<pre>
  aws eks update-kubeconfig --region ap-south-1 --name Cluster_name --profile profile_name
</pre>

<pre>
  Create namespace - kubectl create ns ak-1-ec2
</pre>

Step 7 : Deploy Application with ArgoCD
Install ArgoCD:
<pre>
 follow doc :  https://archive.eksworkshop.com/intermediate/290_argocd/configure/
</pre>

Set Your GitHub Repository as a Source: After installing ArgoCD, create an ArgoCD Application:
<pre>
  Name: Set the name for your application.
  Destination: Define the destination where your application should be deployed.
  Project: Specify the project the application belongs to.
  Source: Set the source of your application, including the GitHub repository URL, revision, and the path to the application within the repository.
  SyncPolicy: Configure the sync policy, including automatic syncing, pruning, and self-healing.
 </pre>
 
Access your Application:
  Grab the dns name of loadbalancer and run 
  kubectl get svc -n ak-1-ec2
  
Security group images:
