---
layout: post
title:  "Disconf Hello world"
date:   2016-05-06 09:49:15 +0800
categories: jekyll update
---

## Disconf,啥玩意
Distributed Configuration Management Platform(分布式配置管理平台)

[官方地址](https://github.com/knightliao/disconf)

### 主要目标
* 部署极其简单：同一个上线包，无须改动配置，即可在 多个环境中(RD/QA/PRODUCTION) 上线

* 部署动态化：更改配置，无需重新打包或重启，即可 实时生效

* 统一管理：提供web平台，统一管理 多个环境(RD/QA/PRODUCTION)、多个产品 的所有配置
核心目标：一个jar包，到处运行

## Begin Hello World

### 先弄下来



{% highlight linux %}
git clone https://github.com/knightliao/disconf.git
{% endhighlight %}

完了之后三个子项目

* disconf-web, 服务端，管理平台

* disconf-client，客户端

* disconf-core 核心

### 首先搞一下服务端-disconf-web

#### 搞之前，你得先装好这些

* 安装`Mysql`（Ver 14.12 Distrib 5.0.45, for unknown-linux-gnu (x86_64) using EditLine wrapper）

* 安装`Tomcat`（apache-tomcat-7.0.50）

* 安装`Nginx`（nginx/1.5.3）

* 安装 `zookeeeper` （zookeeper-3.3.0）

* 安装 `Redis` （2.4.5）

#### 正式稿之前也看看这些

#### disconf-web 项目结构

* `bin` 里面一个Python的脚本,一个数据库数据,用来导入权限的

* `deploy` 安装和部署项目

* `html` nginx 访问的静态管理页面

* `profile` 服务器配置文件

* `sql` 

* `src`

#### 建立disconf配置目录

要建立两个目录(自定义)

* `conf` 存放disconf-web配置文件

* `war` 存放deploy后的war文件和nginx访问的html

{% highlight linux %}
mkdir /root/disconf/conf

mkdir /root/disconf/war

{% endhighlight %}

#### 然后把两个目录导出到环境变量

{% highlight linux %}

export ONLINE_CONFIG_PATH=/root/disconf/config

export WAR_ROOT_PATH=/root/disconf/war

{% endhighlight %}

#### 然后拷贝`profile/rd/*`到`/root/disconf/config`中,注意将`application-demo.properties`改成`application.properties`

然后修改`/root/disconf/config`中的文件,把各种`mysql`,`redis`,`zookpeer`配置改成自己的

之后就开始尝试构建项目

{% highlight linux %}

cd disconf-web
sh deploy/deploy.sh

{% endhighlight %}

构建成功后会 

![deploy.sh构建成功后](http://ww4.sinaimg.cn/large/ac342fedjw1f3lli6veurj20fc016zkf.jpg "deploy.sh构建成功后")

然后会在`/root/disconf/war`中生成如下结构

* `-disconf-web.war`  
* `-html`
* `-META-INF`  
* `-WEB-INF`

#### 开始初始化数据库

可以参考 sql/readme.md 来进行数据库的初始化。

里面默认有6个用户

如果想自己设置初始化的用户名信息，可以参考代码来自己生成用户：

`src/main/java/com/baidu/disconf/web/tools/UserCreateTools.java`

#### 部署war到tomcat

修改server.xml文件，在Host结点下设定Context：

`<Context path="" docBase="/root/disconf/war"></Context>`

然后启动tomcat..

#### 部署Nginx

{% highlight linux %}

upstream disconf {
    server 127.0.0.1:8080;
}

server {

    listen   80;
    server_name localhost;
    access_log /home/work/var/logs/disconf/access.log;
    error_log /home/work/var/logs/disconf/error.log;

    location / {
        root /root/disconf/war ;
        if ($query_string) {
            expires max;
        }
    }

    location ~ ^/(api|export) {
        proxy_pass_header Server;
        proxy_set_header Host $http_host;
        proxy_redirect off;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Scheme $scheme;
        proxy_pass http://disconf;
    }
}

{% endhighlight %}

访问 [http://localhost:80](http://localhost:80)

默认用户 ： 

{% highlight linux %}

username : admin
password : admin

{% endhighlight %}

![ddisconf-web index](http://cheart-blog.oss-cn-beijing.aliyuncs.com/Screen%20Shot%202016-05-06%20at%201.56.31%20PM.png "disconf-web idnex")

至此，disconf-web端部署成功

### 然后开始搞disconf-client

说白了，也就是开始在项目里配置disconf

#### maven依赖

{% highlight maven %}

<dependency>
    <groupId>com.baidu.disconf</groupId>
    <artifactId>disconf-client</artifactId>
    <version>2.6.31</version>
</dependency>

{% endhighlight %}

### 加入并配置disconf.properties

{% highlight properties %}

# 是否使用远程配置文件
# true(默认)会从远程获取配置 false则直接获取本地配置
disconf.enable.remote.conf=true

#
# 配置服务器的 HOST,用逗号分隔  127.0.0.1:8000,127.0.0.1:8000
#
disconf.conf_server_host=192.168.88.30:8080

# 版本, 请采用 X_X_X_X 格式 
disconf.version=1.0.0

# APP 请采用 产品线_服务名 格式
disconf.app=cwang

# 环境
disconf.env=rd

# 忽略哪些分布式配置，用逗号分隔
disconf.ignore=

# 获取远程配置 重试次数，默认是3次

disconf.conf_server_url_retry_times=1

# 获取远程配置 重试时休眠时间，默认是5秒
disconf.conf_server_url_retry_sleep_seconds=1

# 用户指定的下载文件夹, 远程文件下载后会放在这里
disconf.user_define_download_dir=./

{% endhighlight %}

### 配置application-context

#### 在application-context.xml 中加入

{% highlight xml %}

 <!-- 第一次扫描，静态扫描 -->
    <bean id="disconfMgrBean" class="com.baidu.disconf.client.DisconfMgrBean"
          destroy-method="destroy">
        <property name="scanPackage" value="com.example.disconf.demo"/>
    </bean>
    <!-- 第二次扫描，动态扫描 -->
    <bean id="disconfMgrBean2" class="com.baidu.disconf.client.DisconfMgrBeanSecond"
          init-method="init" destroy-method="destroy">
    </bean>
 
    <!-- 使用托管方式的disconf配置(无代码侵入, 配置更改不会自动reload)-->
    <bean id="configproperties_no_reloadable_disconf"
          class="com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean">
        <property name="locations">
            <list>
                <!-- 配置需要注入的远程文件 -->
                <value>config-goods.properties</value>
            </list>
        </property>
    </bean>
    
    <!-- 配置PropertyPlaceholderConfigurer -->
    <bean id="propertyConfigurerForProject1"
      class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="ignoreResourceNotFound" value="true"/>
        <property name="ignoreUnresolvablePlaceholders" value="true"/>
        <property name="propertiesArray">
            <list>
                <ref bean="configproperties_no_reloadable_disconf"/>
            </list>
        </property>
    </bean>

{% endhighlight %}


{% highlight xml %}

<import resource="classpath*:spring/spring-db.xml" />
<import resource="classpath*:spring/spring-dubbo.xml" />
<import resource="classpath*:spring/spring-mq.xml" />
<import resource="classpath*:spring/spring-redis.xml" />

{% endhighlight %}

然后在这些引入的xml配置文件里就可以直接使用`${}`占位符引入配置了

也可以`<import resource="classpath*:spring/spring-redis.xml" />`引用一个文件，前提是引入的文件路径要和前面加入的`disconf.properties`中的`disconf.user_define_download_dir` 所配置的路径是一致的

接下来尝试启动项目

{% highlight log %}

2016-05-07 11:05:47,907 [main-EventThread] INFO  com.baidu.disconf.core.common.zookeeper.inner.ConnectionWatcher - zk SyncConnected
2016-05-07 11:05:47,907 [main] INFO  com.baidu.disconf.core.common.zookeeper.inner.ConnectionWatcher - zookeeper: 127.0.0.1:2181 , connected.
2016-05-07 11:05:47,908 [main] INFO  com.baidu.disconf.core.common.zookeeper.ZookeeperMgr - zoo prefix: /disconf
2016-05-07 11:05:47,917 [main] INFO  com.baidu.disconf.client.DisconfMgr - ******************************* DISCONF END FIRST SCAN *******************************
2016-05-07 11:05:48,084 [main] WARN  org.springframework.beans.GenericTypeAwarePropertyDescriptor - Invalid JavaBean property 'locations' being accessed! Ambiguous write methods found next to actually used [public void com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean.setLocations(java.util.List)]: [public void org.springframework.core.io.support.PropertiesLoaderSupport.setLocations(org.springframework.core.io.Resource[])]
2016-05-07 11:05:48,152 [main] INFO  com.baidu.disconf.client.addons.properties.ReloadablePropertiesFactoryBean - Loading properties file from class path resource [config-goods.properties]
2016-05-07 11:05:48,211 [main] INFO  com.baidu.disconf.client.DisconfMgr - ******************************* DISCONF START SECOND SCAN *******************************
2016-05-07 11:05:48,215 [main] INFO  com.baidu.disconf.client.DisconfMgr - Conf File Map: 
disconf-file:   config-goods.properties 
    DisconfCenterFile [
    keyMaps={}
    additionalKeyMaps={mysql.master.validationQuery=select 1, mysql.master.initialSize=5, mysql.master.timeBetweenEvictionRunsMillis=3000, mysql.master.maxActive=50, mq.address=192.168.88.45:9876, mysql.master.testWhileIdle=true, mysql.master.driverClassName=com.mysql.jdbc.Driver, mysql.master.testOnBorrow=true, redis.address1=192.168.88.41:6379, redis.address2=192.168.88.42:6379, dubbo.zookeeper.address=192.168.88.148:2181,192.168.88.149:2181,192.168.88.201:2181, mysql.master.minIdle=10, redis.address3=192.168.88.43:6379, redis.address4=192.168.88.44:6379, redis.address5=192.168.88.46:6379, mysql.master.maxIdle=50, mysql.master.minEvictableIdleTimeMillis=6000, redis.address6=192.168.88.47:6379, mysql.master.url=jdbc:mysql://192.168.88.43:3306/ync?autoReconnect=true&useUnicode=true&characterEncoding=utf-8, mysql.master.password=ync365.com, dubbo.zookeeper.register=true, mysql.master.username=root, mysql.master.testOnReturn=true}
    cls=null
    remoteServerUrl=/api/config/file?app=cwang&env=rd&type=0&key=config-goods.properties&version=1.0.0]

2016-05-07 11:05:48,215 [main] INFO  com.baidu.disconf.client.DisconfMgr - Conf Item Map: 

2016-05-07 11:05:48,215 [main] INFO  com.baidu.disconf.client.DisconfMgr - ******************************* DISCONF END *******************************


{% endhighlight %}

可以看到console里面的这些log就是disconf从服务端获取的配置信息

client部署成功

### 另外一种配置disconf-client的方式

由于`disconf.properties`文件中的配置决定了项目使用了哪个环境的哪个版本的配置，所以在本地开发时还比较方便，但是如果打包到线上时，就比较麻烦，因为要每次打包前修改`disconf.properties`,所以可以不加入`disconf.properties`这个文件,而是在项目部署打包成功后启动项目运行时加入Disconf的环境版本等配置信息，这样Disconf会根据这些java启动时参数拉取指定配置的配置文件

#### 如果没有`disconf.properties`,可以这样启动项目

{% highlight java %}

java -jar -Ddisconf.env=rd -Ddisconf.conf_server_host=192.168.88.30:8080 -Ddisconf.app=cwang -Ddisconf.version=1.0.0 -Ddisconf.user_define_download_dir=./classes -Ddisconf.enable.remote.conf=true ync-goods-1.0.14-SNAPSHOT.jar

{% endhighlight %}

这样就可以了


#### 延伸

如果项目的rd,qa,local,online四个环境分配配置了四个tomcat容器,那么也可以直接把上面的那些java启动参数放倒tomcat的启动脚本中

编辑tomcat`bin/catalina.sh`文件，配置`JAVA_OPTS`

{% highlight java %}

JAVA_OPTS="-Ddisconf.env=rd -Ddisconf.conf_server_host=192.168.88.30:8080 -Ddisconf.app=cwang -Ddisconf.version=1.0.0 -Ddisconf.user_define_download_dir=./classes -Ddisconf.enable.remote.conf=true"

{% endhighlight %}

更多配置项请参考 [Disconf启动配置参数项](https://github.com/knightliao/disconf/wiki/配置说明)


这样特定环境的项目跑在特定环境的tomcat里就可以了. [可参考](https://github.com/knightliao/disconf/wiki/Tutorial9)


[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
