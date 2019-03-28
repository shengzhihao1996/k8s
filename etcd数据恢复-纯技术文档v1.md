- 注释： 
- 1、 节点损毁包括：服务器宕机、工作目录下数据丢失误删除、服务报错无法启动等。
- 2、 一下提供三个场景，如遇到真实状况，请选取其中之一，详细阅读操作流程以及说明，领会精神奥义之后，再付诸行动。
- 3、 真实情况节点ip可能与本文不符，需要读者自行替换ip地址以模仿操作流程。

# etcd单节点数据恢复
### 场景： 52损毁，53、54正常提供服务
###### 一、删除etcd工作目录下所有内容，修改服务service文件。
~~~
#登录52机器
[root@kubernetes-1 ~]# ifconfig  eno1|awk 'NR==2'
inet 10.90.28.52  netmask 255.255.224.0  broadcast 10.90.31.255

#停etcd服务
[root@kubernetes-1 ~]# systemctl stop etcd

#删etcd工作目录
[root@kubernetes-1 ~]# rm -rf /var/lib/etcd/member/

#修改etcd服务的配置
[root@kubernetes-1 ~]# grep initial-cluster-state  /etc/systemd/system/etcd.service 
  --initial-cluster-state=existing \

#reload  etcd服务
[root@kubernetes-1 ~]# systemctl daemon-reload
~~~
###### 二、寻找一台完好的etcd机器，执行命令删除损毁etcd成员，将其重新加入集群，并依次重启剩余所有etcd机器上的etcd服务
~~~
#登录53机器
[root@kubernetes-2 ~]# ifconfig  eno1|awk 'NR==2'
inet 10.90.28.53  netmask 255.255.224.0  broadcast 10.90.31.255

#查看etcd集群成员及其状态
[root@kubernetes-2 ~]# ETCDCTL_API=3 etcdctl member list
7834e9f420d6092d, started, etcd2, https://10.90.28.53:2380, https://10.90.28.53:2379
828bdf2638bf93a9, started, etcd1, https://10.90.28.52:2380, https://10.90.28.52:2379
f96d4cc8ae8b232f, started, etcd3, https://10.90.28.54:2380, https://10.90.28.54:2379

#删除损毁成员
[root@kubernetes-2 ~]# ETCDCTL_API=3 etcdctl member remove 828bdf2638bf93a9
Member 828bdf2638bf93a9 removed from cluster ed185e5095b9713f

#重新添加成员
[root@kubernetes-2 ~]# ETCDCTL_API=3 etcdctl member add etcd1 --peer-urls=https://10.90.28.52:2380
Member 92acc59cc47078f0 added to cluster ed185e5095b9713f

ETCD_NAME="etcd1"
ETCD_INITIAL_CLUSTER="etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd3=https://10.90.28.54:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.90.28.52:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"

#重启etcd服务
[root@kubernetes-2 ~]# systemctl restart etcd

#登录54机器
[root@kubernetes-3 ~]# ifconfig  eno1|awk 'NR==2'
inet 10.90.28.54  netmask 255.255.224.0  broadcast 10.90.31.255

#重启etcd服务
[root@kubernetes-3 ~]# systemctl restart etcd
~~~
###### 三、返回损毁的etcd机器，启动etcd服务，验证集群状态
~~~
#登录52机器
[root@kubernetes-1 ~]# ifconfig  eno1|awk 'NR==2'
inet 10.90.28.52  netmask 255.255.224.0  broadcast 10.90.31.255

#重启etcd服务
[root@kubernetes-1 ~]# systemctl start etcd

#检测集群状态，如下，说明集群恢复成功
[root@kubernetes-1 ~]# etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  --cert-file=/etc/etcd/ssl/etcd.pem  cluster-health
member 7834e9f420d6092d is healthy: got healthy result from https://10.90.28.53:2379
member 92acc59cc47078f0 is healthy: got healthy result from https://10.90.28.52:2379
member f96d4cc8ae8b232f is healthy: got healthy result from https://10.90.28.54:2379
cluster is healthy
~~~
# etcd集群增加节点
### 场景： 52、53、54为正常提供服务的etcd集群，现需要补充一个etcd节点，节点ip为165
###### 一、 寻找一台完好的etcd机器，执行命令增加一个etcd成员，基于新生成的ETCD_INITIAL_CLUSTER的值替换所有etcd的service文件中的initial-cluster的value，并重启所有etcd服务。
~~~
#登录52机器
[root@kubernetes-1 ~]# ifconfig  eno1|awk 'NR==2'
        inet 10.90.28.52  netmask 255.255.224.0  broadcast 10.90.31.255

#添加etcd集群新成员
[root@kubernetes-1 ~]# ETCDCTL_API=3 etcdctl member add etcd4 --peer-urls=https://10.90.24.165:2380
Member  90e5c08346d6861 added to cluster ed185e5095b9713f

ETCD_NAME="etcd4"
ETCD_INITIAL_CLUSTER="etcd4=https://10.90.24.165:2380,etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd3=https://10.90.28.54:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="https://10.90.24.165:2380"
ETCD_INITIAL_CLUSTER_STATE="existing"

