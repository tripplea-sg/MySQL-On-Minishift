# Deploy MySQL Router on Openshift as DeploymentConfig
## MySQL Router Template 
Copy and paste below YAML code and name it as router-generic-tamplate.yaml.

apiVersion: v1
kind: Template
metadata:
  name: 'router-dc-generic'
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
              image: 172.30.1.1:5000/db-mysql-dev/mysql-router
              env:
                - 
                  name: MYSQL_PASSWORD
                  value: grpass
                - 
                  name: MYSQL_USER
                  value: gradmin
                - 
                  name: MYSQL_PORT
                  value: "3306"
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
  - name: secretpassword 
    displayName: a secret that stores root password
    value: ''
    required: true
Apply router-generic-template.yaml

oc apply -f router-generic-template.yaml
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
