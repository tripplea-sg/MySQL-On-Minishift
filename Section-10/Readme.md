# 	Deploy MySQL Enterprise Monitor (MEM) Container
## 1. Obtain MEM 8.0.21
Login to edelivery.oracle.com and download MySQL Enterprise Monitor.\
File name: V999555-01.zip, copy this file to Openshift server.\
Unzip the file:
```
unzip V999555-01.zip
```
## 2. Create Dockerfile on the same directory
```
FROM oraclelinux:7-slim

RUN groupadd mysql
RUN useradd -g mysql mysql
RUN mkdir /mem

COPY mysqlmonitor-8.0.21.1248-linux-x86_64-installer.bin docker-entrypoint.sh /mem/
WORKDIR /mem

RUN chmod u+x /mem/docker-entrypoint.sh
RUN yum install -y libaio
RUN yum install -y numactl-libs
RUN yum install -y hostname
RUN yum install -y procps-ng
ENTRYPOINT ["/mem/docker-entrypoint.sh"]
```
## 3. Create docker-entrypoint.sh
```
#! /bin/bash
if [ "$ADMINUSER" = "" ]; then
   ADMINUSER="admin"
fi
if [ "$ADMINPASSWORD" = "" ]; then
   ADMINPASSWORD="admin"
fi;
if [ "$SYSTEMSIZE" = "" ]; then
   SYSTEMSIZE="small"
fi;
echo "Admin user is set to $ADMINUSER"
echo "Admin password is set to $ADMINPASSWORD"
echo "System size is set to $SYSTEMSIZE"
if [ `ls -l /opt/mysql/enterprise/monitor | grep total | awk '{print $2}'` -eq 0 ]; then
       echo "Install new Service Manager"
       /mem/mysqlmonitor-8.0.21.1248-linux-x86_64-installer.bin --mode unattended --system_size $SYSTEMSIZE --adminuser $ADMINUSER --adminpassword $ADMINPASSWORD --mysql_installation_type bundled
       /opt/mysql/enterprise/monitor/mysqlmonitorctl.sh status
fi;
while true; do
       echo "Checking if MySQL is Up"
       if [ `/opt/mysql/enterprise/monitor/mysqlmonitorctl.sh status mysql | grep "not running" | wc -l` -gt 0 ]; then
             echo "MySQL is not running, starting up"
             /opt/mysql/enterprise/monitor/mysqlmonitorctl.sh start mysql
       else
             echo "MySQL is already running"
       fi       
       echo "Checking if Tomcat is Up"
       if [ `/opt/mysql/enterprise/monitor/mysqlmonitorctl.sh status tomcat | grep "not running" | wc -l` -gt 0 ]; then
             echo "Tomcat is not running, starting up"
             /opt/mysql/enterprise/monitor/mysqlmonitorctl.sh start tomcat
       else
             echo "Tomcat is already running"
       fi
       /opt/mysql/enterprise/monitor/mysqlmonitorctl.sh status
       sleep 15;
done; 
```
## 4. Build Image
```
set -x
set -e
docker build -m 2g -c 4 -t private/mem:8.0.21 .
docker images
```
## 

