# 离线部署ceph


## 一.拷贝安装包并配置本地源
1、将安装包解压
 将package.tar复制到 /opt 目录下，解压
  
     $ sudo tar -xvf  package.tar
  
2、将安装包所在和源路径添加到系统源source.list
     
     $ sudo vi /etc/apt/sources.list
        deb [ trusted=yes] file:/opt  packs/

   然后将所有的其他deb全部注销掉（#）


3.更新您的存储库并安装ceph-deploy：:
    
     sudo  apt update
     sudo  apt-get install ceph-deploy
     
 
## 二.安装ntp服务器
 
1. 每个节点上都安装NTP，确保每个Ceph节点时间同步： 
          
          `apt install ntp`
          
2.服务端（主节点）配置（192.168.62.131）：
        
                【命令】vim /etc/ntp.conf 
                【内容】
                       #linux自带的时间同步，需要注释掉
                       #pool 0.ubuntu.pool.ntp.org iburst
                       #pool 1.ubuntu.pool.ntp.org iburst
                       #pool 2.ubuntu.pool.ntp.org iburst
                       #pool 3.ubuntu.pool.ntp.org iburst
                       #pool ntp.ubuntu.com
					   添加：
                       restrict 192.168.62.2 mask 255.255.255.0 nomodify notrap     //集群所在网段的网关（Gateway），子网掩码（Genmask）     
                       server 127.127.1.0      
                       fudge 127.127.1.0 stratum 10
                       
                       


       
3.客户端（子节点）配置 (例如：192.168.62.132)： 
       
                 【命令】 vim /etc/ntp.conf 
                 【内容】 
                       #linux自带的时间同步，需要注释掉
                       #pool 0.ubuntu.pool.ntp.org iburst
                       #pool 1.ubuntu.pool.ntp.org iburst
                       #pool 2.ubuntu.pool.ntp.org iburst
                       #pool 3.ubuntu.pool.ntp.org iburst
                       #pool ntp.ubuntu.com
                     # 添加如下语句
                      server 192.168.62.131
                      fudge 192.168.62.131 stratum 10
                      
   # systemctl restart ntp
     
## 三. 每个节点创建安装用户

   1.  分别修改每一个服务器名称
      
           # hostnamectl set-hostname node1 
           # hostnamectl set-hostname node2
           # hostnamectl set-hostname node3
   
   2.  在每个Ceph节点上创建一个新用户: ` sudo useradd -d /home/{username} -m {username} ` 把{username}改为自己需要的用户名,下面类似
    *   设置密码` sudo passwd {username} `
    *   对于添加到每个Ceph节点的新用户，请确保该用户具有 sudo权限。
``` 
          echo "{username} ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/{username}
          sudo chmod 0440 /etc/sudoers.d/{username} 
```
    * 生成ssh密钥，将密码保留为空：` ssh-keygen`  
       （注：ceph-deploy不会提示输入密码，因此必须在admin节点上生成SSH密钥，并将公钥分发给每个Ceph节点）
 
  3. 进行IP地址映射,修改文件/etc/hosts,在里面加入:
    ```
    192.168.218.134 node1
    192.168.218.135 node2
    192.168.218.136 node3

    ```

    *  将密钥复制到每个Ceph节点：
   
           ssh-copy-id {username}@node1
           ssh-copy-id {username}@node2
           ssh-copy-id {username}@node3

  4.修改管理节点的~/.ssh/config文件，ceph-deploy会用创建的用户登录ceph节点，而不用每次都指定(根据自己的节点更换相应的信息)
    
            Host node1
               Hostname node1
               User user1
            Host node2
               Hostname node2
               User user2
            Host node3
               Hostname node3
               User user3
    5、关掉防火墙  systemctl stop  ufw 
	               sudo ufw disable 
	  
##  四. 使用Ceph-deploy安装Ceph


1. 如果遇到问题需要重新开始执行以下命令清除ceph软件包、数据与配置
    * ` ceph-deploy purge {ceph-node} [{ceph-node}]` 
    * ` ceph-deploy purgedata {ceph-node} [{ceph-node}]` 
    * ` ceph-deploy forgetkeys` 
    * ` rm ceph.*` 
2. 创建ceph集群
     * ` ceph-deploy new node1 node2 node3 `  （*注：需要把各ceph节点的hostname改为对应的名字，如 ` hostname node1` ,集群中部署三个）
	     
		 apt install python
     * ` ceph-deploy install --no-adjust-repos node1 node2 node3 --release nautilus`  （指定ceph的版本，这条命令会在各节点上安装ceph）(加入--no-adjust-repos 不修改源；若某个节点安装失败可采用apt install ceph在对应节点安装，部署所有机器)
         
      *vim ceph.conf
	   mon 只保留第一个ip
       public_network = */24  (* 为本机所在全网段业务ip)         --> 可以用echo 'public_network = */25' >> ceph.conf添加到文件尾
	   cluster_network =  */24 (* 为本机所在全网段存储ip)
     * 部署初始监视器并收集密钥：` ceph-deploy mon create-initial`  (首先确定防火墙已关闭)
     * 使用ceph-deploy配置文件和管理密钥复制到ceph各节点：` ceph-deploy admin node1 node2 node3` （所有机器）
     * 部署管理器守护程序：` ceph-deploy mgr create node1  node2 node3` （前三个机器） 
     * 添加osd，最好保证每个节点都有一个未使用过的磁盘  如： （若删除掉 #vgs    #vgremove ）   
         ```
                
              ceph-deploy osd create --data /dev/sdb node2
              ceph-deploy osd create --data /dev/sdb node3											--> 这一步有可能报错“GPT headers found”
         ```																						--> ceph-deploy disk zap node2 /dev/sdb删除磁盘分区信息

3. 查看完整的集群状态： ` ssh node1 sudo ceph -s` 