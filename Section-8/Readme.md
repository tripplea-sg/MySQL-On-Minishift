# Deploy MySQL Router as StatefulSet
## 1. Login to Openshift
```
oc login -u system -p admin
```
You will see that default project is set to "db-mysql-dev"
## 2. Deploy MySQL Router as StatefulSet
```
oc process -n db-mysql-dev router-generic -p namespace=db-mysql-dev -p statefulsetname=router-workgroup1 -p dbnode=workgroup1-0 -p imageName=172.30.1.1:5000/db-mysql-dev/mysql-router -p mysqlpassword=grpass -p mysqluser=gradmin -p mysqlport=3306 | oc create -f -
```
Syntax: oc process -n \<project> \<template> -p namespace=\<namespace> -p statefulsetname=\<statefulsetname> -p dbnode=\<currentClusterPrimaryNode> -p imageName=\<imageName> -p mysqlpassword=\<clusterAdminPassword> -p mysqluser=\<clusterAdminUser> -p mysqlport=\<mysqlPort> | oc create -f -
## 3. Deploy Service for MySQL Router
```
oc process -n db-mysql-dev router-svc -p namespace=db-mysql-dev -p nodename=router-workgroup1-0 | oc create -f -
```
Syntax: oc process -n \<project> \<template> -p namespace=\<namespace> -p nodename=\<prodName> | oc create -f -
## 4. Testing
```
oc exec -it workgroup1-0 -- mysql -ugradmin -pgrpass -hrouter-workgroup1-0 -P6446
oc exec -it workgroup1-0 -- mysql -ugradmin -pgrpass -hrouter-workgroup1-0 -P6447
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@router-workgroup1-0:64460
oc exec -it workgroup1-0 -- mysqlsh gradmin:grpass@router-workgroup1-0:64460
```
