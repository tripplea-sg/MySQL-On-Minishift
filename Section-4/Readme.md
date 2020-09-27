# Backup MySQL on Openshift using MySQL Shell instance dump

## 1. Login to Openshift
```
oc login -u system -p admin
```
you will see that default project is set to "db-mysql-dev"

## 2. Use workgroup1-2 as backup server. 
Prepare directory /backup on workgroup1-2
```
oc exec -it workgroup1-2 -- mkdir -p /backup
```
check directory backup (it should be empty)
```
oc exec -it workgroup1-2 -- ls /backup
```
## 3. Backup workgroup1-0 to workgroup1-2
Backup command:
```
oc exec -it workgroup1-2 -- mysqlsh demo:demo@workgroup1-0:3306 -e "util.dumpInstance('/backup')"
```
syntax: oc exec -it \<backupserver> -- mysqlsh \<user>:\<password>@\<dbserver>:\<port> -e "util.dumpInstance('\<backup folder>')"\
Check directory /backup again (it shouldn't be empty anymore)
```
oc exec -it workgroup1-2 -- ls /backup
```
## 4. Restore backup to workgroup1-1
Prepare user on workgroup1-1 and set mandatory local_infile=on
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "create user root@'%' identified by 'root'; grant all privileges on *.* to root@'%' with grant option; set persist local_infile=on;"
```
start restoration
```
oc exec -it workgroup1-2 -- mysqlsh root:root@workgroup1-1:3306 -e "util.loadDump('/backup')"
```
syntax: oc exec -it \<backupserver> -- mysqlsh \<user>:\<password>@\<targetserver>:\<port> -e "util.loadDump('\<backup folder')"
## 5. check if database and table are exist on workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "select * from demo.demo;"
```
