---
title: COW：一个智能的HTTP代理服务器
date: 2016/09/30 15:51:10 
---

国庆将近，同志们只希望尽快为祖国庆生，以至于没心思工作（微笑）。

既然是这样就利用这个时间来记录一个最近才发现的一个小工具。

介绍小工具前先说说我的科学上网工具

## Shadowsocks

> shadowsocks 是一个可穿透防火墙的快速代理。通过客户端以指定的密码、加密方式和端口连接服务器，成功连接到服务器后，客户端在用户的电脑上构建一个本地 socks5 代理。使用时将流量分到本地 socks5 代理，客户端将自动加密并转发流量到服务器，服务器以同样的加密方式将流量回传给客户端，以此实现代理上网。

我现在是用 [shadowsocks](https://github.com/shadowsocks/shadowsocks/wiki/Shadowsocks-%E4%BD%BF%E7%94%A8%E8%AF%B4%E6%98%8E) 来科学上网的，ss 相比传统 VPN 来说好用不少，就是需要装客户端。ss 帐号可以网上买，也可以自己搭一个 ss 服务器，我自己就搭了一个。

装上 ss 客户端之后有两种上网模式，一个是自动代理模式，一个是全局模式，全局模式很容易理解，就是无论访问什么网站都会使用代理访问，而自动代理模式就是选择性使用代理，这个是比 VPN 爽的地方，因为 ss 已经自动给你生成了 pac 文件了，里面定义了一系列的访问规则。

<!--more-->

## Pac

Pac 叫 Proxy auto-config，是一个用来自动代理的技术，一个 PAC 文件包含一个 JavaScript 形式的函数，定义了一系列访问规则，用户代理根据这些规则适用一个特定的代理其或者直接访问。简单来说，就是指定一部分域名和相应的代理方式，于是我们可以定义一些被 GFW block 的域名来指定 http 代理或者 socks5 代理，而未被 block 的域名直接访问就可以了。这叫智能分流，比 VPN 的全局翻墙更智能快速。

## Shadowsocks 的限制

ss 虽然很好用，但是还有点小瑕疵，就是那张记录着被墙域名的表并不是很智能，有不少被弄到墙外的新成员还是不能及时记录到名单中，而且 Mac 上的 ss 客户端，本来是有自动获取新名单的功能，现在好像也不能用了。导致在自动代理模式下，遇到墙外新成员就要切换到全局模式，而全局模式访问国内网的时候也并不爽快，也是挺烦人的。

这时候 cow 出现了。

## COW

> COW 是一个简化穿墙的 HTTP 代理服务器。它能自动检测被墙网站，仅对这些网站使用二级代理。

也就是说 [COW (Climb Over the Wall) proxy](https://github.com/cyfdecyf/cow) 能自己学习！厉害了word哥！

我对于这些智能的工具真心没有抵抗力啊。

下面粘贴一下 cow 的‘学习方式’。

COW 在配置文件所在目录下的 stat json 文件中记录经常访问网站被墙和直连访问的次数。

- 对未知网站，先尝试直接连接，失败后使用二级代理重试请求，2 分钟后再尝试直接
	- 内置常见被墙网站，减少检测被墙所需时间（可手工添加）
- 直连访问成功一定次数后相应的 host 会添加到 PAC
- host 被墙一定次数后会直接用二级代理访问
	- 为避免误判，会以一定概率再次尝试直连访问
- host 若一段时间没有访问会自动被删除（避免 stat 文件无限增长）
- 内置网站列表和用户指定的网站不会出现在统计文件中

## COW 的使用

OS X, Linux (x86, ARM): 执行以下命令（也可用于更新）

```
curl -L git.io/cow | bash
```

安装时会询问安装的路径，我放在了默认用户路径上。

安装完成后开始配置 cow，配置文件在 `~/.cow/rc` 上，编辑器打开文件找到

```
listen = http://127.0.0.1:7777

```

这监听端口基本是不用改的，除非这个端口跟机子上某个服务端口重复了。

再往下找

```
# shadowsocks:
#   proxy = ss://encrypt_method:password@1.2.3.4:8388
#   proxy = ss://encrypt_method-auth:password@1.2.3.4:8388
#
#   encrypt_method 添加 -auth 启用 One Time Auth
#   authinfo 中指定加密方法和密码，所有支持的加密方法如下：
#     aes-128-cfb, aes-192-cfb, aes-256-cfb,
#     bf-cfb, cast5-cfb, des-cfb, rc4-md5,
#     chacha20, salsa20, rc4, table
#   推荐使用 aes-128-cfb
#
```

把 `proxy = ss://encrypt_method-auth:password@1.2.3.4:8388` 的注释开了，填上自己的加密方式，IP，端口，密码。

搞定保存。

接下来去配置一下本机的代理

1. 打开 `网络偏好设置` -> `高级` -> `代理`
2. 勾选自动代理设置
3. 右边填入 `http://127.0.0.1:7777/pac` (端口是配置文件配置的端口)


最后，开启 cow 服务

我根据网上教程启动 COW：Unix 系统在命令行上执行 `cow &` (若 COW 不在 PATH 所在目录，请执行 `./cow &`)。

我发现我把终端关闭后 COW 不能用了，原来 `&` 只是让命令运行在后台，但无法脱离终端运行。

这时候需要 `nohup` 来协助完成任务。

把运行命令改为 `nohup cow &`，额对了，如果这时候报找不到 `cow` 命令的错，其实这个文件安装在了刚才自己指定的路径上，像我就放在了路径`~`，那命令应该是 `nohup ~/cow &`


```
[1] 89183
appending output to nohup.out

```

搞定！

刚开始用，打开被墙网站可能有有点慢，因为学习机制问题，但后面会越用越爽，重要是，不用再去来回切换模式啦！(之前还想着写个 Alfred 插件去快捷切换模式，现在感觉好蠢)

## 参考资料

[COW (Climb Over the Wall) proxy](https://github.com/cyfdecyf/cow)

[关于PAC自动代理和ios翻墙](https://tyr.gift/pac-proxy.html)

[Linux 技巧：让进程在后台可靠运行的几种方法](https://www.ibm.com/developerworks/cn/linux/l-cn-nohup/)

[Linux命令之后台运行-nohup &](http://blog.csdn.net/wangjunjun2008/article/details/21983087)



以上



