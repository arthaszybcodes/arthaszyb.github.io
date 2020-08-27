---
layout: post
title:  "Personal Blog Website quick deploy Intro"
author: sal
categories: [ Lifestyle ]
tags: [ France ]
image: https://www.aqniu.com/wp-content/themes/anquanniu/timthumb.php?src=https://www.aqniu.com/wp-content/uploads/2015/06/211.jpg&h=450&w=1080&zc=1
rating: 1 

---

## 建站载体

1. github pages。 利用github提供的云部署方式，即可免费完成站点部署和运营。你只需要正常提交代码即可，非常方便还省钱，同时也实现了站点的版本维护。唯一的问题是国内的墙可能导致你的站点被屏蔽。
2. 自建站点。即自己提供服务器，外网IP，域名和DNS解析，自己部署站点服务，自己做好版本管理。优劣点一目了然。下面详述这种方式的步骤。



## 建站步骤



### 机器准备

购买一台云主机，如果不希望自己的资产莫名消失，建议选择国外厂商的服务，其他服务同理。这里我是用的Azure的北美主机。会自带一个公网IP。

![image-20200827173656015](C:UsersyaungzhouPicturesmarkdownResourceassetsimages\image-20200827173656015.png)

### 部署Blog

blog本质是一个静态站点，当前已经有很多静态站点生成器，github pages的自动生成站点是一个道理，其默认用的是Ruby开发的jekyll，除此之外，还有GO语言的HUGO，python下的 Pelican 和Node.JS下的 Metalsmith 等等。各生成器的具体用法很简单，有空我再补上。

另外纯知识分享的推荐用gitbook. 这里不做展开。

如果要正式使用了，请记得启动的时候配置监听为0.0.0.0：80端口。（443需要证书，另外下面dns配置可以直接实现https，因此没必要）

![image-20200827173813196-1598523128953](C:UsersyaungzhouPicturesmarkdownResourceassetsimages\image-20200827173813196-1598523128953-1598523164917.png)

这时可以通过IP地址来访问站点了。但是可用性很差，因为没有https，非域名的形式访问也会被很多浏览场景拒绝。

![image-20200827174300623](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827174300623.png)

### 内容建设

默认生成的博客很简陋，没有装饰，可以直接搜索各生成器的主题，下载下来后直接应用。各生成器的各个不同的模板有各自的规范，比如jekyll的 [Memoirs](https://www.wowthemes.net/memoirs-free-jekyll-theme/) 主题博文默认都放在_posts/下，且以标题中的时间来标记博文展示顺序。一些基本元素都在markdown的tag中声明。

![image-20200827175326873](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827175326873.png)

以上提到的几种生成器默认都是支持markdown格式的博文。

### 域名准备

除了购买域名，也可以选择用freenom.com下的为期一年免费的域名。可以直接用google账号登入，也可以不注册。![image-20200827173552532](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827173552532.png)

### DNS解析

freenom自己可以配置dns管理，但不是很建议，因为现在的域名基本都要求https，而freenom不提供该服务。可以在cloudflare.com上实现dns管理和强制https及其他加速，压缩和安全等基础免费套餐。注册好账号登入后，需要先绑定好该域名，然后配置好A记录，再到freenom下配置为自定义域名管理，填入自定义的dns域名。自此即可用域名实现对云主机的访问了。

![image-20200827173617959](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827173617959.png)

![image-20200827174127448](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827174127448.png)

### 与github pages绑定（非必须）

如果前面说的建站步骤是用github实现的，那么也可以用自己的域名cname到github站点。方法是dns配置一个cname，指向到github的默认域名。cname用的域名可以用三级或四级域名，在dns中直接配置即可。

![image-20200827173400053](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827173400053.png)

同时github的项目中，从setting进去将该域名写入到cname配置项。

![image-20200827173928975](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827173928975.png)

配置cname好后结果如下：

![image-20200827173850748](C:\Users\yaungzhou\AppData\Roaming\Typora\typora-user-images\image-20200827173850748.png)