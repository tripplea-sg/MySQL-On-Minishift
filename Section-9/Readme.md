# Deploy MySQL Router on Openshift as DeploymentConfig
## MySQL Router Template 
Copy and paste below YAML code and name it as router-dc-generic.yaml.
```
apiVersion: v1
kind: Template
metadata:
  name: 'router-dc-generic'
  labels:
    app: mysqlrouter-dc
objects:
  - kind: deploymentConfig 
    apiVersion: apps/v1
    metadata:
      name: '${dcname}'
      labels:
        app: '${dcname}'
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
              name: '${imageName}' 
      replicas: '${replicas}'
      test: false
      selector:
        matchLabels:
          app: '${dcname}'
      template:
        metadata:
          labels:
            app: '${dcname}'
        spec:
          containers:
            -   
              name: mysqlrouter-dc
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
  - name: dcname
    displayName: Data Node statefulset 
    value: ''
    required: true
  - name: dbnode
    displayName: cluster primary node for bootstrap
    value: ''
    required: true
  - name: secretpassword 
    displayName: a secret that stores root password
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

Service Template for MySQL Router Container
Service is used as DNS lookup to connect to MySQL Router container since IP address is always dynamic in cloud native environment. Copy and paste below YAML code and name it as router-svc-template.yaml.

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
