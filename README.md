# Multi-Tier Application Deployment (Local)

A hands-on DevOps project that deploys a Java-based multi-tier application locally using **VirtualBox** and **Vagrant**. The app relies on five services running on five separate VMs: **MySQL (MariaDB), Memcached, RabbitMQ, Tomcat, and Nginx**.

## Architecture

| Service   | Hostname | IP             | OS               |
|-----------|----------|----------------|------------------|
| MySQL     | db01     | 192.168.56.15  | CentOS Stream 9  |
| Memcache  | mc01     | 192.168.56.14  | CentOS Stream 9  |
| RabbitMQ  | rmq01    | 192.168.56.13  | CentOS Stream 9  |
| Tomcat    | app01    | 192.168.56.12  | CentOS Stream 9  |
| Nginx     | web01    | 192.168.56.11  | Ubuntu 22.04     |

**Flow:** User → Nginx (load balancer) → Tomcat (app server) → Memcached (cache check) → MySQL (auth) → RabbitMQ (async messaging/logging).

---

## Part 1 — Provision the VMs

### 1. Install VirtualBox & Vagrant
```bash
brew install virtualbox vagrant
```

### 2. Install the `vagrant-hostmanager` plugin
(updates `/etc/hosts` across all VMs automatically)
```bash
vagrant plugin install vagrant-hostmanager
vagrant plugin list
```
![Vagrant Plugin List](/images/plugins.png) shows the plugin install and `vagrant plugin list` output.

### 3. Verify installation
```bash
VBoxManage --version
vagrant --version
```
![Versions](/images/version.png) shows VirtualBox and Vagrant version output.

### 4. Clone the repo and switch branches
```bash
git clone https://github.com/Muncodex/digiprofile-project.git
cd digiprofile-project
git checkout local-vagrant
```
![Git clone](/images/git_clone.png) shows the clone, branch checkout, and repo contents (`README.md pom.xml src/ vagrant/`).

The `Vagrantfile` (5 VMs defined) lives at:
`digiprofile-project/vagrant/Manual_provisioning_MacOSM1/Vagrantfile`

### 5. Bring the VMs up
```bash
cd digiprofile-project/vagrant/Manual/
vagrant up
```
![Vagrant Up](/images/vagrant.png) shows `vagrant up` provisioning the 5 machines (db01, mc01, rmq01, app01, web01).

Check status:
```bash
vagrant status
```

### 6. SSH into each VM
```bash
vagrant ssh db01
vagrant ssh mc01
vagrant ssh rmq01
vagrant ssh app01
vagrant ssh web01
```

Check IP addresses:
```bash
ip a       # CentOS VMs
ip addr    # Ubuntu VM (web01)
```

Verify inter-VM connectivity (from db01, for example):
```bash
vagrant ssh db01
ping mc01
```

Check `/etc/hosts` on any VM to confirm hostmanager is working:
```bash
cat /etc/hosts
```

---

## Part 2 — Configure Each Service

### 1. MySQL (db01)
```bash
vagrant ssh db01
sudo yum update -y
sudo dnf install -y mariadb-server mariadb
sudo systemctl start mariadb
sudo systemctl enable mariadb
sudo mysql_secure_installation
```
Set root password to `admin` during the secure-installation prompts.

Log in and create the database/user:
```bash
mysql -u root -padmin
```
```sql
create database accounts;
grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin';
grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin';
FLUSH PRIVILEGES;
exit;
```
![Maria cmds](/images/maria_cmds.png) shows creating the `accounts` DB, granting privileges, and flushing privileges.

Copy `src/main/resources/db_backup.sql` from the repo onto the VM (use `nano`/`vim` to paste contents), then restore it:
```bash
mysql -u root -padmin accounts < db_backup.sql
mysql -u root -padmin accounts
show tables;
```
![DB backup](/images/db_backup.png)  shows the DB restore command and `show tables;` output (role, user, user_role).

Open the firewall for MySQL:
```bash
systemctl start firewalld
systemctl enable firewalld
firewall-cmd --get-active-zones
firewall-cmd --zone=public --add-port=3306/tcp --permanent
firewall-cmd --reload
systemctl restart mariadb
```

### 2. Memcache (mc01)
```bash
vagrant ssh mc01
sudo yum update -y
sudo dnf install epel-release -y
sudo dnf install memcached -y
sudo systemctl start memcached
sudo systemctl enable memcached
sudo systemctl status memcached
sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
sudo systemctl restart memcached
```

Open firewall ports:
```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --add-port=11211/tcp
sudo firewall-cmd --runtime-to-permanent
sudo firewall-cmd --add-port=11111/udp
sudo firewall-cmd --runtime-to-permanent
sudo memcached -p 11211 -U 11111 -u memcached -d
sudo firewall-cmd --list-ports
```

