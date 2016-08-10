# How to config redis-cluster for SoftnixLogger [3 nodes]

## Do after install SLGInstallers

### Disable default redis service
```
service redis stop
chkconfig redis off
```

### Make directory
```
mkdir -p /etc/redis/redis-master/data
mkdir -p /etc/redis/redis-slave/data
```

### Copy config
* Copy redis-master.conf in /etc/redis/redis-master to /etc/redis/redis-master
* Copy redis-slave.conf in /etc/redis/redis-slave to /etc/redis/redis-slave
* Copy redis-master in /etc/init.d to /etc/init.d
* Copy redis-slave in /etc/init.d to /etc/init.d

#### Chmod
```
chmod +x /etc/init.d/redis-master
chmod +x /etc/init.d/redis-slave
```

### Enable redis-master,redis-slave service
```
chkconfig redis-master on
chkconfig redis-slave on
```

### Start redis-master **[all nodes]**
```
service redis-master start
```
#### Verify redis-master is running **[all nodes]**
```
redis-cli cluster nodes
```
###### Result
```
9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 :6379 myself,master - 0 0 0 connected
```

### Create redis cluster for redis master
```
[node 1]
redis-trib.rb create 192.168.10.143:6379 192.168.10.147:6379 192.168.10.148:6379
```

###### Result
```
>>> Creating cluster
>>> Performing hash slots allocation on 3 nodes...
Using 3 masters:
192.168.10.148:6379
192.168.10.147:6379
192.168.10.143:6379
M: 9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 192.168.10.143:6379
   slots:10923-16383 (5461 slots) master
M: 75c61d39cfaf09a094599c413d38c26147179681 192.168.10.147:6379
   slots:5461-10922 (5462 slots) master
M: f19d87e82bdba43941e4d6b95f1465537c46c120 192.168.10.148:6379
   slots:0-5460 (5461 slots) master
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join..
>>> Performing Cluster Check (using node 192.168.10.143:6379)
M: 9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 192.168.10.143:6379
   slots:10923-16383 (5461 slots) master
M: 75c61d39cfaf09a094599c413d38c26147179681 192.168.10.147:6379
   slots:5461-10922 (5462 slots) master
M: f19d87e82bdba43941e4d6b95f1465537c46c120 192.168.10.148:6379
   slots:0-5460 (5461 slots) master
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```

#### Verify redis cluster **[all nodes]**
```
redis-cli cluster nodes
```
###### Result
```
f19d87e82bdba43941e4d6b95f1465537c46c120 192.168.10.148:6379 master - 0 1469002082805 3 connected 0-5460
9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 192.168.10.143:6379 myself,master - 0 0 1 connected 10923-16383
75c61d39cfaf09a094599c413d38c26147179681 192.168.10.147:6379 master - 0 1469002084811 2 connected 5461-10922
```

### Start redis-slave **[all nodes]**
```
service redis-slave start
```
#### Verify redis-slave is running **[all nodes]**
```
redis-cli -p 6380 cluster nodes
```
###### Result
```
cce10c807af643a522e6aeccb309b164b556a835 :6380 myself,master - 0 0 0 connected
```
### Add slave node to cluster
```
[node 1]
redis-cli CLUSTER MEET 192.168.10.143 6380
redis-cli CLUSTER MEET 192.168.10.147 6380
redis-cli CLUSTER MEET 192.168.10.148 6380
Result = OK
```
#### Verify redis cluster **[all nodes]**
```
redis-cli cluster nodes
```
###### Result
```
f19d87e82bdba43941e4d6b95f1465537c46c120 192.168.10.148:6379 master - 0 1469002991329 3 connected 0-5460
a26cb609d42adf75536f9a9f0f52604f3461bb96 192.168.10.148:6380 master - 0 1469002991830 5 connected
cce10c807af643a522e6aeccb309b164b556a835 192.168.10.143:6380 master - 0 1469002991329 4 connected
75c61d39cfaf09a094599c413d38c26147179681 192.168.10.147:6379 master - 0 1469002990828 2 connected 5461-10922
dcae654ae883d11bf89427d6853371bed95224ab 192.168.10.147:6380 master - 0 1469002990327 0 connected
9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 192.168.10.143:6379 myself,master - 0 0 1 connected 10923-1638
```

### Set redis-slave for replicate master
**redis-cli -p 6380 CLUSTER REPLICATE (specified node id is a master)**
```
[node 1]
redis-cli -p 6380 CLUSTER REPLICATE 9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85
Result = OK
[node 2]
redis-cli -p 6380 CLUSTER REPLICATE 75c61d39cfaf09a094599c413d38c26147179681
Result = OK
[node 3]
redis-cli -p 6380 CLUSTER REPLICATE f19d87e82bdba43941e4d6b95f1465537c46c120
Result = OK
```
#### Verify redis cluster **[all nodes]**
```
redis-cli cluster nodes
```
###### Result
```
f19d87e82bdba43941e4d6b95f1465537c46c120 192.168.10.148:6379 master - 0 1469003290464 3 connected 0-5460
a26cb609d42adf75536f9a9f0f52604f3461bb96 192.168.10.148:6380 slave f19d87e82bdba43941e4d6b95f1465537c46c120 0 1469003291466 5 connected
cce10c807af643a522e6aeccb309b164b556a835 192.168.10.143:6380 slave 9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 0 1469003289962 4 connected
75c61d39cfaf09a094599c413d38c26147179681 192.168.10.147:6379 master - 0 1469003289460 2 connected 5461-10922
dcae654ae883d11bf89427d6853371bed95224ab 192.168.10.147:6380 slave 75c61d39cfaf09a094599c413d38c26147179681 0 1469003290965 2 connected
9ac2d6bbecdc8e2a93dfb016447c1ec0afd79e85 192.168.10.143:6379 myself,master - 0 0 1 connected 10923-16383
```




