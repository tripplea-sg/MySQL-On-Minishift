# Deploy MySQL Router on Openshift as DeploymentConfig
## MySQL Router Template 
Copy and paste below YAML code and name it as mysql-generic-template.yaml

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
Apply mysql-generic-template.yaml

oc apply -f mysql-generic-template.yaml -n db-mysql-dev
Service Template for MySQL Container
Service is used as DNS lookup to connect to MySQL container since IP address is always dynamic in cloud native environment. Ensure service name for MySQL equals to Pod-name for simplicity of deployment from MySQL perspective. Copy and paste below YAML code and name it as mysql-svc-template.yaml

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
Apply mysql-svc-template.yaml

oc apply -f mysql-svc-template.yaml -n db-mysql-dev
