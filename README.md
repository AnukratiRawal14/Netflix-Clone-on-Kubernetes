# Deploy netflix clone on kubernetes
<h4> <b>Step 1: </b> Launch EC2 instance of t2.large for jenkins, sonarqube </h4>
login via ssh on cmd, clone this repo.
install git ![image](https://github.com/AnukratiRawal14/Netflix-Clone-on-Kubernetes/assets/69693530/33306b4c-1fe5-415c-a5f7-dfdc1d037122)
clone the repo ![image](https://github.com/AnukratiRawal14/Netflix-Clone-on-Kubernetes/assets/69693530/61397490-fc81-461f-9930-efd129bb3c23)
install docker ![image](https://github.com/AnukratiRawal14/Netflix-Clone-on-Kubernetes/assets/69693530/3f9b41d6-0e83-4cfc-ad22-75600b17e971)
then run 'sudo yum update -y',  'sudo yum install docker -y', 'sudo systemctl start docker', 'sudo usermod -aG docker $USER', # Replace with your system's username, e.g., 'ubuntu'
'newgrp docker',
'sudo chmod 777 /var/run/docker.sock'


<h4> <b>Step 2: </b> Setup Jenkins pipeline for CI integration </h4><br>

