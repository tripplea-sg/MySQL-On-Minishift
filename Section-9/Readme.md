# Deploy MySQL Router on Openshift as DeploymentConfig
## MySQL Router Template and Load Balencer Service
Copy and paste below YAML code and name it as router-dc-generic.yaml.
```
apiVersion: v1
kind: Template
metadata:
  name: 'router-dc-generic'
  labeles:
    app: mysql-router
objects:
  - kind: DeploymentConfig
    apiVersion: v1
    metadata:
      name: '${dcname}'
      labels:
        app: mysql-router
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
              - '${dcname}'
            from:
              kind: ImageStreamTag
              namespace: '${namespace}'
              name: '${ImageStreamName}'
      replicas: '${replicas}'
      test: false
      selector:
        app: mysql-router
        deploymentconfig: '${dcname}'
      template:
        metadata:
          creationTimestamp: null
          labels:
            app: mysql-router
            deploymentconfig: '${dcname}'
        spec:
          containers:
            -
              name: '${dcname}'
              image: '${imageName}'
              env:
                - 
                  name: MYSQL_PASSWORD
                  value: '${mysqlpassword}'
                - 
                  name: MYSQL_USER
                  value: '${mysqluser}'
                - 
                  name: MYSQL_PORT
                  value: '${mysqlport}'
                - 
                  name: MYSQL_HOST
                  value: '${dbnode}'
                - 
                  name: MYSQL_INNODB_NUM_MEMBERS
                  value: "3"
              command:
                - "/bin/bash"
                - "-cx"
                - "exec /run.sh mysqlrouter"  
              ports:
                -
                  name: 6446-tcp
                  containerPort: 6446
                  protocol: TCP
                -
                  name: 6447-tcp
                  containerPort: 6447
                  protocol: TCP
                -
                  name: 64460-tcp
                  containerPort: 64460
                  protocol: TCP
                -
                  name: 64470-tcp
                  containerPort: 64470
                  protocol: TCP
          restartPolicy: Always
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          securityContext:
            supplementalGroups:
              - 1100
  - kind: Service
    apiVersion: v1
    metadata:
      name: '${dcname}'
      labels:
        app: mysql-router
    spec:
      ports:
        -
          name: 6446-tcp
          protocol: TCP
          port: 6446
          targetPort: 6446  
        -
          name: 6447-tcp
          protocol: TCP
          port: 6447
          targetPort: 6447
        -
          name: 64460-tcp
          protocol: TCP
          port: 64460
          targetPort: 64460
        -
          name: 64470-tcp
          protocol: TCP
          port: 64470
          targetPort: 64470
      type: LoadBalancer
      selector:
        app: mysql-router
        deploymentconfig: '${dcname}'
parameters:
  - name: namespace
    displayName: OpenShift namespace
    value: ''
    required: true
  - name: dcname
    displayName: Data Node statefulset
    value: ''
    required: true
  - name: dbnode
    displayName: cluster primary node for bootstrap
    value: ''
    required: true
  - name: ImageStreamName
    displayName: image stream name for mysql router
    value: ''
    required: true
  - name: imageName
    displayName: docker image for MySQL Router in Openshift repository
    value: ''
    required: true
  - name: mysqlpassword
    displayName: innodb Cluster admin password
    value: ''
    required: true
  - name: mysqluser
    displayName: innodb Cluster admin user
    value: ''
    required: true
  - name: mysqlport
    displayName: innodb Cluster port
    value: ''
    required: true
  - name: replicas
    displayName: number of router
    value: ''
    required: true
```
Apply router-dc-generic.yaml
```
oc apply -f router-dc-generic.yaml
```
## Deploy MySQL Router as DeploymentConfig and Create Load Balancer Service
```
oc process -n db-mysql-dev router-dc-generic -p namespace=db-mysql-dev -p dcname=routerdc -p dbnode=workgroup1-0 -p ImageStreamName=mysql-router:latest -p imageName=172.30.1.1:5000/db-mysql-dev/mysql-router:latest -p mysqlpassword=grpass -p mysqluser=gradmin -p mysqlport=3306 -p replicas=2 | oc create -f -
```
Syntax: oc process -n \<namespace> \<template> -p namespace=\<namespace> -p dcname=\<routerName> -p dbnode=\<InnoDB Cluster Primary Node pod name> -p ImageStreamName=<imageStreamName> -p imageName=<imageName> -p mysqlpassword=\<clusterAdminPassword> -p mysqluser=\<clusterAdmin> -p mysqlport=\<MySQLPort> -p replicas=\<NoOfReplicas> | oc create -f -
  
## Testing 
```
oc exec -it workgroup1-0 -- mysql -ugradmin -pgrpass -hrouterdc -P6446
oc exec -it workgroup1-0 -- mysql -ugradmin -pgrpass -hrouterdc -P6447
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@routerdc:64460
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@routerdc:64470
```
