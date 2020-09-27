# MySQL Asynchronous Replication on Openshift
We will setup mysql replication from workgroup1-0 (source) to workgroup1-1 (replica)
## 1. Login to Openshift
```
oc login -u system -p admin
```
You will see that default project is set to "db-mysql-dev"

## 2. Set GTID_MODE=ON on workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "set persist gtid_mode=OFF_PERMISSIVE; set persist gtid_mode=ON_PERMISSIVE; set persist enforce_gtid_consistency=ON; set persist gtid_mode=ON;"
```
## 3. Set GTID_MODE=ON  workgroup1-1 and server_id=2
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "set persist gtid_mode=OFF_PERMISSIVE; set persist gtid_mode=ON_PERMISSIVE; set persist enforce_gtid_consistency=ON; set persist gtid_mode=ON; set persist server_id=2;"
```
## 4. Setup replication user on workgroup1-0
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "create user repl@'%' identified with mysql_native_password by 'repl'; grant replication slave on *.* to repl@'%'"
```
## 5. Create replication channel on workgroup1-1 pointing to workgroup1-0
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "change master to master_user='repl', master_password='repl', master_host='workgroup1-0', master_port=3306, master_auto_position=1 for channel 'channel1';"
```
## 6. Start replication channel on workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "start slave for channel 'channel1'"
```
## 7. View replication channel status on workgroup1-1
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "show slave status for channel 'channel1' \G"
```
Look at the status of Slave_IO_Running and Slave_SQL_Running, the values shall be all "Yes"
## 8. Insert new transaction into table demo.demo on workgroup1-0 (the source)
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "insert into demo.demo values (4); insert into demo.demo values (5); insert into demo.demo values (6);"
```
## 9. Select table demo.demo on workgroup1-0 (the source)
```
oc exec -it workgroup1-0 -- mysql -uroot -proot -e "select * from demo.demo;"
```
## 10. Select table demo.demo on workgroup1-1 (the replica)
```
oc exec -it workgroup1-1 -- mysql -uroot -proot -e "select * from demo.demo;"
```
