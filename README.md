# longDB setup tutorial
## download and install

* 環境需求：
  * OS：CentOS7
  * CPU： 4 Cores
  * RAM：26 GB
* software required：
  * git
  * docker 
  * docker-compose

```bash
#download hadoop and jupyter's images
cd /tmp; git clone https://github.com/orozcohsu/webRecommend.git

# start containers
cd /tmp/webRecommend/; docker-compose up -d

#check containers' status
cd /tmp/webRecommend/; docker-compose ps

# pull longDB's image and enter into container
docker run --network="webrecommend_iii_net" \
-it --sysctl net.ipv6.conf.all.disable_ipv6=1 \
--expose 1527 --name longdb --hostname localhost \
-p 1527:1527 -p 4040:4040 -p 7078:7078 -p 8080:8080 \
-p 8090:8090 -p 8091:8091 -p 4041:4041 -p 8081:8081 \
-p 8082:8082 longdb/longdb:v1.0.9

# start/stop service(in container's command)
./start-longdb.sh 
./stop-longdb.sh 
```


## check longDB's status using DBeaver
  * driver choose：ODBC
  * advance setting
    * Class Name: com.splicemachine.db.jdbc.ClientDriver
    * URL Template: jdbc:splice://_*HostIP*_:1527/splicedb
    * Description:longdb
    * Port: 1527
    * db driver download URL: http://repository.splicemachine.com/nexus/content/groups/public/com/splicemachine/db-client/2.7.0.1815/db-client-2.7.0.1815.jar
  * default user/password
    * splice/admin
    
    
## How to use LongDB
### LongDB is base on SparkSQL
#### official document: https://doc.splicemachine.com/
```sql
-- insert data
INSERT INTO 'db'.'table'('colnames1', 'colnames2', ...)VALUES('value1', 'value2', ...); 

-- update data
UPDATE 'db'.'table' SET 'colname'='value' WHERE 'colname'='value';

-- delete data
DELETE FROM 'db'.'table' WHERE 'colname'='value';
```