### 3. RabbitMQ (rmq01)
```bash
vagrant ssh rmq01
sudo yum update -y
sudo dnf install epel-release -y
sudo dnf -y install centos-release-rabbitmq-38
sudo dnf --enablerepo=centos-rabbitmq-38 -y install rabbitmq-server
sudo systemctl enable --now rabbitmq-server
```

Allow external access and create an admin user:
```bash
sudo echo "loopback_users = none" | sudo tee /etc/rabbitmq/rabbitmq.conf
sudo rabbitmqctl add_user test test
sudo rabbitmqctl set_user_tags test administrator
sudo rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
sudo systemctl restart rabbitmq-server
sudo ss -tulnp | grep beam
```

### 4. Tomcat (app01)
```bash
vagrant ssh app01
sudo yum update -y
sudo dnf install java-1.8.0-openjdk java-1.8.0-openjdk-devel -y
sudo dnf install git maven wget -y
java --version
mvn -version
```

Download and extract Tomcat:
```bash
wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.75/bin/apache-tomcat-9.0.75.tar.gz
tar xzvf apache-tomcat-9.0.75.tar.gz
```

Create a dedicated `tomcat` user and move files:
```bash
sudo useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
sudo cp -r /tmp/apache-tomcat-9.0.75/* /usr/local/tomcat/
sudo chown -R tomcat.tomcat /usr/local/tomcat
```

Create the systemd service:
```bash
sudo vim /etc/systemd/system/tomcat.service
```
```ini
[Unit]
Description=Tomcat
After=network.target

[Service]
User=tomcat
WorkingDirectory=/usr/local/tomcat
Environment=JRE_HOME=/usr/lib/jvm/jre
Environment=JAVA_HOME=/usr/lib/jvm/jre
Environment=CATALINA_HOME=/usr/local/tomcat
Environment=CATALINE_BASE=/usr/local/tomcat
ExecStart=/usr/local/tomcat/bin/catalina.sh run
ExecStop=/usr/local/tomcat/bin/shutdown.sh
SyslogIdentifier=tomcat-%i

[Install]
WantedBy=multi-user.target
```

Start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl start tomcat
sudo systemctl enable tomcat
sudo systemctl status tomcat
```

Open the firewall for Tomcat:
```bash
sudo systemctl start firewalld
sudo systemctl enable firewalld
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --zone=public --add-port=8080/tcp --permanent
sudo firewall-cmd --reload
sudo firewall-cmd --zone=public --query-port=8080/tcp
```

Build and deploy the application:
```bash
export MAVEN_OPTS="-Xmx512m"
git clone https://github.com/Muncodex/digiprofile-project.git
cd digiprofile-project/
git checkout local-setup-mun
```

Update `src/main/resources/application.properties` to match your VM hosts/credentials:
```properties
#JDBC Configuration for Database Connection
jdbc.driverClassName=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://db01:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
jdbc.username=admin
jdbc.password=admin

#Memcached Configuration For Active and StandBy Host
memcached.active.host=mc01
memcached.active.port=11211
memcached.standBy.host=127.0.0.2
memcached.standBy.port=11211

#RabbitMq Configuration
rabbitmq.address=rmq01
rabbitmq.port=5672
rabbitmq.username=test
rabbitmq.password=test

#Elasticsearch Configuration
elasticsearch.host=192.168.1.85
elasticsearch.port=9300
elasticsearch.cluster=digiprofile
elasticsearch.node=digiprofilenode
```

Build the WAR file:
```bash
mvn install
```
This produces `target/digiprofile-v2.war`.

Deploy it to Tomcat's webapps folder as `ROOT.war`:
```bash
sudo -i
cd /usr/local/tomcat/webapps/
rm -rf ROOT
cp /path/to/target/digiprofile-v2.war ROOT.war
chown tomcat.tomcat /usr/local/tomcat/webapps -R
```

Verify at `http://192.168.56.12:8080`

### 5. Nginx (web01)
```bash
vagrant ssh web01
sudo -i
cat /etc/hosts
apt update
apt upgrade
apt install nginx -y
```

Create the reverse-proxy config:
```bash
vim /etc/nginx/sites-available/digiprofileapp
```
```nginx
upstream digiprofileapp {
  server app01:8080;
}

server {
  listen 80;
  location / {
    proxy_pass http://digiprofileapp;
  }
}
```

Enable the config and restart Nginx:
```bash
rm -rf /etc/nginx/sites-enabled/default
ln -s /etc/nginx/sites-available/digiprofileapp /etc/nginx/sites-enabled/digiprofileapp
systemctl restart nginx
```

---

## Result

Browse to `http://192.168.56.11:8080/login` (or the mapped port, e.g. `192.168.56.12:8080/login`) to see the application login page.

![Final_Web_App](/images/final_web_app.png) shows the deployed NextGen DevOps login page in the browser, confirming a successful deployment.

