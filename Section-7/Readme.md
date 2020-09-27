# MySQL InnoDB Cluster on Openshift
We will setup InnoDB Cluster on top of existing workgroup1-0 - workgroup1-2
## 1. Login to Openshift
```
oc login -u system -p admin
```
You will see that default project is set to "db-mysql-dev"
## 2. Prepare 3 instances for InnoDB Cluster
```
oc exec -it workgroup1-0 -- mysqlsh -- dba configure-instance { --user=root --password=root --host=localhost } --clusterAdmin=gradmin --clusterAdminPassword=grpass --interactive=false --restart=true

oc exec -it workgroup1-1 -- mysqlsh -- dba configure-instance { --user=root --password=root --host=localhost } --clusterAdmin=gradmin --clusterAdminPassword=grpass --interactive=false --restart=true

oc exec -it workgroup1-2 -- mysqlsh -- dba configure-instance { --user=root --password=root --host=localhost } --clusterAdmin=gradmin --clusterAdminPassword=grpass --interactive=false --restart=true
```
Syntax: oc exec -it \<podName> -- mysqlsh -- dba configure-instance { --user=root --password=\<rootpassword> --host=localhost } --clusterAdmin=\<clusterAdmin-anyName> --clusterAdminPassword=\<clusterAdminPassowrd-anyName> --interactive=false --restart=true
## 3. Create cluster on workgroup1-0
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- dba createCluster 'myCluster'
```
Syntax: oc exec -it \<podName> -- mysqlsh gradmin:grpass@localhost:3306 -- dba createCluster '\<anyNameFor-ClusterName>'
## 4. Add instance workgroup1-1 into cluster
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@workgroup1-1:3306 --recoveryMethod=clone
```
Syntax: oc exec -it \<podName> -- mysqlsh \<clusterAdmin>:\<clusterAdminPassword>@localhost:3306 -- cluster add-instance \<clusterAdmin>:\<clusterAdminPassword>@\<secondNode>:3306 --recoveryMethod=clone
## 5. Add instance workgroup1-2 into cluster
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@workgroup1-2:3306 --recoveryMethod=clone
```
Syntax: oc exec -it \<podName> -- mysqlsh \<clusterAdmin>:\<clusterAdminPassword>@localhost:3306 -- cluster add-instance \<clusterAdmin>:\<clusterAdminPassword>@\<thirdNode>:3306 --recoveryMethod=clone
## 6. View cluster status
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Yay! You have already setup an InnoDB Cluster on Openshift, congratulations !

# High Availability Testing
## 7. Switch Primary Node from workgroup1-0 to workgroup1-1
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster setPrimaryInstance gradmin:grpass@workgroup1-1:3306
```
Syntax: oc exec -it \<prodName> -- mysqlsh \<clusterAdmin>:\<clusterAdminPassword>@localhost:3306 -- cluster setPrimaryInstance \<clusterAdmin>:\<clusterAdminPassword>@\<newPrimaryPodName>:3306
Check cluster status, you will find primary node is now workgroup1-1
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
## 8. Test fail-scenario on secondary node (use workgroup1-2)
```
oc delete pod workgroup1-2
```
During failure, cluster is still up and database instance is available\
Check cluster status, you will see that failed node is auto-joined to cluster again without manual intervention
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
## 9. Test fail-scenario on Primary node (use workgroup1-1)
```
oc delete pod workgroup1-1
```
During failure, primary node will be failover to another node\
Check cluster status, you will see that new primary node is elected and the failed node is auto-joined to cluster again without manual intervention
```
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
