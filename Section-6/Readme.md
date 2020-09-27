# Clone Database
We will setup mysql replication from workgroup1-0 (source) to workgroup1-1 (replica)
## 1. Login to Openshift
```
oc login -u system -p admin
```
You will see that default project is set to "db-mysql-dev"
## 2. Clean up environment
```
oc delete svc workgroup1-2
oc delete svc workgroup1-1
oc delete svc workgroup1-0
oc delete statefulset workgroup1
```
## 3. Wait for a while (upto 1 minutes), and check if services and pods are removed
```
oc get svc
oc get pod
```
You should not see any services or pods with name like 'workgroup1'
## 4. Create statefulset with PV/PVC (replicas=3)
```
oc process -n db-mysql-dev mysql-generic -p namespace=db-mysql-dev -p imageName=172.30.1.1:5000/db-mysql-dev/mysql-enterprise-server-8021:latest -p statefulsetname=workgroup1 -p replicas=3 -p secretpassword=mysqlsecret | oc create -f - 
```
## 5. Create services for each pod
```
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-0 | oc create -f -
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-1 | oc create -f -
oc process -n db-mysql-dev mysql-svc -p namespace=db-mysql-dev -p nodename=workgroup1-2 | oc create -f -
```
## 6. Create database demo and table demo, insert transaction to table demo on workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "create database demo; create table demo.demo (i int primary key); insert into demo.demo values (1); insert into demo.demo values (2); insert into demo.demo values (3);"
```
## 7. select table demo.demo on workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "select * from demo.demo;"
```
## 8. Prepare workgroup1-0 as clone donor (1 time setup)
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "install plugin clone soname 'mysql_clone.so'; create user clone@'%' identified by 'clone'; grant backup_admin on *.* to clone@'%';"
```
## 9. Prepare workgroup1-1 as clone recipient
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "install plugin clone soname 'mysql_clone.so';"
```
## 10. Execute clone from workgroup1-1 
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "set persist clone_valid_donor_list='workgroup1-0:3306'; clone instance from clone@'workgroup1-0':3306 identified by 'clone';"
```
## 11. select table demo.demo on workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "select * from demo.demo;"
```
## 12. Now add new record on table demo.demo of workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "insert into demo.demo values (4); insert into demo.demo values (5); insert into demo.demo values (6);"
```
## 13. Select table demo.demo on workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "select * from demo.demo;"
```
## 14. Clone again from workgroup1-0 to workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "set persist clone_valid_donor_list='workgroup1-0:3306'; clone instance from clone@'workgroup1-0':3306 identified by 'clone';"
```
## 15 Check table demo.demo on workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "select * from demo.demo;"
```