#查看etcd集群成员及其状态
[root@kubernetes-1 ~]# ETCDCTL_API=3 etcdctl member list
90e5c08346d6861, unstarted, , https://10.90.24.165:2380, 
7834e9f420d6092d, started, etcd2, https://10.90.28.53:2380, https://10.90.28.53:2379
92acc59cc47078f0, started, etcd1, https://10.90.28.52:2380, https://10.90.28.52:2379
f96d4cc8ae8b232f, started, etcd3, https://10.90.28.54:2380, https://10.90.28.54:2379

#修改etcd服务的配置
[root@kubernetes-1 ~]# grep initial-cluster=  /etc/systemd/system/etcd.service 
  --initial-cluster=etcd3=https://10.90.28.54:2380,etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd4=https://10.90.24.165:2380 \

#重启etcd服务
[root@kubernetes-1 ~]# systemctl daemon-reload  && systemctl restart etcd

#登录53机器
[root@kubernetes-2 ~]# ifconfig  eno1|awk 'NR==2'
        inet 10.90.28.53  netmask 255.255.224.0  broadcast 10.90.31.255

#修改etcd服务的配置
[root@kubernetes-2 ~]# grep initial-cluster=  /etc/systemd/system/etcd.service
  --initial-cluster=etcd3=https://10.90.28.54:2380,etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd4=https://10.90.24.165:2380 \

#重启etcd服务
[root@kubernetes-2 ~]# systemctl daemon-reload  && systemctl restart etcd

#登录54机器
[root@kubernetes-3 ~]# ifconfig  eno1|awk 'NR==2'
        inet 10.90.28.54  netmask 255.255.224.0  broadcast 10.90.31.255

#修改etcd服务的配置
[root@kubernetes-3 ~]# grep initial-cluster=  /etc/systemd/system/etcd.service
  --initial-cluster=etcd3=https://10.90.28.54:2380,etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd4=https://10.90.24.165:2380 \

#重启etcd服务
[root@kubernetes-3 ~]# systemctl daemon-reload  && systemctl restart etcd
~~~
###### 二、登录新的etcd机器，创建工作目录、秘钥目录，copy文件，创建证书，创建service文件，启动etcd服务，验证集群状态
~~~
#登录165机器
[root@test-k8s-node1 ~]# ifconfig  eth0|awk 'NR==2'
        inet 10.90.24.165  netmask 255.255.224.0  broadcast 10.90.31.255

#创建etcd秘钥目录
[root@test-k8s-node1 ~]# mkdir /etc/etcd/ssl -p

#创建etcd工作目录
[root@test-k8s-node1 ~]# mkdir /var/lib/etcd

#cd切换到etcd秘钥目录下
[root@test-k8s-node1 ~]# cd  /etc/etcd/ssl

从完好的etcd机器上copy文件
[root@test-k8s-node1 ssl]# scp 10.90.28.52:/opt/kube/bin/{etcd,cfssl,cfssljson} /usr/local/bin/
[root@test-k8s-node1 ssl]# scp 10.90.28.52:/etc/etcd/ssl/etcd-csr.json . 

从完好的kube-master机器上copy文件
[root@test-k8s-node1 ssl]# scp 10.90.28.52:/etc/kubernetes/ssl/{ca-config.json,ca.pem,ca-key.pem} .

#修改证书创建json
[root@test-k8s-node1 ssl]# cat  etcd-csr.json 
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "10.90.24.165"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

