内网网站搭建
##########################

IP配置
******************

首先给linux主机配置一个静态IP地址。打开端口扫描工具Advanced（也可以是其他工具）扫描局域网的IP，
地址为192.168.3.1-255.查找一个没有被使用的IP地址。


查看网卡设备号

配置静态IP，ip地址为192.168.3.128。网关为192.168.3.1 DNS：114.114.114.114。

安装nginx

安装教程：https://nginx.org/en/linux_packages.html#Ubuntu

设置nginx默认的位置

/etc/nginx/nginx.config

将html文件夹添加rx权限。


