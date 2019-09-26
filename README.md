# longDB setup tutorial
## download and install
```text
環境需求：
    OS：CentOS7
    CPU： 4 Cores
    RAM：26 GB
software：
    git
    docker 
    docker-compose
```
```zsh
#download hadoop and jupyter's images
cd /tmp
git clone https://github.com/orozcohsu/webRecommend.git

# start containers
cd /tmp/webRecommend/
docker-compose up -d

#check containers' status
cd /tmp/webRecommend/
docker-compose ps

# pull longDB's image
docker run --network="webrecommend_iii_net" -it --sysctl net.ipv6.conf.all.disable_ipv6=1 --expose 1527 --name longdb --hostname localhost -p 1527:1527 -p 4040:4040 -p 7078:7078 -p 8080:8080 -p 8090:8090 -p 8091:8091 -p 4041:4041 -p 8081:8081 -p 8082:8082 longdb/longdb:v1.0.9


```
