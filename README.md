# Jenkins Setup Guide on Amazon Linux

## ✅ Launch EC2 Instance for Jenkins Setup

###  EC2 Configuration

| Setting              | Value                 |
|----------------------|-----------------------|
| **AMI**              | Amazon Linux 2 (64-bit x86) |
| **Instance Type**    | t2.medium             |
| **Storage**          | 20 GB (General Purpose SSD - gp2) |
| **Security Group**   | Allow: 22 (SSH), 80 (HTTP), 8080 (Jenkins Port) |
| **Key Pair**         | Choose an existing or create a new one |


> 🔐 Make sure your security group allows **port 22** for SSH and **2025** (or whatever port Jenkins is configured on), and **80** if using Nginx reverse proxy.
> 



## ✅ Connect to Server and Install Java Dependency for Jenkins

```bash
sudo dnf install java-17-amazon-corretto -y
```

---

## ✅ Install Jenkins and Login

```bash
sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
sudo yum install jenkins -y
sudo systemctl enable jenkins
sudo systemctl start jenkins
```

- Jenkins runs on **port 8080** by default.
- View initial admin password:

```bash
cat /var/lib/jenkins/secrets/initialAdminPassword
```

Login to Jenkins and install required plugins.

---

## ✅ Install Terraform

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://rpm.releases.hashicorp.com/AmazonLinux/hashicorp.repo
sudo yum -y install terraform
```

---

## ✅ Reverse Proxy Using Nginx

Install nginx & update your nginx config (e.g., `/etc/nginx/conf.d/jenkins.conf`):

```nginx
server {
    listen 80;
    server_name 52.23.162.11;

    location / {
        proxy_pass http://52.23.162.11:8080;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Then start nginx:

```bash
sudo systemctl start nginx
```

---

## ✅ Change Default Jenkins Port

Create or edit the override config:

```bash
sudo mkdir -p /etc/systemd/system/jenkins.service.d/
sudo vi /etc/systemd/system/jenkins.service.d/override.conf
```

Add the following content:

```ini
[Service]
ExecStart=
ExecStart=/usr/bin/java -Djava.awt.headless=true -jar /usr/share/java/jenkins.war --webroot=/var/cache/jenkins/war --httpPort=2025
```

Then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Update Nginx config to proxy to port `2025` instead of `8080` if needed.

---

## ✅ Change Jenkins Default Path

```bash
sudo systemctl stop jenkins
sudo mkdir -p /data/jenkins
sudo chown -R jenkins:jenkins /data/jenkins
sudo mv /var/lib/jenkins/* /data/jenkins/
sudo mv /var/lib/jenkins/.* /data/jenkins/
```

Create or edit override config:

```bash
sudo vi /etc/systemd/system/jenkins.service.d/override.conf
```

Add this content:

```ini
[Service]
Environment="JENKINS_HOME=/data/jenkins"
```

Then:

```bash
sudo systemctl daemon-reload
sudo systemctl start jenkins
```

---

## ✅ Revert to Default Jenkins Path

```bash
sudo systemctl stop jenkins
sudo mv /data/jenkins/* /var/lib/jenkins/
sudo mv /data/jenkins/.* /var/lib/jenkins/ 2>/dev/null
sudo chown -R jenkins:jenkins /var/lib/jenkins
sudo rm -f /etc/systemd/system/jenkins.service.d/override.conf
sudo systemctl daemon-reload
sudo systemctl restart jenkins
```

Check if Jenkins is using the default path:

```bash
sudo systemctl show jenkins | grep JENKINS_HOME
# Should show: Environment=JENKINS_HOME=/var/lib/jenkins
```
