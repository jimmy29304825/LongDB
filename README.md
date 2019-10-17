# LongDB setup tutorial
## Download and Install
* 環境需求：
  * OS：CentOS7
  * CPU： 4 Cores
  * RAM：16 GB
* Software Required：
  * git
  * docker 
  * docker-compose

```bash
# Download hadoop and jupyter's images
cd /tmp; git clone https://github.com/orozcohsu/webRecommend.git

# Start hadoop and jupyter's containers
cd /tmp/webRecommend/; docker-compose up -d

# Check containers' status
cd /tmp/webRecommend/; docker-compose ps

# Pull longDB's image and enter into container
docker run --network="webrecommend_iii_net" \
-it --sysctl net.ipv6.conf.all.disable_ipv6=1 \
--expose 1527 --name longdb --hostname localhost \
-p 1527:1527 -p 4040:4040 -p 7078:7078 -p 8080:8080 \
-p 8090:8090 -p 8091:8091 -p 4041:4041 -p 8081:8081 \
-p 8082:8082 longdb/longdb:v1.1.1

# Start/Stop service(in container's command)
./start-longdb.sh 
./stop-longdb.sh 
```


## Connect LongDB using DBeaver
* Download URL: https://dbeaver.io/
* Driver choose：ODBC
* Advance setting
  * Class Name: com.splicemachine.db.jdbc.ClientDriver
  * URL Template: jdbc:splice://_*HostIP*_:1527/splicedb
  * Description:longdb
  * Port: 1527
  * DB driver download URL: http://repository.splicemachine.com/nexus/content/groups/public/com/splicemachine/db-client/2.7.0.1815/db-client-2.7.0.1845.jar
* Default user/password
  * splice/admin
    
    
## How to use LongDB
### LongDB is base on SparkSQL
#### Official document: https://doc.splicemachine.com/
##### OLTP sample
```sql
-- insert
INSERT INTO `db`.`table`(`col_names1`, `col_names2`, ...)VALUES(`value1`, `value2`, ...); 

-- update
UPDATE `db`.`table` SET `col_name`=`value` WHERE `col_name`=`value`;

-- delete
DELETE FROM `db`.`table` WHERE `col_name`=`value`;
```
##### OLAP sample
```sql
select * from `db`.`table`
```


## Jupyter Notebook connect LongDB(using ODBC package)
```sh
# Enter to jupyter's container
sudo docker exec -it jupyter bash

# Download ODBC package
cd /tmp; wget https://splice-releases.s3.amazonaws.com/odbc-driver/Linux64/splice_odbc_linux64-2.7.62.0.tar.gz

# Untar file
tar -zxvf splice_odbc_linux64-2.7.62.0.tar.gz

# Install package
cd splice_odbc_linux64-2.7.62.0; ./install.sh

# Edit odbc.ini file 
# change URL = 172.28.0.2
nano ./odbc.ini
```

```ipnbpython
import pyodbc
# Connect to LongDB
cnxn=pyodbc.connect("DSN=SpliceODBC64")

# Call sql command
cursor=cnxn.cursor()

# Run sql command
cursor.execute("select * from sys.systables")

# Get one row out
row=cursor.fetchone()
print('row:',row)

# Get all row out
row=cursor.fetchall()
print('row:',row)
```

## Sqoop
* Sqoop is a tool which can import data, export data between hadoop hdfs and database
```bash
# Enter into hadoop container and clone git
sudo docker exec -it hadoop bash
cd /tmp; git clone https://github.com/orozcohsu/webRecommend.git

# Copy file to sqoop's path
cp /tmp/webRecommend/db-client-2.7.0.1815.jar /usr/local/sqoop/lib/

# Database -> Hadoop
sqoop import --connect jdbc:splice://172.28.0.2:1527/splicedb \
--username splice --password admin \
--driver com.splicemachine.db.jdbc.ClientDriver \
--query "SELECT [col_names] FROM `db`.`table` where 1=1 and \$CONDITIONS" \
-target-dir $PATH_IN_HADOOP -m1

# Check data
## by command
hadoop fs -ls /user/hdfs/data/weblog
## or visit http://HostIP:50070

# hadoop -> Database
sqoop export --connect jdbc:splice://172.28.0.2:1527/splicedb \
--username splice --password admin \
--driver com.splicemachine.db.jdbc.ClientDriver 
--export-dir $DATA_PATH_IN_HADOOP \
--table `db`.`table` \
--input-fields-terminated-by "," --columns "[col_names]"

# Then you can check data on DBeaver...
```

## Mahout
* due to the mahout version in the image we use is too old, we have to download another version
```sh
# Enter into hadoop container
sudo docker exec -it hadoop bash

# Download and untar
cd /tmp; wget https://archive.apache.org/dist/mahout/0.10.1/apachemahout-distribution-0.10.1.tar.gz
tar -zxvf apache-mahout-distribution-0.10.1.tar.gz

# Go to mahout's path and run
cd apache-mahout-distribution-0.10.1/bin
./mahout
## Check the running version is correct

# Set the maxinum RAM usage for Mahout
export MAHOUT_HEAPSIZE=10000m

# ALS-WR algorithm sample
# Build model
./mahout parallelALS --input /user/hdfs/data/weblog/* \
--output /user/hdfs/data/logout \
--tempDir /user/hdfs/data/tmp \
--numFeatures 5 \
--numIterations 2 \
--lambda 0.065

# Check the result model
hadoop fs -ls /user/hdfs/data/logout
## There should have three directorys in the logout path
### M 目錄: itemFeatures
### U 目錄: userFeatures
### userRatings 目錄

# Use model
./mahout recommendfactorized \
--input /user/hdfs/data/logout/userRatings/ \
--output /user/hdfs/data/recommend/ \
--userFeatures /user/hdfs/data/logout/U \
--itemFeatures /user/hdfs/data/logout/M \
--numRecommendations 3 \
--maxRating 5 \
--numThreads 2

# get result data from Hadoop to local
hadoop fs -cat /user/hdfs/data/recommend/* > /tmp/result.txt

# Make a directory for final result data
hadoop fs -mkdir /user/hdfs/data/final

# ETL result with python and put back into hadoop hdfs
cd /tmp/webRecommend && \
python mahout_to_longdb.py && \
hadoop fs -put -f /tmp/mahout_result.csv /user/hdfs/data/final

# Export data from Hadoop to LongDB
sqoop export --connect jdbc:splice://172.28.0.2:1527/splicedb \
--username splice --password admin \
--driver com.splicemachine.db.jdbc.ClientDriver \
--export-dir /user/hdfs/data/final/mahout_result.csv \
--table SPLICE.RECOMMEND --input-fields-terminated-by "," \
--columns "UUID, ITEM, SCORE"

```
