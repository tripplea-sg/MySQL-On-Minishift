# Deploy MySQL Router on Openshift as StatefulSet
## 1. Login to Openshift
```
oc login -u system -p admin
```
You will see that default project is set to "db-mysql-dev"
## 2. Deploy MySQL Router as StatefulSet
```
oc process -n db-mysql-dev router-generic -p namespace=db-mysql-dev -p statefulsetname=router-workgroup1 -p dbnode=workgroup1-0 -p secretpassword=mysqlsecret | oc create -f -
```
Syntax: oc process -n \<project> \<template> -p namespace=\<namespace> -p statefulsetname=\<statefulsetname> -p dbnode=\<currentClusterPrimaryNode> -p secretpassword=\<secret> | oc create -f -
## 3. Deploy Service for MySQL Router
```
oc process -n db-mysql-dev router-svc -p namespace=db-mysql-dev -p nodename=router-workgroup1-0 | oc create -f -
```
Syntax: oc process -n \<project> \<template> -p namespace=\<namespace> -p nodename=\<prodName> | oc create -f -
