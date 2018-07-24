## openvpn的安装 

【openvpn介绍】

```
虚拟私有网络VPN隧道是通过Internet隧道技术将两个不同地理位置的网络安全的连接起来的技术。当两个网络是使用私有IP地址的私有局域网络时，它们之间是不能相互访问的，这时使用隧道技术就可以使得两个子网内的主机进行通讯
```
【应用场景】
```
公司的内网服务器安装有ci.enai.com等服务，正常情况只有在公司内网才可以访问，如果想在外网也能访问到公司内网的服务，需要安装shadowsocks或者openvpn
```

【服务器端安装过程】   
```shell
服务器端是公司的172.16.1.10黑苹果物理服务器

1.创建需要挂载的目录
mkdir /mnt/disk1/docker-volumes/openvpn
export  ${OVPN_DATA}=/mnt/disk1/docker-volumes/openvpn
docker pull kylemanna/openvpn
2.建立easyrsa PKI证书存储,注意office.xiaoenai.net是公司内网的路由设备外网域名，
  记得要在路由设备上做端口转发
docker run -v ${OVPN_DATA}:/etc/openvpn --rm kylemanna/openvpn ovpn_genconfig -u tcp://office.xiaoenai.net 
3.生成EasyRSA PKI 证书授权中心，要求你输入CA私有密钥的密码，输入密码为123456, Common Name为xiaoenai
docker run -v ${OVPN_DATA}:/etc/openvpn --rm -it kylemanna/openvpn ovpn_initpki
4.生成客户端证书和配置文件，会提示你生成客户端登录密码和输入ca私有秘钥的密码， 输入密码为123456
docker run -v ${OVPN_DATA}:/etc/openvpn --rm -it kylemanna/openvpn easyrsa build-client-full xiaoenai  
5.通过容器内部脚本生成配置文件
docker run -v ${OVPN_DATA}:/etc/openvpn --rm kylemanna/openvpn ovpn_getclient xiaoenai > ${OVPN_DATA}/xiaoenai.ovpn
5.最后生产的文件如下所示，这些文件的权限和属主最好不要去修改
ll /mnt/disk1/docker-volumes/openvpn
drwxr-xr-x 2 root     root     4.0K 7月  13 10:57 ccd
-rw-r--r-- 1 root     root      625 7月  13 11:02 crl.pem
-rw-r--r-- 1 xiaoenai xiaoenai  657 7月  13 11:38 openvpn.conf
-rw-r--r-- 1 root     root      818 7月  13 10:57 ovpn_env.sh
drwx------ 6 root     root     4.0K 7月  13 11:00 pki
-rw-rw-r-- 1 xiaoenai xiaoenai 4.9K 7月  13 11:01 xiaoenai.ovpn
6.修改openvpn.conf配置文件，增加dns解析，否则后续只能通过ip地址访问
vi /mnt/disk1/docker-volumes/openvpn/openvpn.conf
### Push Configurations Below
push "block-outside-dns"
push "dhcp-option DNS 172.16.1.1"   这是公司内网路由设备的内网地址
7.创建docker-compose.yml,便于后续管理
cd /mnt/disk1/docker-compose/docker-openvpn/
vi docker-compose.yml
openvpn:
    image: kylemanna/openvpn:latest
    container_name: openvpn
    user: root
    privileged: true
    restart: always
    ports:
      - "1194:1194/tcp"
    volumes:
      - /mnt/disk1/docker-volumes/openvpn/:/etc/openvpn
    dns: 172.16.1.1
    mem_limit: 32m
    memswap_limit: 32m
docker-compose up -d 
8.日志查看，日志的最后一行提示初始化完成，说明服务端配置成功
docker logs openvpn
......
Thu Jul 12 06:54:13 2018 Initialization Sequence Completed
9.如上步骤做完后，会有如下文件生成，把如下文件复制到客户端，等下客户端会用到这些文件
/mnt/disk1/docker-volumes/openvpn/xiaoenai.ovpn
/mnt/disk1/docker-volumes/openvpn/pki/ca.crt
/mnt/disk1/docker-volumes/openvpn/pki/issued/xiaoenai.crt
/mnt/disk1/docker-volumes/openvpn/pki/private/xiaoenai.key
```
【mac客户端配置】
 下载mac的openvpn客户端，下载地址如下
 http://code.google.com/p/tunnelblick/wiki/DownloadsEntry?tm=2
 客户端的安装配置参照如下文章
 https://www.jianshu.com/p/a5fd8dc95ad4

【mac客户端测试】
```
如果浏览器能正常访问ci.enai.com或者mp.wucaiapp.com，说明openvpn配置成功
```




