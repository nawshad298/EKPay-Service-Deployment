EKPay Service Deployment Guide:

1. Prepare Build Server 

Java Install 
Java is a prerequisite for installing Jenkins. First, install Java on Ubuntu by following these 

steps: 
● sudo apt-get update 
● sudo apt-get install openjdk-17-jdk 
● java -version 

Jenkins Install 
Now, to install Jenkins, issue the following four commands in sequence to initiate the 
installation from the Jenkins repository: 

$ sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \ 
https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key 

$ echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc]" \ 
https://pkg.jenkins.io/debian-stable binary/ | sudo tee \ 
/etc/apt/sources.list.d/jenkins.list > /dev/null 

$ sudo apt-get update 
$ sudo apt-get install jenkins 

Once that’s done, start the Jenkins service with the following command: 

$ sudo systemctl enable jenkins 
$ sudo systemctl start jenkins 

To confirm its status, use: 
$ sudo systemctl status jenkins 

Note: By default, Jenkins runs on port 8080. To ensure that this port is accessible, 
configure the built-in Ubuntu firewall (ufw). To open port 8080 and enable the firewall. 

Use the following commands to open port 8080: 
$ sudo firewall-cmd --add-port=8080/tcp --permanent 
$ sudo firewall-cmd --reload 

After configuring the firewall, it's time to set up Jenkins. Enter the IP address of the 
machine along with the port number in your browser. This will launch the Jenkins setup 
wizard:


An administrator password will be needed to proceed with the configuration. It can be 
easily found inside the /var/lib/jenkins/secrets/initialAdminPassword file. To check the 
initial password, use the cat command as indicated below:

$ sudo cat /var/lib/jenkins/secrets/initialAdminPassword.

Copy the password, go back to the setup wizard, paste it and click Continue.

Next, the 'Customize Jenkins' window will appear. It is recommended to simply select the 
'Install suggested plugins' option for this step.


Allow a few minutes for the installation process to complete. Once it's done, specify a 
username, password, full name, and email address, then click 'Save and Continue' to create 
an admin user. 

Jenkins plugin install 
To install a plugin, go to "Manage Jenkins," then navigate to "Plugins." From the 
“available plugins”, install the following: 
1. Eclipse Temurin Installer 
2. openJDK Native Plugin 
3. NodeJS

2. Install Docker on All Servers 
Uninstall old Docker versions (if any): 
$ sudo apt remove docker docker-engine docker.io containerd runc -y

Install Docker: 
$ sudo apt update 
$ sudo apt install apt-transport-https ca-certificates curl software-properties-common -y 
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add - 
$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu 
$(lsb_release -cs) stable" 

$sudo apt update 
$ sudo apt install docker-ce -y 


Enable Docker: 
$ sudo systemctl start docker 
$ sudo systemctl enable docker 

Verify Installation : 
$ sudo docker --version 

Optional : Allow non-root user to run Docker: 
$ sudo usermod -aG docker $USER

3. Setup Docker Harbor Private Registry 
 
Pull Harbor Installer: 
$ wget https://github.com/goharbor/harbor/releases/download/v2.10.1/harbor-online-installer-v2.10.1.tgz 
$ tar xvf harbor-online-installer-v2.10.1.tgz 
$ cd harbor 

Configure Harbor:   
Edit harbor.yml: 
○ Set hostname
○ Enable HTTPS (optional) 
○ Set admin password 

Install Harbor: 
sudo ./install.sh 

Access Harbor: https://<harbor-host> and log in with admin credentials. 

4. Configure Git Credentials in Jenkins 
1. Go to Jenkins Dashboard → Manage Jenkins → Credentials → System → Global 
credentials. 

2. Add Credentials: 
○ Kind: Username with password (for GitHub/Bitbucket) 
○ Username: Git username 
○ Password/Token: Git password or personal access token 
○ ID: git-credentials (for reference in pipelines) 

 
5. Enable Passwordless SSH for Jenkins Users 

To enable your Jenkins build server to SSH into another server without needing a 
password, follow these steps: 

Switch to the Jenkins User: Run the following command to switch to the Jenkins user: 
● su - 
● su - jenkins 

Generate SSH Key Pair: Create a new SSH key pair by running the following 
command: 

● ssh-keygen -t rsa -b 4096 

Copy the SSH Public Key to the Remote Server: Use ssh-copy-id to copy the 
public key to the target server: 

● ssh-copy-id usser@10.10.10.32 

This will allow the Jenkins server to SSH into the remote server without requiring a 
password. 

6. Save Harbor Registry Credentials in Jenkins 

To securely authenticate with your private Harbor registry from Jenkins pipelines: 

Step 1: Open Jenkins Credentials 

1. Go to Jenkins Dashboard → Manage Jenkins → Credentials → System → Global 
credentials (unrestricted). 
2. Click “Add Credentials” on the left sidebar. 

Step 2: Choose Credential Type 
● Kind: Username with password 
● Username: Harbor username (e.g., admin or service user) 
● Password: Harbor password 
● ID: harbor-credentials 
(You can give any ID, but using a clear name helps reference it in pipelines.) 
● Description: Harbor Docker Registry Credentials 
Click Save. 

7. Create Jenkins Pipeline to Build, Push, and Deploy A demo 
Service 

Pipeline flow: 
1. Run using a Docker agent (clean, isolated environment). 
2. Pull source code from Git. 
3. Build the application with Maven. 
4. Build a Docker image. 
5. Push image to Harbor private registry (using credentials saved earlier). 
6. SSH into the deployment server and run the container. 

Jenkinsfile — EKPay Demo Pipeline 


        } 
 
        stage('Deploy to Server') { 
            steps { 
                sh """ 
                    ssh -o StrictHostKeyChecking=no ${DEPLOY_SERVER} ' 
                        docker pull ${HARBOR_URL}/${PROJECT}/${APP_NAME}:latest && 
                        docker stop ${APP_NAME} || true && 
                        docker rm ${APP_NAME} || true && 
                        docker run -d --name ${APP_NAME} -p 8080:8080 
${HARBOR_URL}/${PROJECT}/${APP_NAME}:latest 
                    ' 
                """ 
            } 
        } 
    } 
 
    post { 
        always { 
            echo 'Pipeline completed.' 
        } 
    } 
} 
