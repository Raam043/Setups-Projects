# Project :- CICD for Containerized Applications on Ansible Nodes


## Project Map
![Project Map](https://user-images.githubusercontent.com/111989928/198696150-0ec910dc-096f-402e-8917-2c719133825e.png)




Requirements:
1. Master EC2 instance with Docker,Jenkins and Ansible installed (SSH key configuration)
2. Nodes with docker+pip installed (copy of master ssh id.rsa.pub on all nodes)
3. Git Repo with Dockerfile, Application file, Anisble yaml amd nodes inverntory files

`STEP - 1`

## Install Docker
```sh
yum update -y
yum install docker -y
systemctl enable docker
systemctl start docker
yum install pip -y
pip install docker-py
```
Generate key for connecting all nodes along with docker
```sh
cd
ssh-keygen
```
```sh
cat /root/.ssh/id_rsa.pub>>/root/.ssh/authorized_keys
chmod 755 /root/.ssh/authorized_keys
cat /root/.ssh/id_rsa.pub>>/home/ec2-user/.ssh/authorized_keys
chmod 755 /home/ec2-user/.ssh/authorized_keys
ssh localhost
```

`STEP - 2`

## Now create the AMI ( image for coping the nodes)
With AMI create 2 + more nodes
![image](https://user-images.githubusercontent.com/111989928/199017364-7fe7bd3d-f624-4a1f-b754-d01c52a9e797.png)


`STEP - 3`

## Install Jenkins
```sh
wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
yum upgrade
amazon-linux-extras install java-openjdk11 -y
yum install jenkins -y
systemctl enable jenkins
systemctl start jenkins
```

## Install Ansible 
```sh
amazon-linux-extras install ansible2 -y
```

## Install Git
```sh
yum install git -y
```

Only for jenkins server to grant docker access for ec2-user
```sh
sudo chmod 666 /var/run/docker.sock
usermod -aG docker ec2-user
```

`STEP - 4`

Create Git Repo with `Dockerfile`, `Nodes.inv`, `httpd_container.yml` & your own`index.html`

**[Dockerfile](https://github.com/Raam043/Jenkins-project-test/blob/main/Dockerfile)**
```sh
FROM httpd
COPY index.html /usr/local/apache2/htdocs
EXPOSE 80
```

**[nodes.inv](https://github.com/Raam043/CICD_Project-Deploy_Webapp_to_docker_containers_using_Ansible-LINUX/blob/2360cc582c9fe661424b4dde6e6ffb77d44c3547/nodes.inv)** Copy all nodes private ip address and paste on below nodes.inv file along with "ansible_user=ec2-user"
```sh
[rnb]
172.31.35.232 ansible_user=ec2-user
172.31.39.215 ansible_user=ec2-user
```

**[Container.yml](https://github.com/Raam043/Jenkins-project-test/blob/main/httpd_container.yml)** Ansible playbook file for deploy application on all nodes
```sh
---
- hosts: rnb
  become: True
  tasks:
        
    - name: Run docker container
      command: "docker stop raam043/test_project:latest"
      command: "docker rm -f raam043/test_project:latest"
      command: "docker image rm -f raam043/test_project:latest"
      
    - name: Run docker container
      docker_container:
        name: WebServer
        image: raam043/test_project:latest
        state: started
        exposed_ports:
        - "80"
        ports:
        - "80:80"
 ```
 
`STEP - 5`

Now Setup the Jenkins profile and add plugins `Ansible` set the path as "/usr/bin" at Global tools configuration setting

![image](https://user-images.githubusercontent.com/111989928/199026626-71ed12ff-3eda-4919-b1e7-e61241ff391e.png)

![image](https://user-images.githubusercontent.com/111989928/199027450-eee436ab-c328-4eaa-a4a7-60b12a258d6f.png)


`STEP - 6`

Create `Pipeline project` with Build Triggers as Poll SCM "* * * * *

For Docker password use Secret text 
Click on `Pipeline Syntax` > select `withCredentials: Bind credentials to variables` > Variable = "DP" > add Credentials = using jenkins Credentials

![image](https://user-images.githubusercontent.com/111989928/199030547-20e46ed9-edf7-4823-9942-e41c9c23f3e3.png)

Select Secret text

![image](https://user-images.githubusercontent.com/111989928/199030755-0a0b420c-e319-432b-96d7-15f24a256478.png)

Add DockerHub Password on "Secret field"
![image](https://user-images.githubusercontent.com/111989928/199030896-810003ba-e272-47ac-8fa9-c9321cab8037.png)

![image](https://user-images.githubusercontent.com/111989928/199031232-7cdf1ee9-9e38-4cfc-b8ff-38bc55dc2329.png)

Remove "// some block" and Paste srcipt on pipeline script


Add Ansible SSH key with playbook script

![image](https://user-images.githubusercontent.com/111989928/199037968-185a9220-305f-4d17-9f8e-487e3adcaba1.png)



![image](https://user-images.githubusercontent.com/111989928/199037309-f225cf92-53d2-494f-bac9-328d3f125a58.png)

![image](https://user-images.githubusercontent.com/111989928/199037518-54c42a97-1cbb-4d91-be40-675f525c65a5.png)


Use Password as "Key pair of AWS" Open with notepad you can able see the password

![image](https://user-images.githubusercontent.com/111989928/199038803-ce6dc2d7-c525-43a1-8974-81ad7d9a7636.png)


In the end please check ssh key check and generate the script
![image](https://user-images.githubusercontent.com/111989928/199038253-1fc18712-b4dc-4e03-8932-bd06bc661103.png)


`First Run Pipeline Script`

```sh
pipeline {
    agent any

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Raam043/Jenkins-project-test.git'
            }
        }
        stage('Docker image Build') {
            steps {
                sh 'docker image rm -f myapp'
                sh 'docker build -t myapp .'
            }
        }
        stage('Docker Conatiner run') {
            steps {
                sh 'docker run -d --name myapp -p 80:80 myapp'
                sh 'docker tag myapp raam043/myapp:latest'
            }
        }
        stage('Image pushing to DockerHub') {
            steps {
                withCredentials([string(credentialsId: 'DP', variable: 'DP')]) {
                    sh 'docker login -u raam043 -p ${DP}'
                    sh 'docker push raam043/myapp:latest'
            }
        }
        }
         stage('Deploying application on Ansible Nodes') {
            steps {
                ansiblePlaybook credentialsId: 'Ansible-private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'nodes.inv', playbook: 'httpd_container.yml'
            }
        }
    }
}
```






`Second Run & Continuous Run`

It will delete running container and old images and build new images with new container

```sh
pipeline {
    agent any

    stages {
        stage('Git checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Raam043/Jenkins-project-test.git'
            }
        }
        stage('Docker image Build') {
            steps {
                sh 'docker stop myapp'
                sh 'docker rm -f myapp'
                sh 'docker image rm -f myapp'
                sh 'docker build -t myapp .'
            }
        }
        stage('Docker Conatiner run') {
            steps {
                sh 'docker run -d --name myapp -p 80:80 myapp'
                sh 'docker tag myapp raam043/myapp:latest'
            }
        }
        stage('Image pushing to DockerHub') {
            steps {
                withCredentials([string(credentialsId: 'DP', variable: 'DP')]) {
                    sh 'docker login -u raam043 -p ${DP}'
                    sh 'docker push raam043/myapp:latest'
            }
        }
        }
         stage('Deploying application on Ansible Nodes') {
            steps {
                ansiblePlaybook credentialsId: 'Ansible-private-key', disableHostKeyChecking: true, installation: 'ansible', inventory: 'nodes.inv', playbook: 'httpd_container.yml'
            }
        }
    }
}

```



Save and run the job.
 
 
 `Output`
 
 ![image](https://user-images.githubusercontent.com/111989928/199040601-9e7aaea4-5a7d-4af7-a992-26e407afb54c.png)


`Jenkins Pipeline View`

![image](https://user-images.githubusercontent.com/111989928/199040378-7e827f55-a8fd-45c9-acff-e745945885d3.png)


 Perfect -- Thanks 👍😊
                  - Ramesh
