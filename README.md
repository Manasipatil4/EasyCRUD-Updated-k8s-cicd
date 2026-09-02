## This project demonstrates a complete CI/CD pipeline using Jenkins, Docker, MariaDB, Kubernetes and AWS EC2 to build, push, and deploy frontend and backend applications.

## Prerequisites
•	AWS EC2 instance (example IP: 35.154.244.253)
•	Open port 3306 for MariaDB in EC2 Security Group
•	Docker Hub account
•	Jenkins installed on EC2


## Installation Steps for jenkins
1. Install Java
```sh
sudo apt update
sudo apt install openjdk-21-jdk
java -version
```
2.  Install Jenkins
```sh
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc]" \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```


## Change Jenkins Default Port
1.	Nano  /lib/systemd/system/kenkins.service
2.	Edit the jenkins.service
3.	Replace port 8080 with 8081 in Environmet variables
4.	Restart Jenkins service


## Install Docker
```sh
sudo apt install docker.io -y
sudo usermod -aG docker jenkins
```

Grant Jenkins Sudo Privileges
```sh
sudo visudo
# Add the following line at the end of the file:
jenkins ALL=(ALL) NOPASSWD: ALL
```


## Now make a EKS Cluster  with node groups.
 

# Launch a new EC2 instance named Jenkins-k8s  and do  following steps

## Installation steps for Kubernates 
```sh
sudo -i
sudo apt update -y 
```
Install kubectl**
1: Download the latest release with the command:
```sh
curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl
```

2: Validate the binary
```sh
 curl -LO https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl.sha256
```

3: Validate the kubectl binary against the checksum file:
```sh
echo "$(cat kubectl.sha256)  kubectl" | sha256sum –check
```

4: Install kubectl:
```sh
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Note: 5: If you do not have root access on the target system, you can still install kubectl to the ~/.local/bin directory:

```sh
chmod +x kubectl
mkdir -p ~/.local/bin
mv ./kubectl ~/.local/bin/kubectl
kubectl version –client
```



Install AWS CLI on Ubuntu**
Download the aws cli bundle using below command
```sh
snap install aws-cli –classic
```


4:Configure AWS CLI
To connect AWS using CLI we have configure AWS user using below command
```sh
aws configure
```

: Log In Into EKS cluster
```sh
aws eks update-kubeconfig --name <clustername>
```

## Now we will setup database
# Make database of Mariadb in  AWS RDS
 
# On our EC2 instance Jenkins-k8s   do the installation for database

MariaDB Setup and Configuration Guide for Windows
This guide explains how to set up MariaDB, create a database, and Create Database User
1. Installing MariaDB
Installing MariaDB on Ubuntu
```sh
apt update && apt install mariadb-server -y
```

2. Securing MariaDB
Open the Command Prompt as Administrator and run the following command to secure your installation:
```sh
mysql_secure_installation
```
Follow the prompts to: Set a root password. Remove insecure default users and test databases. Disable remote root login.

3. Setting Up the Database
Open terminal and login to MariaDB:
```sh

mysql -h <rds endpoint > -uadmin -p 
```
Enter the root password when prompted.


Create a new database and user:
```sh
CREATE DATABASE student_db;
GRANT ALL PRIVILEGES ON springbackend.* TO 'username'@'localhost' IDENTIFIED BY 'your_password';
```

Replace username and your_password with your desired username and password.

4. You will need Database Credentials to Connect Backend with Database
1.	DB_HOST
2.	DB_USER
3.	DB_PASS
4.	DB_PORT
5.	DB_NAME

```sh

