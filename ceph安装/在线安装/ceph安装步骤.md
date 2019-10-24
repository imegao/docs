
#### Ceph部署过程（操作系统centos7.5）
##### 一. 安装Ceph-deploy

1. 配置下载源:
   
    - sudo yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm							-->可以和李韶雄商量下，是否可以改成10.0.53.130这里的镜像了
    -  
```
vi /etc/yum.repos.d/ceph.repo

内容：（配置本地源、此处为nautilus版）
[Ceph]
name=Ceph packages for $basearch
baseurl=http://10.0.47.183/ceph/rpm-nautilus/el7/$basearch
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc
priority=1

[ceph-noarch]
name=Ceph noarch packages
baseurl=http://10.0.47.183/ceph/rpm-nautilus/el7/noarch       
enabled=1
gpgcheck=1
type=rpm-md
gpgkey=https://download.ceph.com/keys/release.asc																			-->可以和李韶雄商量下，是否可以改成10.0.53.130这里的镜像了


安装最新Ceph-deploy：`sudo yum install ceph-deploy`   
   
   
   
   2. 每个节点上都安装NTP，确保每个Ceph节点时间同步： ` sudo yum install ntp ntpdate ntp-doc` 
    
    服务端（主节点）配置（192.168.62.131）：
        
                【命令】vim /etc/ntp.conf 
                【内容】
                       driftfile /var/lib/ntp/drift
                       restrict default nomodify notrap nopeer noquery
                       restrict 192.168.62.131 nomodify notrap nopeer noquery     //当前节点IP地址
                       restrict 127.0.0.1
                       restrict ::1

                       restrict 192.168.62.2 mask 255.255.255.0 nomodify notrap     //集群所在网段的网关（Gateway），子网掩码（Genmask） 
                       # 在server部分添加以下部分，（若为离线应用 请注释掉server 0 ~ n）        
                       server 127.127.1.0      
                       fudge 127.127.1.0 stratum 10
                       
                       includefile /etc/ntp/crypto/pw
                       keys /etc/ntp/keys


       
   客户端（子节点）配置 (例如：192.168.62.132)： 
       
                 【命令】 vim /etc/ntp.conf 
                 【内容】 
                       driftfile /var/lib/ntp/drift
                       restrict default nomodify notrap nopeer noquery
                       restrict 192.168.62.132 nomodify notrap nopeer noquery     //当前节点IP地址
                       restrict 127.0.0.1
                       restrict ::1
                      
                      restrict 192.168.62.2 mask 255.255.255.0 nomodify notrap      //集群所在网段的网关（Gateway），子网掩码（Genmask） 
                     # 在server部分添加如下语句，并注释掉server 0 ~ n  将server指向主节点
                      server 192.168.62.131
                      fudge 192.168.62.131 stratum 10
                      
                      includefile /etc/ntp/crypto/pw
                      keys /etc/ntp/keys
   
   
3. 每个节点创建安装用户，
    *   修改每一个服务器名称:`vi /etc/hostname`修改后重启
    *   在每个Ceph节点上创建一个新用户: ` sudo useradd -d /home/{username} -m {username} ` 把{username}改为自己需要的用户名,下面类似
    *   设置密码` sudo passwd {username} `
    *   对于添加到每个Ceph节点的新用户，请确保该用户具有 sudo权限。
``` 
          echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
          sudo chmod 0440 /etc/sudoers.d/{username} 
```
    * 生成ssh密钥，将密码保留为空：` ssh-keygen`  
（注：ceph-deploy不会提示输入密码，因此必须在admin节点上生成SSH密钥，并将公钥分发给每个Ceph节点）
    *  进行IP地址映射,修改文件/etc/hosts,在里面加入:
    ```
    192.168.218.134 node1
    192.168.218.135 node2
    192.168.218.136 node3

    ```

    *  将密钥复制到每个Ceph节点：
   
           ssh-copy-id {username}@node1
           ssh-copy-id {username}@node2
           ssh-copy-id {username}@node3

*  修改管理节点的~/.ssh/config文件，ceph-deploy会用创建的用户登录ceph节点，而不用每次都指定(根据自己的节点更换相应的信息)
    
            Host node1
               Hostname node1
               User usernode1
            Host node2
               Hostname node2
               User usernode2
            Host node3
               Hostname node3
               User usernode3
    
      
4. 打开必须的端口：ceph-mon默认以6789通信  osd端口范围在6800:7300（或直接关闭防火墙 systemctl stop firewalld）
    *   在监控节点：` sudo firewall-cmd --zone=public --add-service=ceph-mon --permanent`  
    *    在osd上：` sudo firewall-cmd --zone=public --add-service=ceph --permanent` 
（*注：使用--permanent配置firewalld后，可以立即生效，不用重启：` sudo firewall-cmd --reload` ）

    * 为iptables添加ceph-mon端口 : ` sudo iptables -A INPUT -i ens33 -p tcp -s 192.168.218.134/255.255.255.0 --dport 6789 -j ACCEPT` 
（*注：iface 是服务器连接网线的网口，通过ethtool命令查看网口是否连接 ip-address 是服务器网络地址，netmask是网络掩码）

    * 配置完成后时生效：` /sbin/service  iptable save` 
    
5. 把selinux暂时关掉：`  																													--> 暂时关掉是setenforce 0
																																			--> SELINUX=disabled是重启后永久生效，当前不生效
    ```
     vi /etc/sysconfig/selinux 
    
       SELINUX=disabled
    ```


##### 二. 使用Ceph-deploy安装Ceph

1. 如果遇到问题需要重新开始执行以下命令清除ceph软件包、数据与配置
    * ` ceph-deploy purge {ceph-node} [{ceph-node}]` 
    * ` ceph-deploy purgedata {ceph-node} [{ceph-node}]` 
    * ` ceph-deploy forgetkeys` 
    * ` rm ceph.*` 
2. 创建ceph集群
          install python-pkg-resources python-setuptools 
     * ` ceph-deploy new node1 node2 node3 `  （*注：需要把各ceph节点的hostname改为对应的名字，如 ` hostname node1`）
	 
     * ` ceph-deploy install --no-adjust-repos node1 node2 node3 --release nautilus`  （指定ceph的版本，这条命令会在各节点上安装ceph）(加入--no-adjust-repos 不修改源；若某个节点安装失败可采用yum install ceph在对应节点安装)
	     *vim ceph.conf 
          public_network = */25  (* 为本机ip)														--> 可以用echo 'public_network = */25' >> ceph.conf添加到文件尾
     * 部署初始监视器并收集密钥：` ceph-deploy mon create-initial`  (首先确定防火墙已关闭)
     * 使用ceph-deploy配置文件和管理密钥复制到ceph各节点：` ceph-deploy admin node1 node2 node3` 
     * 部署管理器守护程序：` ceph-deploy mgr create node1 ` 
     * 添加osd，最好保证每个节点都有一个未使用过的磁盘  如：    
         ```
                
              ceph-deploy osd create --data /dev/sdb node2
              ceph-deploy osd create --data /dev/sdb node3											--> 这一步有可能报错“GPT headers found”
         ```																						--> ceph-deploy disk zap node2 /dev/sdb删除磁盘分区信息

3. 查看完整的集群状态： ` ssh node1 sudo ceph -s` 



有疑问请查看ceph官网：[Ceph](http://docs.ceph.com/docs/master/start/quick-ceph-deploy/)



















  


































 







