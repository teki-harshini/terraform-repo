1. Install Required Tools on Your Jenkins EC2 Server

(assuming Ubuntu 20.04/22.04 machine)

# Update system
sudo apt update -y && sudo apt upgrade -y

# Install Java (required for Jenkins)
sudo apt install openjdk-17-jre -y
java -version

# Install Git
sudo apt install git -y
git --version

# Install Docker
sudo apt install docker.io -y
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER
docker --version

# Install Kubernetes CLI (kubectl)
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

# Install eksctl (if you want AWS EKS)
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Install Ansible
sudo apt install ansible -y
ansible --version

# Install Jenkins
curl -fsSL https://pkg.jenkins.io/debian/jenkins.io-2023.key | sudo tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/ | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
systemctl status jenkins


Now Jenkins is up at http://<EC2-PUBLIC-IP>:8080

2. Set Up GitHub Repo in Jenkins
# Switch to Jenkins user
sudo su - jenkins

# Generate SSH Key
ssh-keygen -t rsa -b 4096 -C "jenkins@apache"
cat ~/.ssh/id_rsa.pub


👉 Add this public key into GitHub → Repo → Settings → Deploy Keys → Add Key (with write access).

3. Write Ansible Playbook for Apache Installation

Inside Jenkins workspace or infra repo, create apache-setup.yml:

- hosts: all
  become: yes
  tasks:
    - name: Install Apache
      apt:
        name: apache2
        state: present
        update_cache: yes

    - name: Start and Enable Apache
      service:
        name: apache2
        state: started
        enabled: yes


Inventory file (hosts.ini):

[web]
localhost ansible_connection=local


Test playbook:

ansible-playbook -i hosts.ini apache-setup.yml

4. Build & Run Docker Image

Clone repo and build:

git clone https://github.com/akshu20791/apachewebsite.git
cd apachewebsite

# Build image from Dockerfile
docker build -t apache-website:v1 .

# Run container locally
docker run -d -p 8081:80 apache-website:v1

# Test
curl http://localhost:8081

5. Push Docker Image to DockerHub
docker login
docker tag apache-website:v1 sampath231/apache-website:v1
docker push sampath231/apache-website:v1

6. Deploy to Kubernetes

Create deployment.yaml:

apiVersion: apps/v1
kind: Deployment
metadata:
  name: apache-website
spec:
  replicas: 2
  selector:
    matchLabels:
      app: apache-website
  template:
    metadata:
      labels:
        app: apache-website
    spec:
      containers:
      - name: apache-website
        image: <your-dockerhub-username>/apache-website:v1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: apache-service
spec:
  selector:
    app: apache-website
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer

# Install eksctl
curl -sL "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" -o eksctl.tar.gz
tar -xzf eksctl.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version

# Create a cluster (takes ~15–20 mins)
eksctl create cluster --name Kumar-eks --region us-east-1 --nodes 2 --node-type t3.medium

Apply:

kubectl apply -f deployment.yaml
kubectl get pods
kubectl get svc


Copy the LoadBalancer EXTERNAL-IP and open in browser → you should see your Apache website.

7. Automate via Jenkins Pipeline

In Jenkins, create a pipeline job with Jenkinsfile:

pipeline {
    agent any
    stages {
        stage('Checkout') {
            steps {
                git branch: 'master', url: 'https://github.com/sampathkumarkanumolu/apachewebsite.git'
            }
        }
        stage('Ansible Install Apache') {
            steps {
                sh 'ansible-playbook -i hosts.ini apache-setup.yml'
            }
        }
        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh """
                      echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                      docker build -t apache-website:v1 .
                      docker tag apache-website:v1 $DOCKER_USER/apache-website:v1
                      docker push $DOCKER_USER/apache-website:v1
                    """
                }
            }
        }
        stage('Deploy to K8s') {
            steps {
                sh 'kubectl --kubeconfig=/var/lib/jenkins/.kube/config apply -f deployment.yml'
            }
        }
    }
}


Run Jenkins job → it will:

Pull code from GitHub

Install Apache via Ansible

Build & Push Docker image

Deploy to Kubernetes