USE student_db;
CREATE TABLE `students` (
  `id` bigint(20) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `course` varchar(255) DEFAULT NULL,
  `student_class` varchar(255) DEFAULT NULL,
  `percentage` double DEFAULT NULL,
  `branch` varchar(255) DEFAULT NULL,
  `mobile_number` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=80 DEFAULT CHARSET=latin1 COLLATE=latin1_swedish_ci;
```

EXIT FROM DATABASE
```sh
EXIT;

```

------------------------------------------------------------------------------------------------------------------------
Copy paste the rds endpoint  in backend/src/main/resources/application properties in Git Repo

```sh
server.port=8080

spring.datasource.url=jdbc:mariadb://database-1.cv8e4sqis5mc.eu-north-1.rds.amazonaws.com:3306/student_db?sslMode=trust
spring.datasource.username=admin
spring.datasource.password=redhat123

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true

```


on our Jenkins page setup the admin and password , login and  install required plugins.

We need some additional plugins:
1. AWS Credentials Plugin

Now we need to setup the credentials 
1. Do for docker hub
Click on add credentials 
Add Username with password
Enter your dockerhub id and password and give it name docker-cred
2.AWS Credentials
Click on add credentials 
Select AWS Credentials
Give it id as AWS-cred 
Enter your Access key and Secret access key


Now we will setup the pipeline for backend
New item name it:backend  select pipeline

In script paste following script 

```sh
pipeline {
    agent any

    stages {

        stage('Clone Repo') {
            steps {
                git url: "https://github.com/Manasipatil4/EasyCRUD-Updated.git", branch: "main"
            }
        }

        stage('Build Backend Image') {
            steps {
                sh "docker build --no-cache -t imanasipatilz/easycrud1-jenkins:backend ./backend"
            }
        }

        stage('Docker Hub Login & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                    sh 'docker push imanasipatilz/easycrud1-jenkins:backend'
                }
            }
        }

        stage('Run Backend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-backend || true
                    docker run -d --name easycrud1-backend -p 8080:8080 imanasipatilz/easycrud1-jenkins:backend
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-cred', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws eks update-kubeconfig --region eu-north-1 --name cluster

                        echo "=== Nodes ==="
                        kubectl get nodes

                        kubectl apply -f backend/backend-pod.yaml
                        kubectl apply -f backend/backend-svc.yaml

                        echo "=== Pods ==="
                        kubectl get pods

                        echo "=== Services ==="
                        kubectl get svc
                    '''
                }
            }
        }

    }
}
```

Check in EC2 loadbalancer if a loadbalancer is created.

Completed with Backend

Copy it dns and paste in frontend .env in Git Repo
 

 
------------------------------------------------------------------------------------------------------------------------



Now we will setup pipeline for frontend 

New item name it:  select pipeline


```sh

pipeline {
    agent any

    stages {

        stage('Clone Repo') {
            steps {
                git url: "https://github.com/Manasipatil4/EasyCRUD-Updated.git", branch: "main"
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh "docker build -t imanasipatilz/easycrud1-jenkins:frontend ./frontend"
            }
        }

        stage('Docker Hub Login & Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', passwordVariable: 'PASSWORD', usernameVariable: 'USERNAME')]) {
                    sh 'echo $PASSWORD | docker login -u $USERNAME --password-stdin'
                    sh 'docker push imanasipatilz/easycrud1-jenkins:frontend'
                }
            }
        }

        stage('Run Frontend Container') {
            steps {
                sh '''
                    docker rm -f easycrud1-frontend || true
                    docker run -d --name easycrud1-frontend -p 80:80 imanasipatilz/easycrud1-jenkins:frontend
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([aws(accessKeyVariable: 'AWS_ACCESS_KEY_ID', credentialsId: 'aws-cred', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh '''
                        aws eks update-kubeconfig --region eu-north-1 --name cluster

                        echo "=== Nodes ==="
                        kubectl get nodes

                        kubectl apply -f frontend/frontend-deploy.yml
                        kubectl apply -f frontend/frontend-svc.yml

                        echo "=== Pods ==="
                        kubectl get pods

                        echo "=== Services ==="
                        kubectl get svc
                    '''
                }
            }
        }

    }
}

```

Check in EC2 loadbalancer if a loadbalancer is created

Copy it dns and paste in browser hence we can acess the application 


  
