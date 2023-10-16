# DevSecOps : Netflix Clone CI-CD with Monitoring | Email

# Project Blog link :-
# Project Overview :-
- We will be deploying a Netflix clone. We will be using Jenkins as a CICD tool and deploying our application on a Docker container and Kubernetes Cluster and we will monitor the Jenkins and Kubernetes metrics using Grafana, Prometheus and Node exporter. I Hope this detailed blog is useful.


# Project Steps :-
- Step 1 — Launch an Ubuntu(22.04) T2 Large Instance
- Step 2 — Install Jenkins, Docker and Trivy. Create a Sonarqube Container using Docker.
- Step 3 — Create a TMDB API Key.
- Step 4 — Install Prometheus and Grafana On the new Server.
- Step 5 — Install the Prometheus Plugin and Integrate it with the Prometheus server.
- Step 6 — Email Integration With Jenkins and Plugin setup.
- Step 7 — Install Plugins like JDK, Sonarqube Scanner, Nodejs, and OWASP Dependency Check.
- Step 8 — Create a Pipeline Project in Jenkins using a Declarative Pipeline
- Step 9 — Install OWASP Dependency Check Plugins
- Step 10 — Docker Image Build and Push
- Step 11 — Deploy the image using Docker
- Step 12 — Kubernetes master and slave setup on Ubuntu (20.04)
- Step 13 — Access the Netflix app on the Browser.
- Step 14 — Terminate the AWS EC2 Instances.

