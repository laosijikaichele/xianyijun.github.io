---
title: Tomcat共享session配置redis Manager
date: 2016-06-01 21:42:13
tags: Tomcat
---

# **Tomcat共享session配置redis Manager** #
主要对使用nginx、tomcat、redis搭建集群实现session共享的配置记录
## **环境配置** ##
测试环境配置如下，需要自身下载nginx、tomcat、redis具体安装自行谷歌。

- Linux mint 
- nginx 1.10.0
- tomcat_1 7.0.54
- tomcat_2 7.0.54
- redis 2.84

## **构建tomocat-redis-session-manager** ##
构建tomcat-redis-session-manager需要配置jedis和commons-pool依赖
博主使用的是

- jedis-2.5.2
- tomcat-redis-session-manager-2.0.0
- common-pool2-2.2

将jedis、tomcat-redis-sessio-manager、common-pool对应的jar包下载放到tomcat根目录下lib文件夹中进行类加载。
## **Tomcat配置**　##
配置jar包依赖之后要需要对tomcat进行配置。
```xml
   <Valve className="com.orangefunction.tomcat.redissessions.RedisSessionHandlerValve" />
    <Manager className="com.orangefunction.tomcat.redissessions.RedisSessionManager"
            host="localhost" port="6379" database="0" maxInactiveInterval="60" />
```
## **Nginx配置** ##
nginx进行负载均衡配置(upstream),在nginx根目录的conf.d文件夹创建tomcat.conf文件，然后添加以下信息。
```
upstream tomcat｛
    server 127.0.0.1:8080 max_fails=0 weight=1;
    server 127.0.0.1:9090 max_fails=0 weight=1;
｝
```
然后在nginx.conf文件中添加
```
location / {
    proxy_pass http://tomcat;
}
```
## **Session测试** ##
session测试页面index.jsp
Tomcat 1 index.jsp
```html
<%@ pagelanguage="java" %>
<html>
      <head>
      		<title>Tomcat 1</title>
      	</head>
<body>
<h1 style="color: red;">Tomcat 1</h1>

	<table align="centre" border="1">

		<tr>
			<td>Session ID</td>
			<td><%= session.getId() %></td>
		</tr>
		<tr>
			<td>Created on</td>
			<td><%= session.getCreationTime() %></td>
		</tr>
	</table>
</body>
</html>
sessionID:<%=session.getId()%>
<br>
SessionIP:<%=request.getServerName()%>
<br>
SessionPort:<%=request.getServerPort()%>
<%
out.println("This is Tomcat Server 1");
%>
```
Tomcat 2 index.jsp
```
<%@ pagelanguage="java" %>

<html>
      <head>
      		<title>Tomcat2</title>
      	</head>
<body>
<h1 style="color: red;">Tomcat 2</h1>

	<table align="centre" border="1">

		<tr>
			<td>Session ID</td>
			<td><%= session.getId() %></td>
		</tr>
		<tr>
			<td>Created on</td>
			<td><%= session.getCreationTime() %></td>
		</tr>
	</table>
</body>
</html>
sessionID:<%=session.getId()%>
<br>
SessionIP:<%=request.getServerName()%>
<br>
SessionPort:<%=request.getServerPort()%>
<%
out.println("This is Tomcat Server 2");
%>
```

我们可以看到Tomcat Server变了，而对应的session id确没有变化，实现了session共享。