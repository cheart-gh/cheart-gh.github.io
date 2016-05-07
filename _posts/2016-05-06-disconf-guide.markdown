---
layout: post
title:  "Hello world for Disconf"
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
                <value>config-goods.properties</value>
            </list>
        </property>
    </bean>
    
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

需要说明的是`com.baidu.disconf.client.DisconfMgrBean`是静态的disconf Bean 管理器,也就是每次项目构建或者开始运行的时候扫描注入，而`com.baidu.disconf.client.DisconfMgrBeanSecond`是动态的，也就是扫描管理web端配置文件发生的变化




Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
