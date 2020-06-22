### Challenge 1:
[![made-with-python](https://img.shields.io/badge/Made%20with-Python-1f425f.svg)](https://www.python.org/)
[![made-with-Markdown](https://img.shields.io/badge/Made%20with-Markdown-1f425f.svg)](http://commonmark.org)


### 1- Preparing the environment:
Vagrant ubuntu bionic box was used with bash shell provisioner to install jenkins and Docker (later gitea was also installed on the same host)

<details><summary>Vagrantfile</summary>
<p>

```Ruby
Vagrant.configure("2") do |config|
  config.vm.define "challenge" do |challenge|

  challenge.vm.box = "ubuntu/bionic64"
  challenge.vm.hostname = "ITIchallenge"
  challenge.vm.network "forwarded_port", guest: 22, host: 22 # ssh
  challenge.vm.network "forwarded_port", guest: 3000, host: 3000 # Gitea
  challenge.vm.network "forwarded_port", guest: 8080, host: 8080 # Jenkins
  challenge.vm.synced_folder ".", "/vagrant", type: "rsync"
  challenge.vm.box_check_update = false


  challenge.trigger.after :up do |trigger|
      trigger.info = "rsync auto"
      trigger.run = {inline: "bash -c 'vagrant rsync-auto challenge &'"}
      # If you want it running in the background switch these
      #t.run = {inline: "bash -c 'vagrant rsync-auto bork &'"}
    end
  end

   config.vm.provider "virtualbox" do |vb|
  #   # Display the VirtualBox GUI when booting the machine
  #   vb.gui = true
  #
  #   # Customize the amount of memory on the VM:
     vb.memory = "8192"
     vb.cpus = 2
   end

   config.vm.provision "shell", inline: <<-SHELL
    # Docker
    apt-get remove -y docker docker-engine docker.io containerd runc 
    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gnupg-agent software-properties-common git
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
    apt-key fingerprint 0EBFCD88
    add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
    apt-get update
    apt-get install -y docker-ce docker-ce-cli containerd.io
    groupadd docker
    usermod -aG docker $USER
    echo "Docker was Successfully installed"

    # Docker Compose
    curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
    chmod +x /usr/local/bin/docker-compose
    echo "Docker Compose was Successfully installed"
    
    # git config
    git config --global user.name "jaxon"
    git config --global user.email jaxon@jaxon.com

    # Jenkins
    wget -q -O - https://pkg.jenkins.io/debian/jenkins-ci.org.key | sudo apt-key add -
    sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'
    apt-get update
    apt-get install -y openjdk-8-jre jenkins 
    echo "Jenkins was Successfully installed\n\n"
    cat /var/lib/jenkins/secrets/initialAdminPassword
   SHELL
end
```

</p>
</details>

---

### 2- Dockerfile for the tornado app:
A simple Dockerfile that copies the application files and folders into the container and installs the requirements.txt was written as follows

<details><summary>Dockerfile</summary>
<p>

```Dockerfile
FROM python:alpine

WORKDIR /usr/src/app

RUN mkdir //usr/src/app/templates /usr/src/app/tests /usr/src/app/static

COPY templates ./templates 
COPY tests ./tests
COPY static ./static
COPY hello.py requirements.txt ./

RUN pip install -r requirements.txt

ENTRYPOINT [ "python3" ]

```

</p>
</details>

`python3` command is used during testing and running the application so it was placed as an ENTRYPOINT and we pass to it either the testing file location or the hello.py app

--- 

### 3- Docker-compose for joining both the app and redis:
The Docker-compose file contains the 6 environment variables used by the app, the compose file is mainly used for running the application itself as testing is done in a separate container that gets built and then removed in case the tests passed.

<details><summary>docker-compose.yml</summary>
<p>

```yaml
version: "3.8"
services:
  tornado:
    build: .
    container_name: python_iti_challenge
    command: ["hello.py"]

    environment: 
      - REDIS_HOST=redis 
      - REDIS_PORT=6379
      - REDIS_DB=0
      - ENVIRONMENT=dev 
      - HOST=localhost
      - PORT=4222
    depends_on: 
      - redis
    ports: 
      - "4222:4222"

  redis:
    image: redis:alpine
    container_name: redis_iti_challenge
    expose:
      - 6379
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
        
```

</p>
</details>

---

### 4- Jenkinsfile:
Jenkins pipeline that contains 2 stages [testing and running] 
A separate container gets built at stage 1 and then gets removed after all tests pass, then at stage 2 docker-compose runs the main container with our hello.py app

<details><summary>Jenkinsfile</summary>
<p>

```Jenkinsfile
pipeline {
    agent any

    stages {
        stage('docker run tests') {
            steps {
               sh "docker build -t \"pytest\" . && docker run --rm \"pytest\" \"/usr/src/app/tests/test.py\"" 
            }
        }
        stage('docker-compose build & run') {
            steps {
                sh "docker-compose build && docker-compose up -d"
            }
        }
    }
}

```

</p>
</details>

---

### 5- Installing and configuring gitea with jenkins:
Following the [gitea guide](https://docs.gitea.io/en-us/install-from-binary/), gitea was installed as a binary, all the files were pushed to a new repo

![Gitea preview](https://github.com/theJaxon/devops-ITI-challenges/blob/master/challenge-01/etc/gitea.png)

Then it was configured as follows to work with jenkins:
From settings > applications
a new token for jenkins was created
gitea plugin for jenkins was installed, then from Manage jenkins > configure system 
gitea server url [http://localhost:3000] was added, also the token was added to the credentials

in gitea repo > settings > webhooks 
a new webhook was added with a target url [http://localhost:8080/gitea-webhook/post]

finally in the jenkins pipeline configuration Poll SCM was selected in the optional filter to allow building whenever a change is made to the gitea repo

--- 

### Approach 2 [Running tests in multi-stage builds]:
```Dockerfile
FROM python:alpine AS test_stage 
WORKDIR /usr/src/app
CMD [ "python3" ] ["test.py"]
RUN apk update && \ 
    apk --no-cache add git && \ 
    # Clone the tornado app repo into the current WORKDIR
    git clone https://github.com/MohamedMSaeed/DevOps-Challenge-Demo-Code.git . && \ 
    # Install requirements 
    cd tests && \
    python3 test.py 

FROM python:alpine 
WORKDIR /usr/src/app 
COPY --from=test_stage /usr/src/app .
RUN pip install -r requirements.txt
```

```yaml
 
version: "3.8"
services:
  tornado:
    build: .
    container_name: python_iti_challenge_master   
    environment: 
    - REDIS_HOST=redis 
    - REDIS_PORT=6379
    - REDIS_DB=0
    - ENVIRONMENT=dev 
    - HOST=localhost
    - PORT=4222 
    depends_on: 
      - redis
    ports: 
      - "4222:4222"
    command: python3 hello.py

  redis:
    image: redis:alpine
    container_name: redis_iti_challenge_master
    expose:
      - 6379
    deploy:
      replicas: 2
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
```