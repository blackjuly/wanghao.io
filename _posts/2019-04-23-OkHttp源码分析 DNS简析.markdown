---
layout:     post
title:      "OkHttp源码分析（三） DNS简析"
subtitle:   " \"OKHttp 源码系列分析\""
date:       2019-04-23 10:30:00
author:     "wanghao"
header-img: "img/post-bg-2015.jpg"
tags:
    - Android
    - 源码
    - OkHttp
---

> OkHttp在各个层面其实都为我们做到用户拓展，今天我们来简单了解下DNS

## 适读人群
1. 具有JAVA基础开发知识
2. 接触过HTTP,HTTPS，HTTP2,TCP，Socket等相关知识
3. 简单了解设计模式

## 背景
一般情况下，我们的针对一个网络请求的控制，如果只是加密或者解密，又或者简单记录log，OKhttp的拦截器基本可以满足我们的需求，但是对于我们日新月异发展的app来讲，各个方面也要求的也越来越高；最被大家关心的无疑就是网络请求的响应速度要求，我们都知道请求的其中一环便是DNS解析，但是一般想到DNS解析优化我们一般都想到这是后台关心的事情，对于我们前台有何干系？但是OKhttp还是提供了一个DNS方面的简单拓展，那我们来一起看看对我们前台可以做哪些文章......

## DNS拓展设计方案一览
废话不多说，先上成果，笔者目前发现DNS的拓展在我们客户端有两种场景可以用到

* 开发阶段，利用dns拓展可以实现 动态的网络环境切换，而无需更换app安装包
* 生产阶段，利用okhttp的dns拓展可以实现搭建 **HTTPDNS**，以达到提供更快的解析速度，和更好的容灾能力

具体实现且听笔者娓娓道来.....

### OKhttp的DNS拓展api展示
首先，简单了解下OKhttp的拓展形式，定义了这样的一个接口
```java
public interface Dns {
  
  List<InetAddress> lookup(String hostname) throws UnknownHostException;
}
```
传入一个hostname返回一组inetAddress（针对多机房，解析可以返回不止一个ip地址）；默认情况下，OKhtttp使用的是系统默认提供的JAVA的api去直接获取ip
```java
/**
   * A DNS that uses {@link InetAddress#getAllByName} to ask the underlying operating system to
   * lookup IP addresses. Most custom {@link Dns} implementations should delegate to this instance.
   */
  Dns SYSTEM = hostname -> {
    if (hostname == null) throw new UnknownHostException("hostname == null");
    try {
      return Arrays.asList(InetAddress.getAllByName(hostname));
    } catch (NullPointerException e) {
      UnknownHostException unknownHostException =
          new UnknownHostException("Broken system behaviour for dns lookup of " + hostname);
      unknownHostException.initCause(e);
      throw unknownHostException;
    }
  };

```
然后，会在okhttpClientBuilder构建时，没有指定dns时，将默认dns指定进去
```java
 public Builder() {
      //....忽略
      dns = Dns.SYSTEM;
     //....
    }
```
### 简单重写demo展示
```java
 OkHttpClient.Builder builder = new OkHttpClient.Builder();
	  builder.dns(new Dns() {
		
		@Override
		public List<InetAddress> lookup(String hostname) throws UnknownHostException {
			List<InetAddress>  adds = Dns.SYSTEM.lookup(hostname);
			return adds;
		}
	});
```
我们可以通过builder直接将自定义dns设置进去

## 拓展方法详解

### app内部一键网络切换 
一直以来我们android的app在开发期间，都会把baseUrl根据环境定义在gradle的buildType当中，根据不同需求打出对应的包
```
   debug {
            buildConfigField 'String', 'BASE_URL', '"http://axxx.cn/xxx/api/"'
   }
   release {
            buildConfigField 'String', 'BASE_URL', '"http://bxxx.cn/xxx/api/"'
   }
```
优势：在于我们可以不改动代码，直接在gradle配置文件中设值
劣势：如果是需要对比开发环境，测试环境的接口差异，又或者只是为了测试不同阶段的接口，反复还要安装不同环境app无疑是浪费时间
#### 痛点需求分析
那么我们就可以将上面的劣势拓展成以下需求
1. 测试接口可以在app中利用某个开关控制当前app处于哪个网络环境，即访问哪一个baseUrl
2. 要易于开启和关闭，不得造成线上问题
3. 对于实现该功能，尽量做到和业务代码分离减少入侵性，在场景不适用时可以方便和业务代码剥离
#### 方案介绍
基于以上分析，笔者想到了使用定制我们的okhttp的dns解析类，耦合与OKHTTP框架层面，基于上文我们可以知道定制主要就是覆盖重写一个类，所以，只要在不需要该类的情况下移除该类即可
#### 方案具体实现
由于涉及许多类合作，故笔者只简单介绍核心部分的一个思路
```java
public class Example {
  // 1. 有一个可以控制环境的变量的开关 
 public static String flag = "dev"; 
OkHttpClient.Builder builder = new OkHttpClient.Builder();
	  builder.dns(new Dns() {
		
		@Override
		public List<InetAddress> lookup(String hostname) throws UnknownHostException {
      //2.将所有环境的host组装成列表，方便在dns判断是否拦截切换
			List<String> list =  getTargetHosts();
      //3.判断只拦截指定的host
      if(list.contains(hostname)){
          //对应的 flag开关，将指定的 ip地址替换掉
          if("dev".equals(flag)){
            return getDev();
          }
          if("release".equals(flag)){
            return getRelease();
          }
      }
			List<InetAddress>  adds = Dns.SYSTEM.lookup(hostname);
			return adds;
		}
    //组装所有环境下的host
    private List<String> getTargetHosts(){
      List<String> list = new ArrayList<>();
      list.add("a.com");
      list.add("b.com");
      return list;
    }
    //返回开发环境的 ip地址
    private List<InetAddress> getDev(){
      	InetAddress address = InetAddress.getByName("a.com");
			  List<InetAddress> list = new ArrayList<>();
			  list.add(address);
        return list;
    }
    //返回生产环境的 ip地址
    private List<InetAddress> getRelease(){
      	InetAddress address = InetAddress.getByName("b.com");
			  List<InetAddress> list = new ArrayList<>();
			  list.add(address);
        return list;
    }
	});

}  
```

