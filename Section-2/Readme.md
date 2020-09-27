# Upload Template Manifest (YAML)
## MySQL Enterprise Edition Template for deploying MySQL as StatefulSet
Copy and paste below YAML code and name it as mysql-generic-template.yaml
```
apiVersion: v1
kind: Template
metadata:
  name: 'mysql-generic'
  labels:
    app: mysqldb
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
      replicas: '${replicas}'
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
                - 
                  name: MYSQL_ROOT_PASSWORD
                  valueFrom:
                    secretKeyRef:
                       name: '${secretpassword}'
                       key: ROOT_PASSWORD
              ports:
                - containerPort: 3306
                  protocol: TCP
              volumeMounts:
                  - name: mysql-data-dir
                    mountPath: "/var/lib/mysql"
              resources:
                limits:
                  cpu: 1000m
                  memory: 2048Mi
                requests:
                  cpu: 1000m
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
          name: mysql-data-dir
        spec:
          accessModes: [ "ReadWriteOnce" ]
          resources:
            requests:
              storage: 1Gi        
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
  - name: replicas
    displayName: number of mysql nodes
    value: ''
    required: true
  - name: secretpassword 
    displayName: a secret that stores root password
    value: ''
    required: true
 ```
 Apply mysql-generic-template.yaml
 ```
 oc apply -f mysql-generic-template.yaml -n db-mysql-dev
 ```
 ## Service Template for MySQL Container
 Service is used as DNS lookup to connect to MySQL container since IP address is always dynamic in cloud native environment.
 Ensure service name for MySQL equals to Pod-name for simplicity of deployment from MySQL perspective. 
 Copy and paste below YAML code and name it as mysql-svc-template.yaml
 ```
 kind: "Template"
apiVersion: "v1"
metadata:
  name: mysql-svc
objects:
  - kind: Service
    apiVersion: v1
    metadata:
      name: '${nodename}'
      labels:
        app: '${nodename}'
    spec:
      clusterIP: None
      ports:
        -
          name: 3306-tcp
          protocol: TCP
          port: 3306
          targetPort: 3306
      selector:
        statefulset.kubernetes.io/pod-name: '${nodename}'
parameters:
  - name: namespace
    displayName: OpenShift namespace
    value: ''
    required: true
  - name: nodename
    displayName: node name
    value: ''
    required: true
```
Apply mysql-svc-template.yaml
```
oc apply -f mysql-svc-template.yaml -n db-mysql-dev
```
## Create Secret 
Secret is used to store default root password when deploying new MySQL container.
Copy and paste below YAML code and name it as secret.yaml.
```
apiVersion: v1
kind: Secret
metadata:
   name: mysqlsecret
   type: Opaque
data:
   ROOT_PASSWORD: cm9vdA==
```
The above code will set initial root password for new MySQL container as "root". Apply secret.yaml:
```
oc apply -f secret.yaml -n db-mysql-dev
```
## MySQL-Router template for deploying MySQL Router as statefulset
Copy and paste below YAML code and name it as router-generic-tamplate.yaml.
```
apiVersion: v1
kind: Template
metadata:
  name: 'router-generic'
  labels:
    app: mysqlrouter
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
              name: mysqlrouter
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
          restartPolicy: Always
          terminationGracePeriodSeconds: 30
          dnsPolicy: ClusterFirst
          securityContext:
          supplementalGroups:
              - 110
parameters:
  - name: namespace
    displayName: OpenShift namespace
    value: ''
    required: true
  - name: statefulsetname
    displayName: Data Node statefulset 
    value: ''
    required: true
  - name: dbnode
    displayName: cluster primary node for bootstrap
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
```
Apply router-generic-template.yaml
```
oc apply -f router-generic-template.yaml
```
## Service Template for MySQL Router Container
Service is used as DNS lookup to connect to MySQL Router container since IP address is always dynamic in cloud native environment.
Copy and paste below YAML code and name it as router-svc-template.yaml.
```
kind: "Template"
apiVersion: "v1"
metadata:
  name: router-svc
objects:
  - kind: Service
    apiVersion: v1
    metadata:
      name: '${nodename}'
      labels:
        app: '${nodename}'
    spec:
      clusterIP: None
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
          name: 80-tcp
          protocol: TCP
          port: 80
          targetPort: 80
      selector:
        statefulset.kubernetes.io/pod-name: '${nodename}'
parameters:
  - name: namespace
    displayName: OpenShift namespace
    value: ''
    required: true
  - name: nodename
    displayName: node name
    value: ''
    required: true
```
Apply router-svc-template.yaml
```
oc apply -f router-svc-template.yaml -n db-mysql-dev
```
