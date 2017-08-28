# Github博客如何使用

* 网页地址

[申请网站地址](https://pages.github.com/)

* 参考网站

https://woaielf.github.io/

http://www.jianshu.com/p/1260517bbedb


> 大致步骤

1. 创建存储库
2. 克隆存储库
3. 创建索引文件
4. 提交并同步

详细参考如下网址，已有详细步骤
https://pages.github.com/

> 博客名称  ：lzx704/蓝海无涯

<!---->



## 1 注册网站/使用客户端
我们使用的是MAC,
[下载地址](https://desktop.github.com/)

[官网访问网址(需要翻墙)](https://github.com/)

2 创建
https://github.com/lzx704/lzx704.github.io
![点击图片即可](http://ovblf5i76.bkt.clouddn.com/2017-08-28-屏幕快照 2017-08-28 上午8.38.12.png)



## 问题
1 git push -u origin master 连不上

```
chenmodeMacBook-Pro:lzx704.github.io chenmo$ git push -u origin master
fatal: unable to access 'https://github.com/lzx704/lzx704.github.io.git/': Failed to connect to github.com port 443: Operation timed out

```

解决,在/etc/hosts 注释以下github.com前的IP

```
#192.30.252.129   github.com
#192.30.252.131   github.com
#204.13.251.16    github.com
```


