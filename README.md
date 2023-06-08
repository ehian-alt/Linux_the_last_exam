# Linux_the_last_exam
Linux 期末步骤（x代表小学号）
## 准备工作(四台虚拟机，三台服务手动配置ip地址, dns服务器都是192.168.x.5，保证所有虚拟机网卡模式都是一致的，我这里是net模式)
* 第一台（主DNS/DHCP 服务器）：192.168.x.5/24
* 第二台（samba/nfs/辅助dns 服务器）：192.168.x.4/24
* 第三台（web/ftp 服务器）：192.168.x.10/24
* 第四台（client 客户测试端）：通过dhcp服务器提供的DHCP服务来获取IP  

**要在 `编辑 > 虚拟网络编辑器` 中将net模式中的，`使用本地DHCP服务将IP地址分配给虚拟机`关闭** 

![image](https://github.com/ehian-alt/Linux_the_last_exam/assets/79576798/fa68bba9-6381-4d5b-95ef-c1057f1b8ab9)

## 服务器配置步骤
### DHCP 
#### 在DHCP服务器
1. 挂载本地yum源(根据各自yum配置文件挂载)
2. 安装dhcp包，`yum install dhcp -y`
3. 确认ip地址无误后，vim 编辑配置文件`vim /etc/dhcp/dhcpd.conf`
4. 把文件中有一段 `see /usr/share/doc/dhcp*/dhcpd.conf.example` ,把后面的路径复制下来  
在命令模式输入命令: `:r /usr/share/doc/dhcp*/dhcpd.conf.example`
6. 可以直接把没用的信息删掉，只留下需要的，比如下面的，x代表小学号
```
subnet 192.168.x.0 netmask 255.255.255.0 {
  range 192.168.x.12 192.168.x.200;
  option domain-name-servers 192.168.x.5;
  option domain-name "dns.jnet9.com";
  option routers 192.168.43.254;
  option broadcast-address 192.168.x.255;
  default-lease-time 3600;
  max-lease-time 7200;
}
```
6. 保存退出，`systemctl start dhcpd` 启动服务
#### 客户端测试
***客户端测试：*** 将网卡设为通过DHCP服务获取IP地址，将网卡关闭再打开，查看IP是否正确，正常情况下是(192.168.x.12)，到这DHCP要求就完成了

### DNS
#### 主DNS服务器
1. 通过yum源安装bind: `yum install bind -y`
2. 修改两个any, `vim /etc/named.conf`
3. `vim /etc/named.rfc1912.zones`， 在文件最后添加正向反向信息
```
zone "jnet9.com" IN {
        type master;
        file "data/master.jnet9.com.zone";
};

zone "x.168.192.in-addr.arpa" IN {
        type master;
        file "data/master.x.168.192.in-addr.arpa.zone";
};
```
4. `vim /var/named/data/master.jnet9.com.zone`添加正向资源记录
```
$TTL    1D
@       IN      SOA     jnet9.com       root.jnet9.com. (
                                0
                                1D
                                1H
                                1W
                                3H )
@       IN      NS      dns.jnet9.com.com.
dns     IN      A       192.168.x.5
dhcp    IN      A       192.168.x.5
samba   IN      A       192.168.x.4
nfs     IN      A       192.168.x.4
dns1    IN      A       192.168.x.4
www     IN      A       192.168.x.10
ftp     IN      A       192.168.x.10
client  IN      A       192.168.x.12
```
5. `vim /var/named/data/master.x.168.192.in-addr.arpa.zone` 添加反向资源记录
```
$TTL    1D
@       IN      SOA     x.168.192.in-addr.arpa. root.jnet9.com. (
                                0       ;serial
                                1D      ;refresh
                                1H      ;retry
                                1W      ;expire
                                3H )    ;minimum
@       IN      NS      dns.jnet9.com.
5       IN      PTR     dns.jnet9.com.
5       IN      PTR     dhcp.jnet9.com.
4       IN      PTR     samba.jnet9.com.
4       IN      PTR     nfs.jnet9.com.
4       IN      PTR     dns1.jnet9.com.
10      IN      PTR     www.jnet9.com.
10      IN      PTR     ftp.jnet9.com.
12      IN      PTR     client.jnet9.com.
```
6. `systemctl start named`启动dns服务
7. `iptables -F` 关闭防火墙 `setenforce 0` 关闭SELinux
#### 辅助DNS服务器
1. 安装bind `yum install bind -y`
2. 修改两个`any`, `vim /etc/named.conf`
3. `vim /etc/named.rfc1912.zones`， 在文件最后添加正向反向信息
```
zone "jnet9.com" IN {
        type slave;
        file "slave/slave.jnet9.com.zone";
        masters { 192.168.x.5; };
};

zone "x.168.192.in-addr.arpa" IN {
        type slave;
        file "slave/slave.x.168.192.in-addr.arpa.zone";
        masters { 192.168.x.5; };
};
```
4. `systemctl start named` 启动DNS服务
5. `iptables -F` 关闭防火墙 `setenforce 0` 关闭SELinux

#### 客户端测试
***客户端测试：*** `nslookup 192.168.x.4` , `nslookup 192.168.x.5` , `nslookup 192.168.x.10` ,   
`nslookup dns.jnet9.com` , `nslookup samba.jnet9.com` ······ 随便测几个就行

### Samba 
#### 在Samba服务器
1. 通过yum源安装(客户端也要安装)：`yum install samba -y`, `yum install samba-client -y`, `yum install cifs-utils -y` 
2. 创建共享文件夹和用户
```
[root@RHEL7-1 ~]# mkdir /boss
[root@RHEL7-1 ~]# mkdir /tech
[root@RHEL7-1 ~]# useradd boss
[root@RHEL7-1 ~]# groupadd tech
[root@RHEL7-1 ~]# useradd -G tech tech_user
```
3. 添加samba账号
```
[root@RHEL7-1 ~]# smbpasswd -a boss
New SMB password:
Retype new SMB password:
Added user boss.
[root@RHEL7-1 ~]# smbpasswd -a tech_user
New SMB password:
Retype new SMB password:
Added user tech_user.
```
4. 设置文件所属组用户，和文件访问权限
```
[root@RHEL7-1 ~]# chown boss /boss
[root@RHEL7-1 ~]# chown tech_user:tech /tech/
[root@RHEL7-1 ~]# setfacl -m u:boss:rwx /boss/
[root@RHEL7-1 ~]# setfacl -m o::--- /boss/
[root@RHEL7-1 ~]# setfacl -m g:tech:rwx /tech/
[root@RHEL7-1 ~]# setfacl -m u:boss:r-x /tech/
```
5. 修改配置文件`vim /etc/samba/smb.conf` 在文件末添加以下信息:
```
[boss]
        comment = boss
        path = /boss
        public = no
        writable = yes 
[tech]
        comment = tech
        path = /tech
        public = no
        writable = yes 
```
6. 启动服务关闭防火墙和SELinux
```
[root@RHEL7-1 ~]# systemctl start smb
[root@RHEL7-1 ~]# systemctl start nmb
[root@RHEL7-1 ~]# iptables -F
[root@RHEL7-1 ~]# setenforce 0
```

***客户端测试：*** 首先确认这三个安装好了：`yum install samba -y`, `yum install samba-client -y`, `yum install cifs-utils -y`  
然后smbclient 测试  `smbclient //192.168.x.4/boss -U boss` 输入刚刚设置的密码，分别用两个用户去测试文件权限

### NFS
#### 在NFS服务器
1. 安装`yum install nfs-utils -y` , `yum install rpcbind -y` （客户端也要安装）
2. 启动服务， 关闭防火墙和SELinux(刚才关了这里就可以不用关了)
```
[root@smb ~]# systemctl start nfs
[root@smb ~]# systemctl start rpcbind
[root@smb ~]# systemctl start nfs-server
[root@smb ~]# systemctl enable nfs-server
Created symlink from /etc/systemd/system/multi-user.target.wants/nfs-server.service to /usr/lib/systemd/system/nfs-server.service.
[root@smb ~]# systemctl enable rpcbind
[root@smb ~]# iptables -F
[root@smb ~]# setenforce 0
```
3. 创建共享文件夹 `mkdir /dir`, 并创建一个文件作为测试 `echo hhhhhh > /dir/test`
4. 编辑文件 `vim /etc/exports`，写入以下内容
```
/dir  *(rw,anonuid=1001,anongid=1001)
```
#### 客户端测试
***客户端测试：***   创建挂载目录，`mkdir /nfstest`
`showmount -e 192.168.x.4`
```
[root@smb ~]# showmount -e 192.168.43.4
Export list for 192.168.43.4:
/dir *
[root@smb ~]# 
```
像这样显示一般就没问题。

客户端nfs挂载: `mount -t nfs 192.168.x.4:/dir /nfstest`
`ls /nfstest` 查看文件列表是否有nfs服务器刚刚创建的test文件，有就说明成功了，可以用`cat /nfstest/test`，确认内容是`hhhhhh`

### HTTP
#### web服务器
1. yum 安装 Apache 服务 `yum install httpd -y`
2. 启动HTTP，关闭防火墙SELinux
```
[root@RHEL7-2 ~]# systemctl start httpd
[root@RHEL7-2 ~]# systemctl enable httpd
Created symlink from /etc/systemd/system/multi-user.target.wants/httpd.service to /usr/lib/systemd/system/httpd.service.
[root@RHEL7-2 ~]# iptables -F
[root@RHEL7-2 ~]# setenforce 0
``` 
