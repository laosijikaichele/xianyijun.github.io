---
title: Java知乎爬虫之模拟登录
date: 2016-06-14 17:52:41
tags: spider
---

# **Java知乎爬虫之模拟登录** #
最近在写知乎爬虫，记录一下开发遇到的一些问题,写知乎爬虫的话，首先就遇到第一个问题:模拟登录。
## **模拟登录** ##
由于Http是一个无状态的网络协议，而知乎需要我们登录之后才能执行其他的操作，很明显需要保持会话的一致性，保持事务的进行。
而实现Http的会话一致性的话，就需要用到cookie和session。
### **session** ###
对于session的话，我们首先要把session与session实现分开，session是一个抽象的概念，由于http是无状态协议，对于一些事务的操作，我们需要记录通信的状态，因此我们就将origin servers 和 user agents交互的过程称为session。
session实现的话，主要是记录与服务器端进行交互的user agent是哪一个user agent，这就需要一个id来对客户端进行标志，通过id，服务端就可以识别出客户端，记录在session中通信需要保存的信息。
常见的session的实现有cookie、重写url和隐藏表单域。
### **cookie** ###
cookie是http协议的记录在header中的一个字段，是服务端相应通过Set-Cookie对cookie进行设置，cooike根据存储的位置可以分为内存cookie和硬盘cookie，内存cookie是通过浏览器来维护的，当浏览器关闭的时候，cookie就消失了，而硬盘cookie是存储在硬盘中，它有一个过期时间，用户可以对其进行设置，除非是用户手动删除或者过了过期时间，否则cookie一直存在与硬盘中，不过需要注意的是cookie会随着每次http请求中附带，增大了服务器流量，而且还是明文传输的，需要注意安全(除非是https)，cookie的大小限制为4kb左右。
如果想对cookie的详细信息进行进一步了解的话，可以自行阅读[rfc2109](https://www.ietf.org/rfc/rfc2109.txt)。

## **Java实现** ##
### **请求数据** ###
如果我们想要进行模拟登录的话，首先需要知道登录要对服务器端进行传输的数据和请求的地址，我们可以通过chrome developer kit查看请求提交的数据，通过F12打开工具。我们可以看到。
![data](http://7xrl91.com1.z0.glb.clouddn.com/Selection_031.png)
![data](http://7xrl91.com1.z0.glb.clouddn.com/Selection_033.png)
请求地址为https://www.zhihu.com/login/email
需要传输的数据为_xsf、password、remember_me、email。
password、email和remember_me我们看参数名就可以很清楚的知道要传输的数据是什么，不过_xsf又是什么呢。
我们ctrl+u查看页面源码发现
![_xsf](http://7xrl91.com1.z0.glb.clouddn.com/Selection_032.png)
它是一个隐藏表单域，是在请求页面的时候随机生成的字符串，这样是我们可以先获取html页面，然后抽取表单域，获取到_xsf的值，再发起登录请求。
### **HttpClient客户端模拟请求** ###
在了解请求地址和传输数据，我们就可以进行编码了。
我使用了HttpClient作为http客户端和Jsoup进行html解析。

- 首先获取_xsf的值

```java
HttpGet get = new HttpGet(Constant.ZHIHU_URL);
CloseableHttpResponse response = client.execute(get, context);
String responseHtml = EntityUtils.toString(response.getEntity());
String xsrfValue = responseHtml.split("<input type=\"hidden\" name=\"_xsrf\" value=\"")[1].split("\"/>")[0];
```
- 模拟登陆请求

不过这此处我们需要人工输入验证码，我们可以将图片下载到本地，然后手动输入验证码来完成这一步。

```java

List<NameValuePair> valuePairs = new LinkedList<NameValuePair>();
valuePairs.add(new BasicNameValuePair("_xsrf", xsrfValue));
valuePairs.add(new BasicNameValuePair("email", "email@email.com");
valuePairs.add(new BasicNameValuePair("password", "password"));
valuePairs.add(new BasicNameValuePair("remember_me", "true"));
valuePairs.add(new BasicNameValuePair("captcha", valieCode));
UrlEncodedFormEntity entity = new UrlEncodedFormEntity(valuePairs, Consts.UTF_8);
HttpPost loginRequest = new HttpPost(Constant.ZHIHU_LOGIN_URL);
loginRequest.setEntity(entity);
CloseableHttpResponse loginResponse = null;
try {
	loginResponse = client.execute(loginRequest, context);
} catch (ClientProtocolException e) {
	e.printStackTrace();
} catch (IOException e) {
	e.printStackTrace();
}
return loginResponse;

```

- 下载验证码图片

验证码图片的地址为固定地址，https://www.zhihu.com/captcha.gif?type=login，不过要注意此处要保持请求的上下文，我们可以在execute请求的时候传入对应的HttpContext。

```java

HttpGet method = new HttpGet(url);
try {
	CloseableHttpResponse response = client.execute(method, context);
	File file = new File(savePath);
	if (!file.exists() && !file.isDirectory()) {
		file.mkdir();
	}
	file = new File(savePath + fileName);
	if (!file.exists() || isReplacable) {
		OutputStream os = new FileOutputStream(file);
		InputStream is = response.getEntity().getContent();
		byte[] buffer = new byte[(int) response.getEntity().getContentLength()];
		while (true) {
			int len = is.read(buffer);
			if (len == -1) {
				break;
			}
			byte[] temp = new byte[len];
			System.arraycopy(buffer, 0, temp, 0, len);
			os.write(temp);
		}
		os.close();
		is.close();
	}
} catch (ClientProtocolException e) {
	e.printStackTrace();
} catch (IOException e) {
	e.printStackTrace();
} finally {
	method.releaseConnection();
}

```
- 序列化cookie

为了不用每次都模拟登录再去爬去数据，我们可以将cookie序列化到磁盘，在需要启动爬虫的时候，对cookie进行加载读取。

```java
context.setCookieStore((CookieStore) LoginCookiesHelper.antiSerializeCookies("/cookies"));
```
反序列化
```java
public static Object antiSerializeCookies(String name) {
	InputStream fis = LoginCookiesHelper.class.getResourceAsStream(name);
	ObjectInputStream ois = null;
	Object object = null;
	try {
		ois = new ObjectInputStream(fis);
		object = ois.readObject();
	} catch (IOException e) {
		e.printStackTrace();
	} catch (ClassNotFoundException e) {
		e.printStackTrace();
	} catch (NullPointerException e) {
		e.printStackTrace();
	} finally {
		try {
			if (fis != null) {
				fis.close();
			}
			if (ois != null) {
				ois.close();
			}
		} catch (IOException e) {
			e.printStackTrace();
		}
	}
	return object;
}
```
序列化
```java
public static void serializeCookies(CookieStore cookieStore, String path) {
    ObjectOutputStream oos = null;
    FileOutputStream fos = null;
    try {
        File file = new File(path);
        fos = new FileOutputStream(file);
        oos = new ObjectOutputStream(fos);
        oos.writeObject(cookieStore);
        oos.flush();
    } catch (IOException e) {
        logger.info(e.getMessage(), e);
    } finally {
        try {
            if (oos != null) {
                oos.close();
                oos = null;
            }
            if (fos != null) {
                fos.close();
                fos = null;
            }
        } catch (IOException e) {
            logger.error(e.getMessage(), e);
        }
    }
}
```

```java
serializeCookies(context.getCookieStore(), Constant.COOIKES_SERIALIZE_PATH);
```

不过此处需要记得设置cookieSpec，cookie管理规范
>网景公司草案：这个规范符合由网景通讯发布的原始草案规范。应当避免，除非有绝对的必要去兼容遗留代码。
RFC 2109：官方HTTP状态管理规范并取代的老版本，被RFC 2965取代。
RFC 2965：官方HTTP状态管理规范。
浏览器兼容性：这个实现努力去密切模仿（mis）通用Web浏览器应用程序的实现。比如微软的Internet Explorer和Mozilla的FireFox浏览器。
最佳匹配：’Meta’（元）cookie规范采用了一些基于又HTTP响应发送的cookie格式的cookie策略。它基本上聚合了以上所有的实现到以一个类中。
我们这里用的是CookieSpecs.BROWSER_COMPATIBILITY

```java
RequestConfig requestConfig = RequestConfig.custom().setConnectTimeout(15000)
				.setSocketTimeout(15000).setConnectionRequestTimeout(15000)
				.setCookieSpec(CookieSpecs.BROWSER_COMPATIBILITY).build();
builder.setDefaultSocketConfig(socketConfig).setDefaultRequestConfig(requestConfig);
```
