## 配置主集群
### 创建主机用户
>
```
mkdir -p /data/crm
chmod 755 /data/crm
useradd -d /data/crm/crmcache crmcache
passwd crmcache
chown -R crmcache:crmcache /data/crm/crmcache
su - crmcache
```


### 下载安装包
```
http://download.redis.io/releases/redis-4.0.1.tar.gz下载地址：
上传安装包至用户目录下
如果主机可上外网则可以直接用wget下载安装包，否则只能下载到本机后用ftp或scp工具上传
```
### 解压编译安装包
```
tar -zxf redis-4.0.1.tar.gz
cd redis-4.0.1
make
cd src
make install PREFIX=/data/crm/crmcache/redis
mkdir -p /data/crm/crmcache/redis/conf
mv redis.conf /data/crm/crmcache/redis/conf
cd /data/crm/crmcache
rm -rf redis-4.0.1
```
### 创建rdb目录
```
mkdir -p /data/crm/crmcache/redis/rdb/
```
### 创建日志目录
```
mkdir -p /data/crm/crmcache/redis/logs/
```
### 修改redis.conf配置文件
```
cd /data/crm/crmcache/redis/conf
cat>redis.conf
#绑定ip，需要个性化
bind 192.191.100.36
#启用保护模式
protected-mode yes
#绑定的服务端口，需要个性化
port 6379
tcp-backlog 511
#日志级别debug # verbose# notice# warning
loglevel warning
#redis连接多久不连接就关闭,单位秒,0表示不断开
timeout 0
#用于检测连接是否挂死，每多少秒发送一个ack
tcp-keepalive 60
daemonize no
stop-writes-on-bgsave-error no
rdbcompression no
#本地文件存储的数据，需要个性化
dbfilename dump.rdb
#本地数据存储文件dbfilename的目录
dir /data/crm/crmcache/redis/rdb
#开启日志文件
logfile /data/crm/crmcache/redis/logs/redis.log
#pidfile
pidfile /data/crm/crmcache/redis/redis.pid
#配置连接数
maxclients 20000
#配置使用内存
maxmemory 4gb
#内存过期策略
maxmemory-policy volatile-lru
#需要个性化
appendfilename appendonly.aof
appendfsync everysec
no-appendfsync-on-rewrite no
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
######集群配置模式

#开启集群模式
cluster-enabled yes
#运行过程中集群信息保存的文件名，不能冲突，需要个性化
cluster-config-file nodes.conf
#集群节点通信内限定的超时时间
cluster-node-timeout 9000
#以追加的方式写数据
appendonly yes
#在某个主节点挂死的情况下，其他主节点仍然可以工作
cluster-require-full-coverage no
```
### 创建启动脚本
```
cd /data/crm/crmcache/redis
cat > run.sh 
DIR=`dirname $0`
$DIR/bin/redis-server $DIR/conf/redis.conf &
```
### 创建停止脚本
```
cd /data/crm/crmcache/redis
cat > kill.sh 
DIR=`dirname $0`
PID=`cat $DIR/redis-6379.pid`
if [ x`ps -ef | grep $PID | grep -v grep` == x ]; then
        echo "redis not runing ,exit!"
        exit
fi
DIR=`dirname $0`
echo "`ps -ef | grep $PID | grep -v grep`"
echo "kill -9 $PID, yes or no"
read  -t3 a
if [ x$a == xyes ]; then
        kill -9 $PID
else
        echo "not kill "
fi
```
### 启动redis
```
cd /data/crm/crmcache/redis
sh run.sh
```
### 检查启动情况
```
ps -ef|grep redis|grep -v grep
```

### 搭建集群
```
所有master节点都按照以上步骤安装部署完毕后，即可执行以下流程配置主集群
登录其中一个master节点执行以下命令
cd /data/crm/crmcache/redis/bin
./redis-cli -c -p 6379 -h 192.191.100.36 cluster meet 192.191.100.43 6379
./redis-cli -c -p 6379 -h 192.191.100.36 cluster meet 192.191.100.51 6379
./redis-cli -c -p 6379 -h 192.191.100.43 cluster meet 192.191.100.36 6379
./redis-cli -c -p 6379 -h 192.191.100.43 cluster meet 192.191.100.51 6379
./redis-cli -c -p 6379 -h 192.191.100.51 cluster meet 192.191.100.36 6379
./redis-cli -c -p 6379 -h 192.191.100.51 cluster meet 192.191.100.43 6379
```
### 分配slot
```
cd /data/crm/crmcache/redis/bin
start_slots=0
end_slots=5461
i=${start_slots}
while [ $i -le $end_slots ]
do
   ./redis-cli -c -h 192.191.100.36 -p 6379 cluster addslots ${i}
   i=$(($i+1))
done

start_slots=5462
end_slots=10921
i=${start_slots}
while [ $i -le $end_slots ]
do
   ./redis-cli -c -h 192.191.100.43 -p 6379 cluster addslots ${i}
   i=$(($i+1))
done

start_slots=10922
end_slots=16383
i=${start_slots}
while [ $i -le $end_slots ]
do
   ./redis-cli -c -h 192.191.100.51 -p 6379 cluster addslots ${i}
   i=$(($i+1))
done
```
### 查看主集群信息
```
cd /data/crm/crmcache/redis/bin
./redis-cli -c -h 192.191.100.36 -p 6379 cluster nodes
```

## 配置从集群
###  启动前步骤与主集群一致（端口为7379）：
```
1.副集群无需分配slot
2.启动副集群redis，然后登陆：
cd /data/crm/crmcache/redis/bin/
./redis-cli -c -p 7379 -h 192.191.100.36 cluster meet 192.191.100.36 6379 #连接主节点的ip和端口
./redis-cli -c -p 7379 -h 192.191.100.36 cluster nodes #查看主从的nodes
./redis-cli -c -p 7379 -h 192.191.100.36 cluster replicate 08b0b22104f6240d40490782ac1f27786fd1337f #建立主从连接
./redis-cli -c -p 7379 -h 192.191.100.36 readonly #从节点设置为只读
```

### 启动zookeeper
```
cd /data/crm/crmcache/redis
redis-server /data/crm/crmcache/redis/conf/redis.conf
```
### 关闭zookeeper
```
cd /data/crm/crmcache/redis
redis-cli shutdown
或者
pkill redis-server
```
### 查看集群信息
```
cd /data/crm/crmcache/redis/bin
./redis-cli -c -h 192.191.100.36 -p 6379 cluster nodes
 ```




bin目录下的几个文件
```
　　redis-benchmark：redis性能测试工具
　　redis-check-aof：检查aof日志的工具
　　redis-check-dump：检查rdb日志的工具
　　redis-cli：连接用的客户端
　　redis-server：redis服务进程
```
