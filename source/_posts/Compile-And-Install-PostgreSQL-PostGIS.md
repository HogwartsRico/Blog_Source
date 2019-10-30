---
title: 编译安装PosgreSQL和PostGIS
date: 2016-09-12 20:43:34
tags: Linux
---

这篇文章主要记录在ubuntu server下编译安装postgresql9.3和postgis2.1  

# 下载PostgreSQL 
打开[PostgreSQL](https://www.postgresql.org/download/linux/ubuntu/)的[下载网站](http://www.enterprisedb.com/products-services-training/pgbindownload)  

下载源码   

![](2.png) 

我下载9.3版本的   




# 编译安装PostgreSQL

```
tar -jxvf postgresql-9.0.2.tar.bz2  
cd postgresql-9.0.2  
./configure --prefix=/usr/local/pgsql  <==配置软件的安装目录
make 
make install
```

在安装过程中可能或出现`make(build-essential)`  `gcc` `c++` `readline` `zlib`等包的缺失 

安装 build-essential

```
 sudo aptitude install build-essential
```

安装 g++ 

```
sudo apt-get install g++-4.8 
```
安装gcc 

`sudo apt-get install gcc-4.8` 
下面是安装zlib和readline的  

虽然可以加上 "--without-readline" 即`./configure --prefix=/usr/local/pgsql --without-readline `  从而避开这个ERROR，  
但Postgresql官方不推荐这么做，所以还是安装吧。  

```
sudo apt-get install zlib1g-dev

sudo apt-get install libreadline-dev 或apt-get install libreadline-gplv2-dev


```

![](3.png) 

![](4.png) 




PostgreSQL9.3 安装完成   

## 安装完成后配置  

PostgreSQL 不能以 root 用户运行，所以我们
### 创建 postgres 用户

```
adduser postgres
passwd postgres
```

一般执行adduser之后会自动让你输入密码和一些信息


### 创建postgresql 数据目录：


`mkdir /usr/local/pgsql/9.1/data/` 为啥不用`mkdir /usr/local/pgsql/9.1/data/`是为了防止以后安装了不同版本的pg 

`chown postgres /usr/local/pgsql/9.1/data或 chown postgres:postgres /usr/local/pgsql/data` 

`ls -ld /usr/local/pgsql/data` 


会出现如下界面

![](9.png) 

### 初始化postgresql数据目录

su postgres切换到postgres用户

/usr/local/pgsql/bin/initdb -D /usr/local/pgsql/9.3/data/

![](10.png) 

### 验证postgresql数据目录

ls -l /usr/local/pgsql/data

出现如下界面：

![](11.png) 

### 启动pgsql数据库

```
touch /usr/local/pgsql/9.3/data/logfile 创建日志文件
/usr/local/pgsql/bin/postgres -D /usr/local/pgsql/9.3/data > /usr/local/pgsql/9.3/data/logfile 2>&1 &  让postgresql在后台运行
```
出现如下界面：

![](12.png) 


### 创建测试数据库 

`/usr/local/pgsql/bin/createdb test` 

### 进入test数据库

`/usr/local/pgsql/bin/psql test` 


![](13.png) 

输入建表语句：

`CREATE TABLE mytable (id VARCHAR(20), name VARCHAR(30));`

建立完成后，会得到一条 “CREATED” 的信息，表示建立成功。


![](14.png) 

现在插入一条数据：

`INSERT INTO mytable VALUES('author', 'rico');` 


psql 返回类似 INSERT 0 1

查询插入是否成功：

`SELECT * FROM mytable;`

![](15.png) 


退出 psql :  
 
`\q`

### 如何关闭 PostgreSQL

  切换到 postgres 用户

`su - postgres`

关闭 PostgreSQL

`/usr/local/pgsql/bin/pg_ctl stop -D /usr/local/pgsql/9.3/data`

![](16.png) 

当然 你还可以查看下日志logfile

![](17.png)  


退出 postgres 用户

`exit`

参考 http://blog.csdn.net/longshengguoji/article/details/38468449  
http://blog.csdn.net/leonzhouwei/article/details/7934810

# 安装 pgAdmin

注意，有桌面才装pgadmin，只有命令行模式不要安装pgadmin 

[pgAdmin主页](https://www.pgadmin.org/download/source.php) 上有pgAdmin3和4

我这里安装pgAdmin3 的1.22版本    
[下载](https://www.postgresql.org/ftp/pgadmin3/release/v1.22.1/src/)  

1 安装必要的库  

```
sudo apt-get install libxml2-dev

sudo apt-get install libxslt1-dev

sudo apt-get install libpq-dev

sudo apt-get install wx-common libwxgtk2.8-dev

```

2  如果 `/usr/lib` 下有 `libcrypto.so`  

```
cd /usr/lib

sudo ln -s libcrypto.so.x.y.z libcrypto.so  // 创建软连接  libcrypto.so.x.y.z 是你的 /usr/lib 下已有的某个版本的crypto动态库文件名
```

3 切换到你的 pgAdmin3 解压后的目录后编译安装 pgAdmin3  

```
tar -zxvf pgadmin3-1.22.1.tar.gz 
cd pgadmin3-1.22.1
./configure
make all
sudo make install
```


`./configure`执行完成之后可能会[报错](http://www.guguncube.com/2798/pgadmin3-installation-latest-version) 

>configure: error: could not find a suitable C++ compiler to build pgAdmin  

解决办法：  

执行 `sudo apt-get install -y build-essential` //是为了 install g++ compiler   

然后就可以make了  
![](5.png) 

编译很慢，耐心等待   

![](6.png)

![](7.png) 

![](8.png) 


运行 pgAdmin3
```
cd /usr/local/pgadmin3/bin

sudo ./pgadmin3
```




# 下载PostGIS

我这里下载2.1版本的  
打开PostGIS的[源码网站](http://postgis.net/source/) 下载2.1的源码    
![](1.png) 


安装postgis之前还需要安装 [GEOS](http://trac.osgeo.org/geos), [Proj.4](https://trac.osgeo.org/proj/), [GDAL](http://gdal.org/), [LibXML2](http://www.xmlsoft.org/) 和 [JSON-C](https://github.com/json-c/json-c) 这几个库 

虽然这三个库不是安装postgis强制的，但是，没有这三个包，

postgis一定程度上失去了空间数据库的意义。因为Proj4提供了投影的相关操作，如postgis中的transform()函数，geos则为postgis提供了很多拓扑

检查功能的函数，如Touches(), Contains(), Disjoint() 还有一些空间操作函数，如Intersection(), Union() 以及 Buffer()等 ，而Libxml2则提供了对GML和KML的操作函数，如ST_GeomFromGML(), ST_GeomFromKML()等,如果丧失了这样特性，空间数据库将会怎样！

## GEOS
官网  https://trac.osgeo.org/geos 下载源码包 


编译安装 


```
tar -jxvf geos-3.5.0.tar.bz2 
cd geos-3.5
./configure 或加上 --prefix=/usr/local/geos 
make 
make install 
```

![](19.png)   

![](20.png) 

参考 https://trac.osgeo.org/geos/wiki/BuildingOnUnixWithCMake

## Proj.4
https://github.com/OSGeo/proj.4  

https://github.com/OSGeo/proj.4/wiki

```
./autogen.sh
./configure
If another path prefix is required, then execute:
./configure --prefix=/my/path 例如 ./configure --prefix=/usr/local/proj 
In either case, the directory of the prefix path must exist and be
writable by the installer.
After executing configure, execute:
make
make install
```

![](26.png) 

![](27.png) 

![](28.png) 


 

## GDAL
gdal官网 http://www.gdal.org/
GDAL 下载 https://trac.osgeo.org/gdal/wiki/DownloadingGdalBinaries
https://trac.osgeo.org/gdal/wiki/DownloadSource



编译指南  macos linux unix windows等
https://trac.osgeo.org/gdal/wiki/BuildHints  
https://trac.osgeo.org/gdal/wiki/BuildingOnUnix

```
./configure  或加上--prefix=/usr/local/gdal 
make 
make install 

```

## LibXML2
Libxml2是个C语言的XML程式库，能简单方便的提供对XML文件的各种操作，并且支持XPATH查询，及部分的支持XSLT转换等功能。Libxml2的下载地址是
[http://xmlsoft.org/](http://xmlsoft.org/)  

源码包列表 ftp://xmlsoft.org/libxml2/

我这里下载了libxml2-2.9.4.tar.gz  

![](18.png) 


它的git地址 https://git.gnome.org/browse/libxml2/


```
tar -zxvf tar -zxvf libxml2-2.9.4.tar.gz 
cd libxml22-2.9.4
./configure 或加上--prefix=/usr/local/xmllib2 
make 
make install
```


![](21.png) 


然而make过程中报错 了
![](22.png) 

报错说没有python 那就安装python

`apt-get install python-dev`

然后就make成功了

![](23.png) 

再继续make install

参考 http://www.cnblogs.com/shanzhizi/archive/2012/07/09/2583739.html

## JSON-C

json-c github  https://github.com/json-c/json-c
https://github.com/json-c/json-c/releases
下载解压 

直接`./configure`然后make会报这个错 
 
>error: variable 'size' set but not used [-Werror=unused-but-set-variable]

![](29.png) 


所以找到对应目录中的Makefile文件，找到 -Werror 字段，去掉-Werror，重新编译，则问题解决!

```
sed -i s/-Werror// Makefile.in tests/Makefile.in 
./configure --prefix=/usr/local/jsonc --disable-static       
make -j1
make install 
```


# 编译安装PostGIS
以上环境都准备好了，开始安装PostGIS

```
./configure --with-pgconfig=/usr/local/pgsql/bin/pg_config  --with-gdalconfig=/usr/local/gdal/bin/gdal-config 文件路径根据自己安装的
如果缺其他配置可以继续加上  例如 --with-projdir=/usr/local/proj --with-geosconfig=/usr/local/geos/bin/geos_config等 
make 
make install 


```

`./configure`之后的截图 
![](30.png) 

![](31.png) 

![](32.png) 



参考 http://blog.csdn.net/zuiaikg703/article/details/46776095
http://www.linuxfromscratch.org/blfs/view/svn/general/json-c.html



## 测试安装是否正确-----创建空间数据库。

熟悉windows环境下postgis的朋友，都会注意到，安装了postgis后，pgsql中多了一个数据库template_postgis，这是个空间数据库的模板，其实就是个空间数据库。  

而在linux环境下通过源码安装的postgis，默认没有创建这个空间数据库，下面我就用创建者个模板空间数据库来验证上面的安装是否正确。  
一旦创建了一个空间数据库模板，以后每次创建空间数据库只要在这个模板空间数据库上创建就可以了，省时省力！ 


```
切换用户
su - postgres 
创建日志文件 
touch /usr/local/pgsql/9.3/data/logfile

启动服务 
/usr/local/pgsql/bin/pg_ctl -D /usr/local/pgsql/9.3/data/ -l /usr/local/pgsql/9.3/data/logfile  start

创建数据库
/usr/local/pgsql/bin/createdb template_postgis  

创建空间数据库模板
/usr/local/pgsql/bin/psql -f /usr/local/pgsql/share/contrib/postgis-2.1/postgis.sql -d template_postgis

```
![](47.png) 




![](33.png) 

执行postgis.sql脚本，创建相关空间数据库相关的函数，类型，操作符等  
执行完这个脚本，该数据库就具有了空间特性了  


```
/usr/local/pgsql/bin/psql -f /usr/local/pgsql/share/contrib/postgis-2.1/spatial_ref_sys.sql -d template_postgis
```
![](34.png) 

该命令在postgis数据库中创建了spatial_ref_sys表，用于存放空间投影信息。 

```
createdb [-U username] -T template_postgis mypostgisdb
```
下次再创建数据库，只要以这个模板就可以了，不必每次都执行这个脚本,-U指定用户名，默认就是postgres

`psql my_spatial_db` 连接到创建的空间数据库  

`select postgis_full_version();`查询postgis的版本信息，包含用到的三个库信息
                                          

![](35.png) 


创建一张表 

![](36.png) 


增加一列空间字段

![](37.png) 


随意插入一列数据 

![](38.png) 


查看数据 


![](39.png) 

查看一些配置信息

![](40.png) 


# 修改白名单 

这时如果你打开web应用却发现连接不上PostgreSQL数据库

![](45.png) 


postgresql默认只开启了本地访问权限，如果有web应用程序不是本地的需要开启

`vi /usr/local/pgsql/9.3/data/pg_hba.conf`

![](41.png) 

IPV4区域 
如果你只想放开某个IP
`host    all             all             10.10.21.62/32            trust`

如果可以让任意IP都可连接 

`host    all         all         0.0.0.0/0             trust` 


IPV6区域就根据IPV6的来改 


然后重启一下  

切换到postgre用户

`su - postgres`  
 
到这里还不够
打开  
`vi /usr/local/pgsql/9.3/data/postgresql.conf`
你会看到默认只监听了本地
![](43.png)  

这时，你改成`*` 


![](44.png) 


重启 

`/usr/local/pgsql/bin/pg_ctl  -D /usr/local/pgsql/9.3/data/ -l /usr/local/pgsql/9.3/data/logfile   restart`

其他命令  


```

#停止服务
/usr/local/pgsql/bin/pg_ctl -D  /usr/local/pgsql/9.3/data/ -l /usr/local/pgsql/9.3/data/logfile  stop

#端口是否启用
netstat -anp | grep 5432
```

![](42.png) 




这时再开启web应用就可以连接上了。  

![](46.png) 


![](48.png) 



至此 ，从编译安装PostgreSQL到PostGIS到连接，创库，创空间模板，创空间库，应用连接数据库都可以了

(完)

# 参考

http://www.cnblogs.com/zhoulf/p/4040768.html
http://blog.csdn.net/dracotianlong/article/details/7907633
http://postgis.net/source/
https://trac.osgeo.org/postgis/wiki/UsersWikiInstall
http://postgis.net/install/
https://trac.osgeo.org/postgis/wiki/UsersWikiPostGIS21Ubuntu1404src
