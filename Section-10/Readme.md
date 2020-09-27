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
Get the \<ImageID>
## 5. Register Image to Openshift Registry
```
docker login -u `oc whoami` -p `oc whoami --show-token` 172.30.1.1:5000
sudo docker tag <imageID> 172.30.1.1:5000/db-mysql-dev/mem 
sudo docker push 172.30.1.1:5000/db-mysql-dev/mem
```
## 6. Upload Template for MEM
Copy and paste the following YAML code and save as mem-template.yaml
```
apiVersion: v1
kind: Template
metadata:
  name: 'mem-template'
  labels:
    app: mem
objects:
  - kind: StatefulSet
    apiVersion: apps/v1
    metadata:
      name: '${statefulsetname}'
      labels:
        app: '${statefulsetname}'
    spec:
      strategy:
        type: Rolling
        rollingParams:
          updatePeriodSeconds: 1
          intervalSeconds: 1
          timeoutSeconds: 600
          maxUnavailable: 25%
          maxSurge: 25%
      triggers:
        - 
          type: ConfigChange
        - 
          type: ImageChange
          imageChangeParams:
            automatic: true
            containerNames:
              - '${statefulsetname}'
            from:
              kind: ImageStreamTag
              namespace: '${namespace}'
              name: '${imageName}' 
      replicas: 1
      test: false
      selector:
        matchLabels:
          app: '${statefulsetname}'
      template:
        metadata:
          labels:
            app: '${statefulsetname}'
        spec:
          containers:
            -   
              name: '${statefulsetname}'
              image: '${imageName}' 
              env:
                - name: ADMINUSER
                  value: '${memadmin}'
                - name: ADMINPASSWORD
                  value: '${mempassword}'
                - name: SYSTEMSIZE
                  value: '${memsize}' 
              ports:
                - 
                  name: dbport
                  containerPort: 3306
                  protocol: TCP
                - 
                  name: memport
                  containerPort: 18443
                  protocol: TCP
              volumeMounts:
                  - name: mem-vol
                    mountPath: /opt/mysql/enterprise/monitor
              resources:
                limits:
                  cpu: 2000m
                  memory: 4096Mi
                requests:
                  cpu: 2000m
                  memory: 2048Mi
              imagePullPolicy: Always
          restartPolicy: Always
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          securityContext:
          supplementalGroups:
              - 110
      volumeClaimTemplates:
      - metadata:
          name: mem-vol
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi        
  - kind: Service
    apiVersion: v1
    metadata:
      name: '${statefulsetname}'
      labels:
        app: '${statefulsetname}'
    spec:
      clusterIP: None
      ports:
        -
          name: 3306-tcp
          protocol: TCP
          port: 3306
          targetPort: 3306
        -
          name: 18443-tcp
          protocol: TCP
          port: 18443
          targetPort: 18443
      selector:
        statefulset.kubernetes.io/name: '${statefulsetname}'
parameters:
  - name: namespace
    displayName: OpenShift namespace
    value: ''
    required: true
  - name: imageName
    displayName: ndb cluster docker image name
    value: ''
    required: true 
  - name: statefulsetname
    displayName: Data Node statefulset 
    value: ''
    required: true
  - name: memadmin
    displayName: mysql user name
    value: ''
    required: true
  - name: mempassword
    displayName: mysql user password
    value: ''
    required: true
  - name: memsize
    displayName: big small medium
    value: ''
    required: true
```
Apply mem-template.yaml
```
oc apply -f mem-template.yaml
```