# STEP1:
- Launch an Ubuntu(22.04) T2 Large Instance
- Launch an AWS T2 Large Instance. Use the image as Ubuntu. You can create a new key pair or use an existing one. Enable HTTP and HTTPS settings in the Security Group and open all ports (not best case to open all ports but just for learning purposes it's okay).
<img width="952" alt="image" src="https://github.com/rutikdevops/DevOps-Project-11/assets/109506158/898a4ca5-4126-45a5-a9d0-b8efb0a22dbe">


# Step 2 :
- Install Jenkins, Docker and Trivy
- 2A — To Install Jenkins
- Connect to your console, and enter these commands to Install Jenkins
```bash
vi jenkins.sh #make sure run in Root (or) add at userdata while ec2 launch
```

```bash
#!/bin/bash
sudo apt update -y
#sudo apt upgrade -y
wget -O - https://packages.adoptium.net/artifactory/api/gpg/key/public | tee /etc/apt/keyrings/adoptium.asc
echo "deb [signed-by=/etc/apt/keyrings/adoptium.asc] https://packages.adoptium.net/artifactory/deb $(awk -F= '/^VERSION_CODENAME/{print$2}' /etc/os-release) main" | tee /etc/apt/sources.list.d/adoptium.list
sudo apt update -y
sudo apt install temurin-17-jdk -y
/usr/bin/java --version
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
                  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
                  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
                              /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y
sudo systemctl start jenkins
sudo systemctl status jenkins
```


```bash
sudo chmod 777 jenkins.sh
./jenkins.sh    # this will installl jenkins
```

- Once Jenkins is installed, you will need to go to your AWS EC2 Security Group and open Inbound Port 8080, since Jenkins works on Port 8080.
- Now, grab your Public IP Address


```bash
<EC2 Public IP Address:8080>
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

- 2B — Install Docker
```bash
sudo apt-get update
sudo apt-get install docker.io -y
sudo usermod -aG docker $USER   #my case is ubuntu
newgrp docker
sudo chmod 777 /var/run/docker.sock
``

- After the docker installation, we create a sonarqube container (Remember to add 9000 ports in the security group).
```bash
docker run -d --name sonar -p 9000:9000 sonarqube:lts-community
```

- 2C — Install Trivy
```bash
vi trivy.sh
```
```bash
sudo apt-get install wget apt-transport-https gnupg lsb-release -y
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | gpg --dearmor | sudo tee /usr/share/keyrings/trivy.gpg > /dev/null
echo "deb [signed-by=/usr/share/keyrings/trivy.gpg] https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main" | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update
sudo apt-get install trivy -y
```

# Step 3: Create a TMDB API Key
- Next, we will create a TMDB API key
- Open a new tab in the Browser and search for TMDB
- Click on the first result, you will see this page
- Click on the Login on the top right. You will get this page.
- You need to create an account here. click on click here. I have account that's why i added my details there.
- once you create an account you will see this page.
- Let's create an API key, By clicking on your profile and clicking settings.
- Now click on API from the left side panel.
- Now click on create
- Click on Developer
- Now you have to accept the terms and conditions.
- Provide basic details
- Click on submit and you will get your API key.

# Step 4 :
- Install Prometheus and Grafana On the new Server
- First of all, let's create a dedicated Linux user sometimes called a system account for Prometheus. Having individual users for each service serves two main purposes:
- It is a security measure to reduce the impact in case of an incident with the service.
- It simplifies administration as it becomes easier to track down what resources belong to which service.
- To create a system user or system account, run the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false prometheus
```

- --system - Will create a system account.
- --no-create-home - We don't need a home directory for Prometheus or any other system accounts in our case.
- --shell /bin/false - It prevents logging in as a Prometheus user.
- Prometheus - Will create a Prometheus user and a group with the same name.
- You can use the curl or wget command to download Prometheus.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
```

- Then, we need to extract all Prometheus files from the archive.
```bash
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
```
- Usually, you would have a disk mounted to the data directory. For this tutorial, I will simply create a /data directory. Also, you need a folder for Prometheus configuration files.
```bash
sudo mkdir -p /data /etc/prometheus
```

- Now, let's change the directory to Prometheus and move some files.
```bash
cd prometheus-2.47.1.linux-amd64/
```

- First of all, let's move the Prometheus binary and a promtool to the /usr/local/bin/. promtool is used to check configuration files and Prometheus rules.
```bash
sudo mv prometheus promtool /usr/local/bin/
```

- Optionally, we can move console libraries to the Prometheus configuration directory. Console templates allow for the creation of arbitrary consoles using the Go templating language. You don't need to worry about it if you're just getting started.
```bash
sudo mv consoles/ console_libraries/ /etc/prometheus/
```

- Finally, let's move the example of the main Prometheus configuration file.
```bash
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
```


- To avoid permission issues, you need to set the correct ownership for the /etc/prometheus/ and data directory.
```bash
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/
```

- You can delete the archive and a Prometheus folder when you are done.
```bash
cd ..
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
```


- We're going to use some of these options in the service definition.
- We're going to use Systemd, which is a system and service manager for Linux operating systems. For that, we need to create a Systemd unit configuration file.
```bash
sudo vim /etc/systemd/system/prometheus.service
```

- Prometheus.service
```bash
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
```


- Let's go over a few of the most important options related to Systemd and Prometheus. Restart - Configures whether the service shall be restarted when the service process exits, is killed, or a timeout is reached.
RestartSec - Configures the time to sleep before restarting a service.
User and Group - Are Linux user and a group to start a Prometheus process.
--config.file=/etc/prometheus/prometheus.yml - Path to the main Prometheus configuration file.
--storage.tsdb.path=/data - Location to store Prometheus data.
--web.listen-address=0.0.0.0:9090 - Configure to listen on all network interfaces. In some situations, you may have a proxy such as nginx to redirect requests to Prometheus. In that case, you would configure Prometheus to listen only on localhost.
--web.enable-lifecycle -- Allows to manage Prometheus, for example, to reload configuration without restarting the service.

- To automatically start the Prometheus after reboot, run enable.
```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status Prometheus
```

- Suppose you encounter any issues with Prometheus or are unable to start it. The easiest way to find the problem is to use the journalctl command and search for errors.
```bash
journalctl -u prometheus -f --no-pager
```

- Now we can try to access it via the browser. I'm going to be using the IP address of the Ubuntu server. You need to append port 9090 to the IP.
```bash
<public-ip:9090>
```
- If you go to targets, you should see only one - Prometheus target. It scrapes itself every 15 seconds by default.


- Install Node Exporter on Ubuntu 22.04
- Next, we're going to set up and configure Node Exporter to collect Linux system metrics like CPU load and disk I/O. Node Exporter will expose these as Prometheus-style metrics. Since the installation process is very similar, I'm not going to cover as deep as Prometheus.
- First, let's create a system user for Node Exporter by running the following command:
```bash
sudo useradd \
    --system \
    --no-create-home \
    --shell /bin/false node_exporter
```

- Use the wget command to download the binary.
```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
```

- Extract the node exporter from the archive.
```bash
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
```

- Move binary to the /usr/local/bin.
```bash
sudo mv \
  node_exporter-1.6.1.linux-amd64/node_exporter \
  /usr/local/bin/
```

- Clean up, and delete node_exporter archive and a folder.
```bash
rm -rf node_exporter*
```

- Next, create a similar systemd unit file.
```bash
sudo vim /etc/systemd/system/node_exporter.service
```

- node_exporter.service
```bash
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
ExecStart=/usr/local/bin/node_exporter \
    --collector.logind

[Install]
WantedBy=multi-user.target
```


- Replace Prometheus user and group to node_exporter, and update the ExecStart command.
- To automatically start the Node Exporter after reboot, enable the service.
```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```


- If you have any issues, check logs with journalctl
```bash
journalctl -u node_exporter -f --no-pager
```

- At this point, we have only a single target in our Prometheus. There are many different service discovery mechanisms built into Prometheus. For example, Prometheus can dynamically discover targets in AWS, GCP, and other clouds based on the labels. In the following tutorials, I'll give you a few examples of deploying Prometheus in a cloud-specific environment. For this tutorial, let's keep it simple and keep adding static targets. Also, I have a lesson on how to deploy and manage Prometheus in the Kubernetes cluster.
- To create a static target, you need to add job_name with static_configs.
```bash
sudo vim /etc/prometheus/prometheus.yml
```

- prometheus.yml
```bash
  - job_name: node_export
    static_configs:
      - targets: ["localhost:9100"]
```

- By default, Node Exporter will be exposed on port 9100.
- Since we enabled lifecycle management via API calls, we can reload the Prometheus config without restarting the service and causing downtime.
- Before, restarting check if the config is valid.
```bash
promtool check config /etc/prometheus/prometheus.yml
```

- Then, you can use a POST request to reload the config.
```bash
curl -X POST http://localhost:9090/-/reload
```

- Check the targets section
```bash
http://<ip>:9090/targets
```


- Install Grafana on Ubuntu 22.04
- To visualize metrics we can use Grafana. There are many different data sources that Grafana supports, one of them is Prometheus.
- First, let's make sure that all the dependencies are installed.
```bash
sudo apt-get install -y apt-transport-https software-properties-common
```

- Next, add the GPG key.
```bash
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
```

- Add this repository for stable releases.
```bash
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
```


- After you add the repository, update and install Garafana.
```bash
sudo apt-get update
sudo apt-get -y install grafana
```

- To automatically start the Grafana after reboot, enable the service.
```bash
sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

- Go to http://<ip>:3000 and log in to the Grafana using default credentials. The username is admin, and the password is admin as well.
```bash
username admin
password admin
```

- To visualize metrics, you need to add a data source first.
- Click Add data source and select Prometheus.
- For the URL, enter localhost:9090 and click Save and test. You can see Data source is working.
- Let's add Dashboard for a better view
- Click on Import Dashboard paste this code 1860 and click on load
- Select the Datasource and click on Import
- You will see this output

































