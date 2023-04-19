# SonarQube Provisioning Manually
Installation of SonarQube on Ubuntu 20.04 LTS with PostgreSQL database
## Prerequisite
-  Ubuntu 20.04 LTS
-  2GB of RAM minimum
-  1 vCPU cores minimum
<br>

# Procedure 

### Refresh server’s local package index
```sh
sudo apt update -y
```

### Install OpenJDK 11
```sh
sudo apt install openjdk-11-jdk -y
```

### Update the alternatives config for JRE/JDK installations
```sh
sudo update-alternatives --config java
```
```sh
java -version
```

### Add the PostgreSQL repository
```sh
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
```

### Add the PostgreSQL signing key
```sh
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
```

### Install PostgreSQL and PostgreSQL additional extensions & utilities
```sh
sudo apt install postgresql postgresql-contrib -y
```

#### <ins> *Note* </ins>  : A user 'postgres' would have been created after the installation
```sh
cat /etc/passwd
```

`postgres:x:113:121:PostgreSQL administrator,,,:/var/lib/postgresql:/bin/bash`

### Start and Enable the 'postgresql' database server
```sh
sudo systemctl start postgresql.service
```
```sh
sudo systemctl enable postgresql.service
```
```sh
sudo systemctl status postgresql.service
```

### Create a password for the user 'postgres' with passwd 'admin123'
```sh
sudo passwd postgres
```

### Switch to the 'postgres' system user
```sh
su - postgres
```

### Create a user named 'sonar' here
```sh
createuser sonar
```

### Log in to PostgreSQL database
```sh
psql
```

### Set a password for the 'sonar' user. Use a strong password in place of 'admin123'
```sh
ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';
```

### Create a database 'sonarqube' with its owner as 'sonar' user
```sh
CREATE DATABASE sonarqube OWNER sonar;
```

### Grant all the privileges on the 'sonarqube' database to the 'sonar' user.
```sh
GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;
```

### Exit from PostgreSQL database
```sh
\q
```

### Exit from the 'postgres' user account
```sh
exit
```

### Restart the 'postgresql' database server
```sh
systemctl restart postgresql
```

### Verify for Active service and ports
```sh
systemctl status -l postgresql
```
```sh
netstat -ntpluea | grep postgres
```

### Create 'sonarqube' directory and cd into it
```sh
sudo mkdir -p /sonarqube/
```
```sh
cd /sonarqube/
```

### Download SonarQube

### To download the latest version of sonarqube, Refer: https://www.sonarsource.com/products/sonarqube/downloads/   
#### Note: Only download the latest 'LTS' version 
```sh
sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
```

### Install zip & unzip
```sh
sudo apt-get install zip unzip -y
```

### Extract the downloaded zip file in '/opt' directory
```sh
sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
```

### Rename the unzipped file in '/opt' directory as 'sonarqube' 
```sh
sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
```

### Create a group 'sonar'
```sh
sudo groupadd sonar
```

### Create a user 'sonar' and set '/opt/sonarqube' as the home directory and group as 'sonar'
```sh
sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
```
> where, <br>
> -c = comment , -d = home directory , -g = group <br>
> 
### Grant the 'sonar' user access to the '/opt/sonarqube' directory
```sh
sudo chown sonar:sonar /opt/sonarqube/ -R
```

### Edit the SonarQube configuration file
```sh
sudo vi /opt/sonarqube/conf/sonar.properties
```
> Find the following lines:
> <br> `sonar.jdbc.username=`
> <br> `sonar.jdbc.password=`
> 

### Uncomment that lines, and add the database user and password
~~~
sonar.jdbc.username=sonar
sonar.jdbc.password=admin123
~~~
### Below those above two lines, add all the following configuration lines then save and quit the file

~~~
sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
sonar.web.host=0.0.0.0
sonar.web.port=9000
sonar.web.javaAdditionalOpts=-server
sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
sonar.log.level=INFO
sonar.path.logs=logs
~~~
 
### Setup Systemd service for SonarQube service
```sh
sudo vi /etc/systemd/system/sonarqube.service
```

Paste the following lines to the file then save and quite the file
~~~
[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always

LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target
~~~

### Reload and Enable the SonarQube service to run at system startup
```sh
systemctl daemon-reload
```
```sh
systemctl enable sonarqube.service
```
```sh
systemctl status -l sonarqube.service
```

### Modify Kernel System Limits
#### <ins> *Note* </ins>  :  SonarQube uses Elasticsearch to store its indices in an MMap FS directory. It requires some changes to the system defaults.

### Edit the sysctl configuration file
```sh
sudo vi /etc/sysctl.conf
```

### Add the following lines at bottom and Save and exit the file
~~~
vm.max_map_count=262144
fs.file-max=65536
ulimit -n 65536
ulimit -u 4096
~~~
>  where,
> - vm.max_map_count = The maximum number of memory map areas a process may have <br>
> - fs.file-max = The maximum number of file-handles that the Linux kernel will allocate <br>
> - ulimit = Command that allows viewing or limiting system resource amounts that individual users consume <br>
> - -n = nofile = The maximum number of file descriptors that a process can have <br>
> - -u = nproc = Specifies how many processes/threads a user can create <br>

### Edit the resource limits for the 'sonarqube' database
```sh
sudo vi /etc/security/limits.conf 
```

### Add the following lines at bottom and Save and exit the file
~~~
sonarqube   -   nofile   65536
sonarqube   -   nproc    409
~~~

### Reboot the system to apply the changes
```sh
sudo reboot
```

### Access SonarQube in a Web browser at your server's IP address on port 9000. For example,
~~~
http://192.168.29.44:9000
~~~
> <ins> *Note* </ins>
> <br>
> By default, Log in with username ***admin*** and password ***admin*** at the SonarQube homepage
> <br>
> <br>

# Install Ngnix (optional)
- ### Ngnix is installed to act as a reverse proxy and HTTPS terminator for sonarqube instance
- ### Ngnix service listen on port 80 and route the requests to sonarqube in localhost on port 9000
```sh
sudo apt install nginx -y
```

### Remove the default Nginx config files
```sh
sudo rm -rf /etc/nginx/sites-enabled/default
```
```sh
sudo rm -rf /etc/nginx/sites-available/default
```

### Configure ngnix to listen on port 80 and route the request to sonarqube on port 9000 locally by adding the following lines
```sh
sudo vi /etc/nginx/sites-available/sonarqube
```
~~~
server{
    listen      80;
    server_name sonarqube.groophy.in;
    access_log  /var/log/nginx/sonar.access.log;
    error_log   /var/log/nginx/sonar.error.log;
    proxy_buffers 16 64k;
    proxy_buffer_size 128k;
    location / {
        proxy_pass  http://127.0.0.1:9000;
        proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
        proxy_redirect off;
              
        proxy_set_header    Host            \$host;
        proxy_set_header    X-Real-IP       \$remote_addr;
        proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header    X-Forwarded-Proto http;
    }
}
~~~
### Create Symbolic link to activate the ngnix proxy server
```sh
sudo ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
```

### Enable ngnix to start at runtime
```sh
sudo systemctl enable nginx.service
```

### Allow the following ports in Ubuntu firewall and reboot system
```sh
sudo ufw allow 80,9000,9001/tcp
```
```sh
reboot
```