#### 方案劣势

当然此方案也有有一定劣势，就是url多个环境一定要除了域名以前其他都是一致的
例如：

```xml
dev: http://a.com/simpleProject/api/login
release: http://b.com/simpleProject/api/login
```

比如以下几种不规范情况就不行了

* 项目被修改

```xml
dev: http://a.com/simpleProject/api/login
release: http://b.com/simpleProject1/api/login

```

* http和https混用

```xml
dev: http://a.com/simpleProject/api/login
release: https://b.com/simpleProject/api/login
```

故仍有一定局限性

### HTTPDNS的定制使用

首先，由于HTTPDNS搭建涉及面很广，无法用简单一个示例展示清楚，故只做一个简单科普性质的分享，提供读者一个解决问题的思路！

#### 传统dns的痛点
要了解httpDNS,我们需要传统dns到底有怎样的问题，这里就简单举几类：
##### 域名缓存问题
1. 一般为了减少时间和资源的消耗一般dns服务器都会做本地缓存，而非每次都访问权威DNS服务器，所以，当我们ip发生改变而dns服务器缓存没有及时刷新就会产生问题
2. 同时由于缓存的不一定是每一个用户离的最近的服务器，这就导致全局负载均衡的失效
3. 有的运营商甚至进静态界面都进行缓存，但是平常界面没有问题，可以一旦原始界面有更新，而缓存没有更新都会造成影响
##### 域名转发
有些运营商（下文就代称为A）比较偷懒，，故直接把自己的解析的请求转发给其他运营商（下文代称为B），B运营商通过一系列DNS查询，得到了一个距离B运营商比较近的ip地址返回给A时会造成每一次的访问都是跨运营商的，速度会很慢！
##### 解析延迟问题
抛开运营商的问题，DNS本身查询是一个递归遍历的过程，所以这个本身也会带来时延，甚至解析超时！

#### 方案介绍
传统DNS问题众多，那我们怎么办呢？直接使用IP地址肯定不可取，所以就有了HTTPDNS!
即不走传统的DNS解析，通过自己的HTTP接口将ip地址返回，即每家基于http协议自己实现自己的域名解析，而一般使用这种方案的都是手机应用；通过在手机端嵌入支持HTTPDNS的客户端的sdk从而实现效果！由于本文重点不在HTTPDNS,故不做过多分析解释.....

#### 方案实现
HttpDNS服务提供商
[DNSPOD | D+](https://www.dnspod.cn/httpdns)
推荐几个网络比较推荐第三方的sdk

[新浪-安卓版(支持D+企业版加密功能)](https://github.com/CNSRE/HTTPDNSLib)

[七牛-安卓版(支持D+企业版加密功能)](https://github.com/qiniu/happy-dns-android)

[七牛-OC版(支持D+企业版加密功能)](https://github.com/qiniu/happy-dns-objc)

接入说明，大家可以网上大佬的详细博文说明，本文参考自 [https://www.jianshu.com/p/6bd131de81d3](https://www.jianshu.com/p/6bd131de81d3)

借用这位大佬的一个示例响应本次的主题，这是他们封装的一个解析类,代码实现效果并非重点，笔者也只是想用这个示例来为大家提供一种客户化DNS的场景与思路！
```java
public class HttpDns implements Dns {

    private DnsManager dnsManager;

    public HttpDns() {
        IResolver[] resolvers = new IResolver[1];
        try {
            resolvers[0] = new Resolver(getByName("119.29.29.29"));
            dnsManager = new DnsManager(NetworkInfo.normal, resolvers);
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }
    }

    @Override
    public List<InetAddress> lookup(String hostname) throws UnknownHostException {
        if (dnsManager == null)  //当构造失败时使用默认解析方式
            return Dns.SYSTEM.lookup(hostname);

        try {
            String[] ips = dnsManager.query(hostname);  //获取HttpDNS解析结果
            if (ips == null || ips.length == 0) {
                return Dns.SYSTEM.lookup(hostname);
            }

            List<InetAddress> result = new ArrayList<>();
            for (String ip : ips) {  //将ip地址数组转换成所需要的对象列表
                result.addAll(Arrays.asList(getAllByName(ip)));
            }
            return result;
        } catch (IOException e) {
            e.printStackTrace();
        }
        //当有异常发生时，使用默认解析
        return Dns.SYSTEM.lookup(hostname);
    }
}
```


## 总结

说了这么多，更多是希望大家可以有所了解DNS在OKhttp的使用场景，其实我们框架层面的很多拓展性，我们看起来没有用到不代表没有任何作用，开源大神的很多设计都是基于对场景掌握的数量很多，才能对很多关键性的位置都留下用户拓展的空间！一起共勉向大佬们学习！



## NEW FEATURE
1. RetryAndFollowUpInterceptor重试重定向的分析
2. BridgeInterceptor的分析
3. CacheInterceptor缓存机制分析
4. ConnectInterceptor连接复用相关分析
5. CallServerInterceptor实际IO操作分析
6. OKhttp同步与异步calls的调用管理分析












