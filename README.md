README FILE FOR BLUE-GREEN DEPLOYEMENT

youtube video for this project: https://youtu.be/tstBG7RC9as?si=q4vP173QMf-Nn2H6

This is a project for the BLUE-GREEN deployment of a java based bank app to Kubernetes using 
- Terraform (to create the K8 cluster),
- Jenkins (to run the pipeline for the deployment of the application)
- SonarQube (checks for code quality, vulnerability and bugs in code)
- Maven (to compile the code - for syntax errors, build and package the application including the dependencies -to generate the executable format - jar format. also runs unit test as defined the developer)
- Uwas dependency checks (scans for vulnerability in the code dependencies and libraries used to build and package the code)
- Nexus artifactory ( a repository management platform where packaged/built application file is stored. maintains the deferent versions of the application)
- Docker (used to build and tag the application as image which is run in K8S as container)
- Aqua Trivy (needed to scan for vulnerability in the docker image of the application. scans by layer)
- Dockerhub or private registry e.g ECR (built image is pushed here, maintains the different versions as tags)
- Jenkins pipeline is used to deploy the application from the registry to K8s as deployments.
- Monitoring of the cluster and application with Prometheus, Grafana or ELK


The Steps taken are as follows

STEP 1: create a server to run the terraform (this can also be done using a separate Jenkins pipeline)
- The terraform server is an EC2 instance of T2.medium -4gb RAM, 20gb storage and security group open for ports 500-1000, 80, 1000-11000, 443 and 22.
- access the server using SSH or SSH client like Mobaxterm.
- install AWS CLI,
	curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
	sudo apt install unzip
	unzip awscliv2.zip
	sudo ./aws/install
- connect the server to your aws account using 'aws configure'
- install terraform using
	sudo snap install terraform --classic

- Clone the GitHub repository where the terraform files are using 
	git clone <repository url>
- from the folder containing the terraform files (see the folder named 'cluster'), run terraform init, terraform plan and terraform apply to create the k8s cluster as defined.
- the eks cluster is a 3 node group ( see the terraform file in the cluster folder)

STEP 2: jenkins, Sonarqube and nexus servers
- create 3 servers each for jenkins, sonarqube and nexus using ec2 to have similar configurations as the terraform server but with 25gb storage.
- use SSH client to access the servers
- for the jenkins server, java is needed. install using
	sudo apt install openjdk-17-jre-headless
- install jenkins using
	sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
 	https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
	echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \
  	https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  	/etc/apt/sources.list.d/jenkins.list > /dev/null
	sudo apt-get update
	sudo apt-get install Jenkins
- start Jenkins using
	sudo systemctl enable Jenkins
	sudo systemctl status jenkins
- jenkins listens on port 8080 by default
- Nexus is run as a docker container, so install docker using 
	sudo apt install docker.io -y
- you can give permission to users (eg ubuntu) to run docker command on the machine using
	sudo usermod -aG docker ubuntu
	newgrp docker
- run the nexus as container using
	docker run -d -p 8081:8081 sonatype/nexus3
- access the nexus server on port 8081 using username as admin and password retrieved for /nexus-data/admin.password. get the password location by going into the nexus container using
	docker exec -it <container id> /bin/bash
- install TRIVY on the Nexus server using
	sudo apt-get install wget apt-transport-https gnupg lsb-release
	wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
	echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
	sudo apt-get update
	sudo apt-get install trivy


- Sonarqube is run as a docker container, so install docker using 
	sudo apt install docker.io -y
- run SonarQube as a docker container using
	docker run -d -p 9000:9000 sonarqube:lts-community
- access the SonarQube server on port 9000 using admin for both username and password.

 
STEP 3: install Jenkins server dependencies like docker, kubectl
- install docker for ubuntu from the official page using
	# Add Docker's official GPG key:
	sudo apt-get update
	sudo apt-get install ca-certificates curl
	sudo install -m 0755 -d /etc/apt/keyrings
	sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
	sudo chmod a+r /etc/apt/keyrings/docker.asc

	# Add the repository to Apt sources:
	echo \
  	"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  	$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  	sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
	sudo apt-get update

	# install the latest version		
	sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
- give permission to Jenkins user to run docker
	sudo usermod -aG docker jenkins
- install kubectl on Jenkins server using
	sudo snap install kubectl --classic

STEP 4: connect the terraform server to the EKS cluster
- install kubectl on the server using
	sudo snap install kubectl --classic
- connect to the cluster using
	aws eks --region eu-west-2 update-kubeconfig --name <cluster name>
- create service account, role, rolebindings and secret in a namespace for the Jenkins to assume while executing commands on the cluster. the files for these k8s objects are in Cluster/eks-rbac.md
NB: these are needed for role based access control (RBAC)

STEP 5: configure the Jenkins with plugins, tools and credentials
- create a credential for the K8s secret, GitHub personal access token, dockerhub password 
- download plugins needed such as are SonarQube scanner, config file provider, maven integration, pipeline maven integration, pipeline stage view, docker pipeline, Kubernetes, Kubernetes cli, Kubernetes client API, Kubernetes credentials

STEP 6: write the pipeline script (refer to the Jenkins file in GitHub: )
- first stage is "git checkout". this will checkout the codes/files from GitHub into Jenkins pipeline.
- second stage is compile. this uses maven to compile the code and check for syntax errors.
- third stage is unit test using maven.
- fourth stage is Trivy file system scan.
- fifth stage is SonarQube analysis. this uses the SonarQube scanner plugin. this need to be defined as environment variable.
- sixth stage is build code "quality gate check" using SonarQube. at this point, define timeout of 1hr.
- seventh stage is the build stage using maven package.
- eighth stage is to publish artifact to nexus
- introduce parameter at the beginning of the job for switching between blue and green deployment. the docker image is tagged as blue or green and is used to decide the environment to switch to. hence, the image name and tag is defined as environment variables.
- ninth stage is docker build and tag image. the command is sh 'docker build -t $IMAGE_NAME:$TAG .'
- tenth stage is trivy scan of the docker image.
- eleventh stage is docker push to dockerhub.
- next is to deploy the MySQL db and its service to Kubernetes. ensure to make necessary changes in the label to reflect blue or green tags as the case may be.
- then deploy the service of the application and then the application itself.

- configure pipeline parameter to deploy alternate version (blue or green) and switch traffic as between them. 


