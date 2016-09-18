---
title: PHP开发：Mac 下 PHPStorm + XDebug 断点调试
date: 2016/09/17 23:00:48
---

最近不仅换厂了，连开发方向都换了，从事PHP开发去了。

个中原因一言难尽，希望在不久将来能成为全栈吧！

接下来的博客内容可能会以web开发为主了，如果有时间研究iOS也是极好的。

这篇东西主要想记录一下我在配置PHP断点调试遇到的一些坑（仅仅记录我遇到的）。

## 安装XDebug

获取相应版本XDebug步骤:

1. 复制phpinfo()打印出来的页面(`command+a`, `command+c`);
2. 浏览器打开网址[https://xdebug.org/wizard.php](https://xdebug.org/wizard.php)
3. 把刚才复制的phpinfo内容粘贴到页面的输入框中(`command+v`)，然后点击下方`Analyse my phpinfo() output`按钮;

接下来会得到以下页面（不同机子，不同版本可能得到的内容不一样，仅记录我获得的）

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-3E683CD6-B144-4D83-8A41-5A41F0B12E68.png)

页面上显示了对本机配置的描述`SUMMARY`，还有针对此配置的安装`XDebug`的详细步骤`INSTRUCTIONS`（巨详细，但是有几个坑）。

按照给出的步骤一步一步往下走，一个一个坑的踩。

<!--more-->

### 坑1:

第二步，解压缩的时候，按照上面的指令去解压，不要在图形界面上解压。

### 坑2:

第四部，运行命令`phpize`的时候报错如下:

```
/usr/local/Cellar/php70/7.0.10_1/bin/phpize: line 61: /usr/local/Library/Homebrew/shims/super/sed: No such file or directory
/usr/local/Cellar/php70/7.0.10_1/bin/phpize: line 62: /usr/local/Library/Homebrew/shims/super/sed: No such file or directory
/usr/local/Cellar/php70/7.0.10_1/bin/phpize: line 63: /usr/local/Library/Homebrew/shims/super/sed: No such file or directory
Configuring for:
PHP Api Version:
Zend Module Api No:
Zend Extension Api No:
/usr/local/Cellar/php70/7.0.10_1/bin/phpize: line 155: /usr/local/Library/Homebrew/shims/super/sed: No such file or directory
Cannot find autoconf. Please check your autoconf installation and the
$PHP_AUTOCONF environment variable. Then, rerun this script.
```

报错主要说明了两个问题，一个是在指定路径上`sed`命令没找到，另一个是`Cannot find autoconf`。

填坑:

1. 发现`sed`在其他路径上，把找到的`sed`文件复制到指定路径上，也就是上面的`/usr/local/Library/Homebrew/shims/super/sed`。
2. 缺少`autoconf`，缺少那就安装，使用`Homebrew`安装，在命令行键入`brew install autoconf`

### 坑3:

第七步与第八步路径不一样，或者说第七步路径缺少了一部分！！

填坑：

把第七步命令上的路径按照第八步的路径补全

`cp modules/xdebug.so /usr/local/Cellar/php70/7.0.10_1/lib/php/extensions/no-debug-non-zts-20151012/xdebug.so`

## 配置XDebug

在`php.ini`文件末尾添加如下代码:

```
zend_extension = /usr/local/Cellar/php70/7.0.10_1/lib/php/extensions/no-debug-non-zts-20151012/xdebug.so
xdebug.idekey=PhpStorm
xdebug.remote_enable = On
xdebug.remote_host=localhost
xdebug.remote_port=9001
xdebug.remote_mode=req
xdebug.remote_handler=dbgp
```

由于端口`9000`被占用了，我改用了`9001`。

什么？找不到`php.ini`文件?

好吧，再次打开`phpinfo`

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-DingTalk20160918180407.png)

配置完成，重启webserver，php-fpm。

## 检验安装配置是否成功

重新打开`phpinfo`页面，看到下图这个，说明安装成功

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-QQ20160918-0.png)

或者命令行键入`php -m`，拉到最后看是否有`XDebug`

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-QQ20160918-1.png)

## PHPStorm中配置XDebug

### 配置端口

里面的`debug port`必须要与刚才`php.ini`配置的`remote_port`保持一致。

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%886.21.59.png)

### 配置servers

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%886.27.07.png)

### 配置debug

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%886.30.31.png)

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%887.07.06.png)

配置好后，点击 ![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%886.32.55.png) 这个按钮，开始debug，别忘了打断点哦！

![](http://oaayo42x2.bkt.clouddn.com/2016-09-18-%E5%B1%8F%E5%B9%95%E5%BF%AB%E7%85%A7%202016-09-18%20%E4%B8%8B%E5%8D%886.35.59.png)

可以断点了，大功告成！

以上。
