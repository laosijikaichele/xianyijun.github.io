---
title: Java设置ip代理方法
date: 2016-06-17 11:52:37
tags: spider
---

# **Java设置ip代理方法** #
在写知乎爬虫的时候，如果访问频率过快的话，账号会被限制访问，我们可以通过动态替换ip代理来减少别识别爬虫的概率。
我们可以根据访问http://www.ip.cn/　查看访问的ip是否成功进行了代理。
## **Jdk设置系统属性** ##
```java
public static void main(String[] args) throws IOException {

    System.setProperty("http.maxRedirects", "50");
    System.getProperties().setProperty("proxySet", "true");
    String ip = "103.35.149.133";
    System.getProperties().setProperty("http.proxyHost", ip);
    System.getProperties().setProperty("http.proxyPort", "1080");
    StringBuilder result = new StringBuilder();
    BufferedInputStream in = null;

    URLConnection connection = null;
    try {
        URL url = new URL("http://www.ip.cn/");
        connection = url.openConnection();
        connection.setRequestProperty("User-Agent",
                "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/50.0.2661.102 Chrome/50.0.2661.102 Safari/537.36");
        in = new BufferedInputStream(connection.getInputStream());
        String line = null;
        byte[] buffer = new byte[1024];
        while ((in.read(buffer)) != -1) {
            line = new String(buffer, "UTF-8");
            result.append(line);
        }
        System.out.println(Jsoup.parse(result.toString()));
    } finally {
        if (in != null) {
            in.close();
        }
        connection = null;
    }
}
```
如果想进一步了解jdk设置networking properties，可以去官方文档的[networking proprties](http://docs.oracle.com/javase/8/docs/api/java/net/doc-files/net-properties.html)

## **JDK Proxy** ##

```JAVA

public static void main(String[] args) throws IOException {
    Proxy proxy = new Proxy(Proxy.Type.HTTP, new InetSocketAddress("182.140.132.107", 8888));
    StringBuilder result = new StringBuilder();
    BufferedInputStream in = null;

    HttpURLConnection connection = null;
    try {
        URL url = new URL("http://www.ip.cn/");
        connection = (HttpURLConnection) url.openConnection(proxy);
        connection.setReadTimeout(10000);
        connection.setConnectTimeout(10000);
        connection.setRequestProperty("User-Agent",
                "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Ubuntu Chromium/50.0.2661.102 Chrome/50.0.2661.102 Safari/537.36");
        in = new BufferedInputStream(connection.getInputStream());
        String line = null;
        byte[] buffer = new byte[1024];
        while ((in.read(buffer)) != -1) {
            line = new String(buffer, "UTF-8");
            result.append(line);
        }
        System.out.println(Jsoup.parse(result.toString()));
    } finally {
        if (in != null) {
            in.close();
        }
        connection = null;
    }
}

```

## **HttpClient** ##
```java
public static void main(String[] args) throws IOException {
    HttpHost proxy = new HttpHost("112.122.219.96", 8118,"http");

    Registry<ConnectionSocketFactory> req = RegistryBuilder.<ConnectionSocketFactory> create()
            .register("http", PlainConnectionSocketFactory.INSTANCE)
            .register("https", SSLConnectionSocketFactory.getSocketFactory()).build();
    PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager(req);
    connectionManager.setDefaultMaxPerRoute(100);

    RequestConfig config = RequestConfig.custom().setProxy(proxy).setConnectTimeout(5000).setSocketTimeout(5000)
            .setConnectionRequestTimeout(5000).build();
    HttpClientBuilder builder = HttpClients.custom()..setUserAgent(Constant.DEFAULT_USER_AGENT).setConnectionManager(connectionManager)
            .setDefaultRequestConfig(config);

    CloseableHttpClient client = builder.build();
    try {

        HttpGet request = new HttpGet("http://www.ip.cn/");
        CloseableHttpResponse response = client.execute(request);
        HttpEntity entity = response.getEntity();
        String str = EntityUtils.toString(entity);
        Document doc = Jsoup.parse(str);
        System.out.println(doc);
    } finally {
        client.close();
    }
}
```