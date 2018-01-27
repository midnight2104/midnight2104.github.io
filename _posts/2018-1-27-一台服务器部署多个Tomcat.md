---
layout: post
title: 一台服务器部署多个Tomcat
---

如果现在一台机器上已经部署了一个tomcat服务，无论这个tomcat是否已经注册为服务了，或者没有注册windows服务，或者注册了，都没关系。都可以采用下面的方法实现。 
如果该tomcat已经注册为windows服务了，从window的环境变量中找不到  
CATALINA_HOME和CATALINA_BASE，也可以采用下面的方式实现。  

当第一个tomcat启动后，后面tomcat的server.xml中的端口不管怎么改，仍然会报端口冲突。后来在dos下运行才发现所有的tomcat都会去找CATALINA_HOME和CATALINA_BASE这两个环境变量，因此步骤如下：   
1.使用压缩版的tomcat不能使用安装版的。   
2.第一个tomcat的配置不变。   
3.增加环境变量CATALINA_HOME2，值为新的tomcat的地址；增加环境变量CATALINA_BASE2，值为新的tomcat的地址。   
4.修改新的tomcat中的startup.bat，把其中的CATALINA_HOME改为CATALINA_HOME2。   
5.修改新的tomcat中的catalina.bat，把其中的CATALINA_HOME改为CATALINA_HOME2，CATALINA_BASE改为CATALINA_BASE2。   
6.修改conf/server.xml文件：   
6.1 <Server port="8005" shutdown="SHUTDOWN">把端口改为没有使用的端口。   
6.2 <Connector port="8080" maxHttpHeaderSize="8192"   
  maxThreads="150" minSpareThreads="25" maxSpareThreads="75"   
  enableLookups="false" redirectPort="8443" acceptCount="100"   
  connectionTimeout="20000" disableUploadTimeout="true" /> 把端口改为没有使用的端口。   
6.3<Connector port="8009"   
  enableLookups="false" redirectPort="8443" protocol="AJP/1.3" /> 把端口改为没有使用的端口。   

7成功！  

8 第三、第四.....等N台服务器参考3~6 步顺序进行即可！
