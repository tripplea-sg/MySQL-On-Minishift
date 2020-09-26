# Deployment of MySQL on Openshift

## 1. Login to Openshift
```
oc login -u system -p admin
```

## 2. Check project name
```
oc get project
```
The output shall show "db-mysql-dev"

## 3. Check the deployed templates for this workgroup
```
oc get templates
```
The output will show list of templates have been added into the openshift:
 1. mysql-generic	        : template to deploy MySQL with n replicas
 2. mysql-svc             : template to deploy service for MySQL
 3. router-generic        : template to deploy router and application
 4. router-svc            : template to deploy service for router and application

## 4. Check secret
```
oc get secret
```
The mysqlsecret will be used for root password. The value of mysqlsecret is "root"

## 5. Check images
```
oc get is
```
The output will show list of installed images in the Openshift internal registry:
1. 172.30.1.1:5000/db-mysql-dev/mysql-enterprise-server-8021:latest : this image is MySQL 8.0.21
2. 172.30.1.1:5000/db-mysql-dev/mysql-router	                      : this image is MySQL router

## 6. Deploy a statefulset with 3 replicas 
```
oc process -n db-mysql-dev mysql-generic -p namespace=db-mysql-dev -p imageName=172.30.1.1:5000/db-mysql-dev/mysql-enterprise-server-8021:latest -p statefulsetname=workgroup1 -p replicas=3 -p secretpassword=mysqlsecret | oc create -f -
```
Syntax:
oc process -n \<project> \<template> -p namespace=\<namespace> -p imageName=\<image> -p statefulsetname=\<clustername> -p replicas=\<numberOfReplicas> -p secretpassword=\<secret> | oc create -f -

## 7. Deploy services for the 3 replicas
```
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-0 | oc create -f -
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-1 | oc create -f -
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-2 | oc create -f -
```
syntax: 
oc process -n <project> <template> -p namespace=<namespace> -p nodename=<targetNode> | oc create -f -

## 8. Login to workgroup1-0 and create user demo
```
oc exec -it workgroup1-0 -- mysql -uroot -proot
```
syntax:
oc exec -it <PodName> -- mysql -u<user> -p<password>

On SQL, create user:
```
mysql > create user demo@'%' identified with mysql_native_password by 'demo';
mysql > grant all privileges on *.* to demo@'%';
```

## 9. Play with the database workgroup1-0
Still on first node, create database demo, create table demo, and insert data into table demo
```
mysql > create database demo;
mysql > create table demo.demo (i int primary key);
mysql > insert into demo.demo values (1);
mysql > insert into demo.demo values (2);
mysql > insert into demo.demo values (3);
mysql > exit;
```

## 10. Connect to database workgroup1-0 from workgroup1-1
Try to connect to workgroup1-1, then run mysql connects to workgroup1-0 as "demo" user
```
oc exec -it workgroup1-1 -- mysql -udemo -pdemo -hworkgroup1-0
```
syntax: oc exec -it <PodName> -- mysql -u<user> -p<password> -h<mysqlserver>
Check which server is actually connecting to, and query table demo.demo
```
mysql > select @@hostname;
```
It should show workgroup1-0
```
mysql > select * from demo.demo;
mysql > exit;
``` 
Now, actually SQL commands can be executed in single command line from "oc"
```
oc exec -it workgroup1-0 -- mysql -udemo -pdemo -hworkgroup1-0 -e "select @@hostname; select * from demo.demo;"
oc exec -it workgroup1-1 -- mysql -udemo -pdemo -hworkgroup1-0 -e "select @@hostname; select * from demo.demo;"
```