#执行命令创建证书
[root@test-k8s-node1 ssl]# cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
2019/03/27 18:02:38 [INFO] generate received request
2019/03/27 18:02:38 [INFO] received CSR
2019/03/27 18:02:38 [INFO] generating key: rsa-2048
2019/03/27 18:02:38 [INFO] encoded CSR
2019/03/27 18:02:38 [INFO] signed certificate with serial number 200711656905856481121944760467469431523520319454
2019/03/27 18:02:38 [WARNING] This certificate lacks a "hosts" field. This makes it unsuitable for
websites. For more information see the Baseline Requirements for the Issuance and Management
of Publicly-Trusted Certificates, v.1.1.6, from the CA/Browser Forum (https://cabforum.org);
specifically, section 10.2.3 ("Information Requirements").

#查看密钥目录下文件
[root@test-k8s-node1 ssl]# ll
total 28
-rw-r--r-- 1 root root  292 Mar 27 17:59 ca-config.json
-rw------- 1 root root 1679 Mar 27 17:59 ca-key.pem
-rw-r--r-- 1 root root 1346 Mar 27 17:59 ca.pem
-rw-r--r-- 1 root root 1041 Mar 27 18:02 etcd.csr
-rw-r--r-- 1 root root  252 Mar 27 18:02 etcd-csr.json
-rw------- 1 root root 1679 Mar 27 18:02 etcd-key.pem
-rw-r--r-- 1 root root 1407 Mar 27 18:02 etcd.pem

#创建etcd服务文件
[root@test-k8s-node1 ssl]# cat  /etc/systemd/system/etcd.service 
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd/
ExecStart=/usr/local/bin/etcd \
  --name=etcd4 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls=https://10.90.24.165:2380 \
  --listen-peer-urls=https://10.90.24.165:2380 \
  --listen-client-urls=https://10.90.24.165:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.90.24.165:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd3=https://10.90.28.54:2380,etcd2=https://10.90.28.53:2380,etcd1=https://10.90.28.52:2380,etcd4=https://10.90.24.165:2380 \
  --initial-cluster-state=existing \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target

#启动etcd服务
[root@test-k8s-node1 ssl]# systemctl daemon-reload && systemctl start etcd

#检测集群状态，如下，说明节点添加成功
[root@test-k8s-node1 ssl]# etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  --cert-file=/etc/etcd/ssl/etcd.pem  cluster-health

~~~
# etcd集群数据恢复
### 场景： 原集群全部损毁，有备份文件，需要在新集群实现备份恢复。
###### 一、拿到备份文件
~~~
#登录52节点
[root@kubernetes-1 clean]# ifconfig  eno1|awk 'NR==2'
        inet 10.90.28.52  netmask 255.255.224.0  broadcast 10.90.31.255
#备份完好的etcd集群数据，名称为：snapshot.db
[root@kubernetes-1 clean]# ETCDCTL_API=3 etcdctl snapshot save snapshot.db
Snapshot saved at snapshot.db
[root@kubernetes-1 clean]# ll
total 7216
-rw-r--r-- 1 root root 7385120 Mar 27 18:55 snapshot.db
~~~
###### 二、关停所有etcd服务，删除所有etcd工作目录下的文件，复制备份文件至每个节点，依次执行备份恢复命令，生成新的目录，将其下的member考到etcd工作目录下，依次重启etcd服务。
~~~
#切换到新的etcd集群
#查看集群状态
[root@etcd1 ~]#  etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  --cert-file=/etc/etcd/ssl/etcd.pem  cluster-health
member 5c636e94d29b6a17 is healthy: got healthy result from https://192.168.1.1:2379
member 861b0df73b6dfe36 is healthy: got healthy result from https://192.168.1.2:2379
member b49efd0caebd99b3 is healthy: got healthy result from https://192.168.1.3:2379
#检查集群内数据，为空
[root@etcd1 ~]# ETCDCTL_API=3 etcdctl get / --prefix --keys-only
#批量停止etcd服务，批量删除etcd工作目录
[root@etcd1 ~]# for i in 192.168.1.1 192.168.1.2 192.168.1.3;do  ssh $i "systemctl stop etcd" ;done 
[root@etcd1 ~]# for i in 192.168.1.1 192.168.1.2 192.168.1.3;do  ssh $i "rm -rf /var/lib/etcd/member" ;done 

#在每个节点执行以下命令，initial-advertise-peer-urls的value为当前节点ip
ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
--name etcd1 \
--initial-cluster=etcd3=https://192.168.1.3:2380,etcd2=https://192.168.1.2:2380,etcd1=https://192.168.1.1:2380 \
--initial-cluster-token etcd-cluster-0 \
--initial-advertise-peer-urls https://192.168.1.1:2380
[root@etcd1 ~]# mv etcd1.etcd/member/ /var/lib/etcd/

ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
--name etcd2 \
--initial-cluster=etcd3=https://192.168.1.3:2380,etcd2=https://192.168.1.2:2380,etcd1=https://192.168.1.1:2380 \
--initial-cluster-token etcd-cluster-0 \
--initial-advertise-peer-urls https://192.168.1.2:2380
[root@etcd2 ~]# mv etcd2.etcd/member/ /var/lib/etcd/

ETCDCTL_API=3 etcdctl snapshot restore snapshot.db \
--name etcd3 \
--initial-cluster=etcd3=https://192.168.1.3:2380,etcd2=https://192.168.1.2:2380,etcd1=https://192.168.1.1:2380 \
--initial-cluster-token etcd-cluster-0 \
--initial-advertise-peer-urls https://192.168.1.3:2380
[root@etcd3 ~]# mv etcd3.etcd/member/ /var/lib/etcd/


#三个节点同时启动etd
systemctl start etcd

#查看etcd集群状态
[root@etcd1 ~]#  etcdctl --ca-file=/etc/kubernetes/ssl/ca.pem  --key-file=/etc/etcd/ssl/etcd-key.pem  --cert-file=/etc/etcd/ssl/etcd.pem  cluster-health
member 5c636e94d29b6a17 is healthy: got healthy result from https://192.168.1.1:2379
member 861b0df73b6dfe36 is healthy: got healthy result from https://192.168.1.2:2379
member b49efd0caebd99b3 is healthy: got healthy result from https://192.168.1.3:2379
cluster is healthy

#查看etcd集群数据，看到刷屏的效果表示数据成功导入，对接apiserver用kubectl查看集群详细数据
[root@etcd1 ~]# ETCDCTL_API=3 etcdctl get / --prefix --keys-only

~~~
