# fhir-adapter-server-config

# Chapter 1. Introduction

***Java configuration***

Ensure java home is set up appropriately
```
sudo apt install openjdk-8-jdk
```

Check the java home path
```
sudo update-java-alternatives -l
```
Create java script

Since we want the variable to hold its value despite restarting the system and for all users, we will create he jdk_home.sh script in the /etc/profile.d/ directory:

```
sudo vi /etc/profile.d/jdk_home.sh
```

Paste the following lines to the jdk_home.sh file (use the JAVA_HOME value correct for your system):

```
#!/bin/bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
export DHIS2_HOME=/opt/dhis2
```
Next, hit the Ctrl+O shortcut followed by clicking Enter to save the changes and Ctrl+X to exit the file. The operation may fail if your user doesn’t have permission to modify the /etc content. However, running the nano editor in the sudo mode should solve this problem.

To verify that the file was modified correctly you can use the cat command to print its content:

```
cat /etc/profile.d/jdk_home.sh 
#!/bin/bash
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64
export PATH=$PATH:$JAVA_HOME/bin
export DHIS2_HOME=/opt/dhis2
```

The profile files are ignored by non-login shells but Ubuntu's default GUI login will read some of them. To solve this issue, we just use .bashrc
```
sudo vi /etc/bash.bashrc
```

And pase the following block 
```
if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
```

If you log out and log back in, the variables will be set and can be tested with the echo command.

***Database configuration***

You will need the uuid-ossp module which provides functions to generate universally unique identifiers (UUIDs).
Navigate into the postgres container, if you are in the host machine you can type
```
lxc exec postgres bash
```

Once in the container, you can navigate to your postgres database for the adapter using the following command
```
psql adapterDB
```
At this point you can install the needed module with the following command

```
CREATE EXTENSION "uuid-ossp";
```

You can exit the database by using the ```exit``` command or the ```\q``` command for quitting

***Make Logs and Artemis Directories***

Create the directories which will have the logs
```
sudo mkdir -p /opt/dhis2/services/fhir-adapter/artemis
sudo mkdir -p /opt/dhis2/services/fhir-adapter/logs
```

Create the adapter configuration file
sudo vi /opt/dhis2/services/fhir-adapter/application.yml 

***Update Adapter configuration***

Create a configuration file
```
sudo vi /opt/dhis2/services/fhir-adapter/application.yml
```

Edit to suit your environment e.g.
```
server:
  # The default port on which HTTP connections will be available when starting
  # the Adapter as a standalone application.
  port: 8081

spring:
  datasource:
    # The JDBC URL of the database in which the Adapter tables are located.
    url: jdbc:postgresql://localhost/dhis2-fhir
    # The username that is required to connect to the database.
    username: dhis-fhir
    # The password that is required to connect to the database.
    password: dhis-fhir

dhis2.fhir-adapter:
  # Configuration of DHIS2 endpoint that is accessed by the adapter.
  endpoint:
    # The base URL of the DHIS2 installation.
    url: http://localhost:8080
    # The API version that should be accessed on the DHIS2 installation.
    api-version: 30
    # Authentication data to access metadata on DHIS2 installation.
    # The complete metadata (organization units, tracked entity types, 
    # tracked entity attributes, tracker programs, tracker program stages) 
    # must be accessible.
    system-authentication:
      # The username that is used to connect to DHIS2 to access the metadata.
      username: admin
      # The password that is used to connect to DHIS2 to access the metadata.
      password: district
```

Update permissions on directories

```
sudo chown -R tomcat:tomcat /opt/dhis2/services
sudo chmod -R 600 $DHIS2_HOME/services/fhir-adapter/application.yml
```

***Install the adapter***

Make a directory to install the adapter, and navigate into it
```
mkdir adapter
cd adapter/
```
Clone the repository from github, if you do not have the adapters' WAR file.

```
sudo git clone https://github.com/ITINordic/dhis2-fhir-adapter.git
```

Run the following commands from the directory cloned from source control.
```
apt install maven
```

Navigate into the cloned repository and start to generate the package

```
cd dhis2-fhir-adapter/
```
Run the following commands
```
mvn install
```
If all tests are successful, run the below command to create the package
```
mvn package
```
***Deploy to Tomcat***
Create a web directory in Tomcat
```
mkdir /var/lib/tomcat9/webapps/adapter
```

Unzip and copy the war file
```
unzip -q -d /var/lib/tomcat9/webapps/adapter/ app/target/dhis2-fhir-adapter.war
```

Edit the tomcat settings. In this instance, you can use the below command to open the requisite file:
```
vi /etc/systemd/system/tomcat9.service.d/override.conf
```

Paste the configuration below or similar
```
[Unit]
Description=Apache Tomcat Web Application Container
After=network.target

[Service]
ReadWritePaths=/opt/dhis2
ReadWritePaths=/opt/glowroot
ReadWritePaths=/var/log/tomcat9
ReadWritePaths=/etc/hapi-fhir
Environment=DHIS2_HOME=/opt/dhis2
Environment=CATALINA_PID=$CATALINA_BASE/tomcat.pid
Environment=JRE_HOME=/usr/lib/jvm/java-1.8.0-openjdk-amd64

[Install]
WantedBy=multi-user.target
```
before you start the adapter, make sure DHIS2 allows connections from the container: Go to the DHIS2 containet and allow connections from the adapter, eg from the relevent IP as below:
```
sudo ufw allow from 192.168.0.53 to any port 8080
```

***Voila! you can restart the adapter conainter and follow the logs***
