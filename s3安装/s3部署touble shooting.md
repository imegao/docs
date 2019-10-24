# s3部署Trouble-shooting
## 部署TiDB
1. ...could not find the requested service tidb-4000...
可能是tidb服务部署未成功，重新执行
```
$ansible -playbook deploy.yml -t tidb
```
2. swap is on，for best performance，turn swap off error
请关闭分区
```
swaoff -a
```
再次启动集群:
```
$ansible-playbook start.yml
```
3. VersionConflict: (cryptography 1.7.2 (/usr/lib64/python2.7/site-packages), Requirement.parse('cryptography>=2.5'))
   更新
```
$pip pip install --upgrate
```
安装wheel
下载：
```
$wget https://files.pythonhosted.org/packages/2a/fb/aefe5d5dbc3f4fe1e815bcdb05cbaab19744d201bbc9b59cfa06ec7fc789/wheel-0.31.1.tar.gz
```
解压：
```
$tar -zxvf wheel-0.31.1.tar.gz
```
进入目录：
```
$cd wheel-0.31.1/
```
安装：
```
$python setup.py install
```
下载：
```
$wget https://files.pythonhosted.org/packages/87/e6/915a482dbfef98bbdce6be1e31825f591fc67038d4ee09864c1d2c3db371/cryptography-2.3.1-cp27-cp27mu-manylinux1_x86_64.whl
```
安装：
```
$pip install cryptography-2.3.1-cp27-cp27mu-manylinux1_x86_64.whl
```
cryptography的方法二： 
卸载crypto包
```
#rpm -qa |grep python-crypto
#yum -y remove python-cryptography
```
安装crypto包
```
# yum -y install ansible
```
4. Ansible FAILED! => playbook: bootstrap.yml error :This machine does not have sufficient CPU to run TiDB, at least 8 cores
```
$vi bootstrap.yml
```
注释掉以下内容(#)：
```
- name: check system
  hosts: all
  any_errors_fatal: true
  roles:
    - check_system_necessary
#    - { role: check_system_optional, when: not dev_mode }


如果是非SSD 测试的话 ，最好将如下的内容注释掉 
- name: tikv_servers machine benchmark
  hosts: tikv_servers
  gather_facts: false
  roles:
#    - { role: machine_benchmark, when: not dev_mode }
```
5. playbook: deploy.yml error: check the ntp service start
确保ntp服务开启
```
#systemctl start ntpd
```
## 部署ceph
1. try other mirror failed ..
可以尝试配置一个国内的yum源，例如阿里云。参考：https://www.cnblogs.com/jimboi/p/8437788.html
2. ceph-deploy create node error：ensuring /etc/yum.repo/ceph.repo cotains a high priority error..
执行
```
yum install centos-release-ceph-luminous.noarch
```
3. ceph-deploy install node1..error 
将执行失败的语句单独在节点上执行看是否成功，成功则继续
安装ceph失败可以单独在此节点执行
```
yum -y install ceph
```
4. 执行ceph-deploy mon create-initial 时，产生 ... node1 is still no in the quram..某节点还未在监控队列中
可关闭此节点的firewall
5. ceph -s 集群状态health-warn,'there is too few pg'
重新设置pool的pg值一定参考pg的计算方法。
```
ceph osd pool set volumes pg_num 512
```
6. 执行命令：#ceph-deploy install node1 node2 node3
错误：ERROR：[ceph_deploy][ERROR ] RuntimeError: Failed to execute command: rpm -Uvh --replacepkgs http://ceph.com/rpm-giant/el6/noarch/ceph-release-1-0.el6.noarch.rpm
解决：更改/etc/yum.repos.d/ceph.repo,全部替换为：
```
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
gpgkey=https://download.ceph.com/keys/release.asc
```
这里参考:https://www.cnblogs.com/freedom314/p/8596031.html#

## 部署Caddy
如果产生go get安装一些第三方库的网络问题"golang.org/x/sys/unix"请手动下载相应的package
```
    $mkdir -p $GOPATH/src/golang.org/x/
    $cd !$
    $git clone https://github.com/golang/net.git
    $git clone https://github.com/golang/sys.git
    $git clone https://github.com/golang/tools.git
```
## 部署Yig
1. 如果出现版本不兼容问题，安装docker容器
```
    $yum insall docker
```
然后在yig目录下执行
```
    $make build 
```
