# Nacos持久化配置和集群搭建

### 环境准备



| 服务器名 | IP | 说明 |
| --- | --- | --- |
| MySQL | 192.168.223.135 | 部署MySQL数据库和Nginx |
| Nacos | 192.168.223.137 | 部署Nacos集群 |

资源有限，MySQL 部署了一台机器，Nginx 和 Nacos 集群部署在了另一台机器。如果在生产环境部署，可以按照自己的需求调整。


### 配置步骤


> 下载地址：[https://github.com/alibaba/nacos/releases/download/1.3.0/nacos-server-1.3.0.tar.gz](https://github.com/alibaba/nacos/releases/download/1.3.0/nacos-server-1.3.0.tar.gz)



将压缩包拷贝到对应部署 Nacos 的机器上


1. MySQL 数据库配置



[MySQL安装教程](https://cxhello.github.io/docs/#/article/%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85/MySQL%E5%AE%89%E8%A3%85%E4%B8%8E%E9%85%8D%E7%BD%AE)


安装好 MySQL 以后，需要初始化 MySQL 数据库，数据库初始化文件在压缩包 conf 文件下的 nacos-mysql.sql，在对应的数据库环境下导入 SQL 文件
```bash
# 进入MySQL终端
mysql -u root -p123456
mysql> create database nacos_config;
mysql> use nacos_config;
mysql> source /root/nacos-mysql.sql
```


2. application.properties 配置



在 nacos 的解压目录 nacos/ 的 conf 目录下，有配置文件 application.properties，修改 conf/application.properties 文件，增加支持 MySQL 数据源配置
```
spring.datasource.platform=mysql

db.num=1
db.url.0=jdbc:mysql://192.168.223.135:3306/nacos_config?characterEncoding=utf8&connectTimeout=1000&socketTimeout=3000&autoReconnect=true
db.user=root
db.password=123456
```



3. 配置集群配置文件



在 nacos 的解压目录 nacos/ 的 conf 目录下，有配置文件 cluster.conf，请每行配置成ip:port。（请配置3个或3个以上节点）


```bash
cp cluster.conf.example cluster.conf
vim cluster.conf
```
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606487373012-84befe54-99be-4e6f-a5d3-13be79cc0d86.png#align=left&display=inline&height=154&margin=%5Bobject%20Object%5D&name=image.png&originHeight=154&originWidth=246&size=5459&status=done&style=none&width=246)


4. 编辑 Nacos 的启动脚本 startup.sh，使它能够接受不同的启动端口



修改前

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606488337394-e1e67699-037f-4b33-b2b5-647ecf9344cf.png#align=left&display=inline&height=78&margin=%5Bobject%20Object%5D&name=image.png&originHeight=78&originWidth=670&size=11210&status=done&style=none&width=670)

修改后

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606488400490-81afe926-b941-417b-a086-804c36939c32.png#align=left&display=inline&height=81&margin=%5Bobject%20Object%5D&name=image.png&originHeight=81&originWidth=938&size=12922&status=done&style=none&width=938)


5. 配置 Nginx 作为负载均衡器



[Nginx安装教程](https://cxhello.github.io/docs/#/article/%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85/Nginx%E5%AE%89%E8%A3%85%E4%B8%8E%E9%85%8D%E7%BD%AE)


在 nginx.conf 文件`#gzip  on;`下方添加如下内容
```
upstream cluster {
	server 192.168.223.137:3333;
	server 192.168.223.137:4444;
	server 192.168.223.137:5555;
}
server {
	listen       1111;
	server_name  localhost;
	location / {
		#root front;
		#index index.htm;
		proxy_pass http://cluster;
	}
}
```


6. 启动测试



```bash
# 启动nacos集群
sh startup.sh -p 3333
sh startup.sh -p 4444
sh startup.sh -p 5555
ps -ef | grep nacos | grep -v grep | wc -l
# 启动nginx
/usr/local/nginx/sbin/nginx
ps -ef | grep nginx
# 浏览器访问
http://192.168.223.135:1111/nacos
```
![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606490093557-89e84535-5aa8-49e3-abd4-910c4dc041b6.png#align=left&display=inline&height=59&margin=%5Bobject%20Object%5D&name=image.png&originHeight=59&originWidth=538&size=4080&status=done&style=none&width=538)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606490945408-7ad57595-af76-43b5-a359-52f2390829e8.png#align=left&display=inline&height=96&margin=%5Bobject%20Object%5D&name=image.png&originHeight=96&originWidth=818&size=11849&status=done&style=none&width=818)

新增一个配置进行测试查看是否存入数据库

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606491336960-97a5767d-efb9-4beb-bc67-79567ab9ea81.png#align=left&display=inline&height=528&margin=%5Bobject%20Object%5D&name=image.png&originHeight=528&originWidth=1476&size=52741&status=done&style=none&width=1476)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606491560813-87708009-698e-4fff-b00b-d4f1f7fc71b5.png#align=left&display=inline&height=396&margin=%5Bobject%20Object%5D&name=image.png&originHeight=396&originWidth=510&size=22023&status=done&style=none&width=510)

在 nacos-spring-cloud-provider-example 中将 application.properties 中服务注册的地址修改为 spring.cloud.nacos.discovery.server-addr=192.168.223.135:1111 进行测试

![image.png](https://cdn.nlark.com/yuque/0/2020/png/2584604/1606494749377-769bfeba-98a0-4ab1-81e9-fc22bdc3e6ff.png#align=left&display=inline&height=432&margin=%5Bobject%20Object%5D&name=image.png&originHeight=432&originWidth=1890&size=45192&status=done&style=none&width=1890)
