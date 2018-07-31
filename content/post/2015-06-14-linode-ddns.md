---
title: "通过 Linode API 实现动态 DNS"
date: 2015-06-14
draft: false
author: "Mu Xian Ming"

categories: [Infra]
tags: [Linux, Linode, DDNS]
---

Linode的VPS用了也有几年了，最近才发现它的API的实用性，其中一项就是可以简单又灵活的实现动态DNS。

要使用DNS相关的API先决条件当然是得由Linode来托管你的域名，关于Linode的DNS托管可以看下[这里](https://www.linode.com/docs/networking/dns/dns-manager)。下面进入主题：

### **服务器端的配置**

进入账号后点击DNS Manager标签页，
![DNS Manager](/linode-ddns/1.png)
然后在下面的列表里点击自己的域名，进入该域名的编辑页面。

之后点击A/AAAA Records列表下的“Add a new A record”链接，新增一个域名和IP的对应关系。
![Add a new record](/linode-ddns/2.png)
Hostname填想要配置成的动态域名，例如home.example.com；IP Address可以先随便写一个，比如127.0.0.1，因为当动态域名配置好之后这个IP地址是会被自动更新的；最后的TTL应该设置成一个稍短的时间，因为一般来说ISP会比较频繁的更新你的IP地址，这样域名应该设置较短的存活时间以及时反映IP的变化。
接下来要实现在客户端(使用动态域名指向的IP的设备)周期性的更新刚刚配置的域名所对应的IP。

### **客户端的配置**

Linode提供了[不少API](https://www.linode.com/api/dns)用以实现对DNS的查询和操作。想要使用这些API得先申请一个API Key。

点击页面右上角的my profile，然后点击API Keys标签页。
![API Key](/linode-ddns/3.png)
Label处填写一个API Key的标签，比如DDNS。Expires选Never，永远不过期。创建之后把key复制下来保存好。

接下来在浏览器里打开下面这个链接，用这个API来查看你所有的域名：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.list
```

api_key要等于刚才申请的key。返回的结果是一个JSON对象，找到你的域名对应的DOMAINID，记好。再打开下面这个链接查看你的域名下的所有记录：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.resource.list&domainid=your-domain-id
```

api_key和domainid都要换成你刚刚记下来的内容。在返回的结果里找到你的动态域名(home)对应的RESOURCEID。最后这个API就是会把域名的IP更新为API调用的地址：

```
https://api.linode.com/?api_key=your-api-key&api_action=domain.resource.update&domainid=your-domain-id&resourceid=your-resource-id&target=[remote_addr]
```

照例把api\_key，domainid和resourceid对应成刚才记下来的内容，最后的[remote\_addr]代表的就是API调用方(打开这个链接的电脑)的IP地址，不需要更改。最后的任务就是在客户端创建一个cron job来周期性的调用Linode API：

```bash
crontab -e
```
打开当前用的cron任务文件。在文件里加一行：

```bash
*/30 * * * * /bin/echo `/bin/date`: `/usr/bin/wget -qO- --no-check-certificate https://api.linode.com/?api_key=your-api-key\&api_action=domain.resource.update\&domainid=your-domain-id\&resourceid=your-resource-id\&target=[remote_addr]` >> /var/log/linode_dyndns.log
```
每半小时更新一次动态域名对应的IP。到这就大功告成了:)

P.S. 如果还没有Linode账号希望能用我的referral链接注册:P  [注册Linode](https://www.linode.com/?r=b5e79f5672ed45c37b58ea482f99d13d7f0d347e)
