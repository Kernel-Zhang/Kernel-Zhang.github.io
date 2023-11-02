---
title: 傻瓜式设置Nim服务器
author: kernel
date: 2023-06-08 20:34:00 +0800
categories: [Nim语言, 博客翻译]
tags: [Nim语言, 服务器开发]
---

Nim是服务器开发的理想选择，但如果你想运行自己的服务器，又是服务器管理和Linux领域的新手，那么要做好这一点，你需要了解的信息量可能会令人生畏。在本文中，我将简要介绍如何架设一个小型服务器，以安全的方式运行 Nim服务器。这篇文章比我的许多其他文章都要低级得多，其中包括大量的命令行片段和配置文件细节，所以请随时将它加入书签，在需要时再来阅读。我也会努力保持更新，并添加来自Nim社区的任何建议。本文要求你熟练掌握基本的Linux任务，如在shell中运行命令和编辑文件（使用Vim、Emacs、Nano等）。

首先要设置一台虚拟机或其他机器，然后安装一些Linux版本。本指南基于LTS版本的Ubuntu（本文撰写时为22.04），运行于最低层级的Linode（每月 5 美元，1GB 内存，共享CPU）。选择这台机器的原因很简单，因为我在写这篇文章的时候刚好在设置这台机器，但你也可以使用其他服务，比如Hetzner，在写这篇文章的时候，它能以更便宜的价格提供更多服务。如果你对Linux有所了解，可以随意选择任何发行版，但如果你是Linux新手，我建议你从Ubuntu LTS开始，因为所有这些命令都是为Ubuntu LTS编写的。

您可能还想为您的服务设置一个域名或子域名。只需在域名DNS配置中将服务器的IPv4和IPv6地址分别添加为A和 AAAA记录即可。每个注册商的做法都不尽相同，因此这里不涉及注册商的具体做法。请注意，您也可以在同一台机器上运行多个Nim服务器，如果是这种情况，请创建多个子域名，并将它们指向同一IP。

## 加固

服务器安装并启动后，最好设置防火墙。只需通过SSH登录服务器（Linode会以root用户身份登录，我们稍后会解决这个问题），运行`apt install nftables`，完成后再运行`systemctl start nftables && systemctl enable nftables`启动并使能服务。要配置它，你可以使用默认配置，只需将其放入类似`/etc/nftables/firewall.conf`的文件中，然后运行`nft -f /etc/nftables/firewall.conf`来启用它：

```conf
flush ruleset                                                                    
                                                                                 
table inet firewall {
        # 回复在ipv4和ipv6上的ping，但限制它们的速率
    chain inbound_ipv4 {
        icmp type echo-request limit rate 5/second accept
    }
    chain inbound_ipv6 {
        # 接受邻居发现，否则连接中断
        icmpv6 type { nd-neighbor-solicit, nd-router-advert, nd-neighbor-advert } accept
        icmpv6 type echo-request limit rate 5/second accept
    }
 
    chain inbound {
        # 默认情况下，丢弃所有不符合以下规则的流量
        type filter hook input priority 0; policy drop;
 
        # 允许已建立连接和相关连接的流量，但丢弃无效连接
        ct state vmap { established : accept, related : accept, invalid : drop }
 
        # 允许环回流量，即同一台机器上不同服务之间的流量
        iifname lo accept
 
        # 这将上述icmp/ping规则连接起来
        meta protocol vmap { ip : jump inbound_ipv4, ip6 : jump inbound_ipv6 }
 
        # 允许SSH使用TCP/22端口，允许HTTP(S) TCP/80和TCP/443用于IPv4和IPv6。
        tcp dport { 22, 80, 443} accept
    }
 
    chain forward {
        # 不允许转发（转发用于路由器之类的设备）
        type filter hook forward priority 0; policy drop;
    }
}
```

如果你已经在本地建立了服务器，并在其他端口（如5000或8080）上运行，请不要将该端口添加到配置中的允许端口列表中，我们将添加一个反向代理来处理这个问题。反向代理将通过80或433端口与更广泛的互联网对话，然后通过环回接口与我们的服务器进行对话（可以看到我们在上文允许了环回接口）。

以根用户身份登录后，你还可以下载Nginx和“Let's Encrypt”的certbot，以及将二者集成的脚本：`apt install nginx certbot python3-certbot-nginx`。Certbot是我们用来在服务器上启用HTTPS并更新密钥的工具。它会自动获取密钥，并重新配置Nginx以使用它们。如果你知道自己在做什么，并希望获得更多控制权，可以使用`dehydrated`软件包，但需要手动设置更多配置文件。

现在，让我们为服务器创建一个用户，并禁止以root用户身份登录。首先，我们需要添加一个有权限提升为root的用户。

```shell
useradd -m -s /bin/bash server # 创建名为server的用户，该用户拥有一个主目录和bash登录shell
passwd server # 为我们的用户创建密码，运行此命令时系统会提示您输入密码
usermod -a -G sudo server # 将服务器用户添加到sudo组
```

为了让`usermod`步骤允许我们使用`sudo`命令，必须在`/etc/sudoers`文件中加入这样一行。建议使用`visudo`执行此操作，因为弄乱该文件会导致系统锁定。在这种情况下，由于是全新安装，所以如果你不喜欢Vim并想用其他编辑器进行编辑，可以冒这个险：

```
%sudo ALL=(ALL:ALL) ALL
```

Ubuntu LTS应该已经有了这个功能，但如果你没有，或者有类似的一行，但用`%wheel`代替，那么就把它加上去，或者把组名改成`%`后面的名字。如果名称前没有百分号，那就是用户而不是组，如果没有`ALL:ALL`部分，你可能就不必使用sudo来执行root操作（这很糟糕）。

现在用`su server`登录新用户，检查是否无法运行`cat /etc/sudoers`等命令，但可以用`sudo cat /etc/sudoers` 访问。还要确保可以通过SSH直接进入`server`用户（运行类似`ssh server@myserver.com`的命令）。如果一切正常，我们就可以禁止root用户登录。只需打开文件`/etc/ssh/sshd_config`，并更改以下一行：

```
PermitRootLogin yes
```

至：

```
PermitRootLogin no
```

然后使用`systemctl restart sshd` 重启SSH守护进程服务，这将禁止root用户直接登录，而必须通过我们的限制访问用户登录。

不过，在SSH上使用密码登录并不是最好的安全方式，因此建议使用SSH密钥登录。如果你还没有，可以使用`ssh-keygen`命令生成一个（在你的普通机器上，而不是服务器上）。SSH密钥由两部分组成，即私钥和公钥。我们将公钥部分上传到服务器，并确保私钥部分的安全。这样，服务器就可以使用公钥部分解密私钥部分加密的内容，反之亦然。我建议你在生成SSH密钥时为其设置密码，但如果你知道没有其他人可以访问私钥文件，则没有必要严格设置密码。

使用`ssh-copy-id server@myserver.com`或手动将公钥复制到`.ssh/authorized_keys`文件夹并设置适当的权限，将公钥上传到服务器。私钥需要放在`ssh`命令能找到的地方。在Linux上，`ssh-keygen`默认会将新生成的密钥放置在这样的地方，但在其他操作系统上可能并非如此。

安装好密钥后，使用密钥文件验证是否可以SSH登录。如果文件放在SSH能够找到的地方（如Linux上的`~/.ssh/id_rsa`），就可以直接运行常规的`ssh`命令。否则，你可以通过运行以下命令手动传递密钥：

```shell
ssh -i <path to keyfile> server@myserver.com
```

系统会要求你输入SSH密钥的密码（不是我们之前创建的用户密码），输入后你就可以登录服务器了。确定成功后，打开`/etc/ssh/sshd_config`并设置这一行：

```
PasswordAuthentication no
```

然后用以下命令重新启动SSH守护进程：

```shell
sudo systemctl restart ssh
```

完成此操作后，没有私钥就无法登录服务器，因此请务必将私钥备份到安全的地方！

现在，我们已经完成了一些基本的服务器加固，让我们来设置实际的网络服务器！

## 反向代理和HTTPS

Nim网络服务器通常依赖于格式良好的HTTP流量，为了让HTTPS正常工作，我们需要处理证书。为了稍稍保护自己免受网络不良分子的侵害，并且不必在Nim代码中实施HTTPS和多域路由，我们将使用反向代理。这基本上只是一个网络服务器，其唯一的工作就是验证信息、处理HTTPS事宜并将流量传递给我们。在本文中，我们将使用Nginx，但你也可以使用Apache这样的服务器。

要设置Nginx（我们之前已经安装过），首先使用`unlink /etc/nginx/sites-enabled/default`禁用默认站点，然后在`/etc/nginx/conf.d/myserver.com.conf`中打开配置文件（用服务器的（子）域名替换`myserver.com`部分）。在本例中，我们将使用Nim服务器来提供静态文件，因此一切都将通过Nim服务器进行。这将托管我们的配置，配置相当简单：

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name myserver.com www.myserver.com; # 服务器域名列表
 
    location / {
        proxy_pass http://localhost:8080; # 将所有流量传输到我们服务器上的8080端口
        proxy_send_timeout 600;
        proxy_set_header Host $host; # 添加一些额外的标头，以备不时之需
        proxy_set_header X-Real_IP $remote_addr;
    }
}
```

如果你在同一台机器上设置了多个Nim服务器，并想用不同的（子）域名指向它们，请注意你只能将其中一个标记为`default_server`。只需将上述配置中的`default_server`部分删除即可。要为每个文件设置（子）域名，请将文件名和`server_name`项改为正确的域名。如果想使用Nginx为静态文件提供服务（这很可能会提高性能），可以在上述配置的`服务器`块中添加这样一个块：

```
location /assets/ {
    root /www/assets
}
```

这将从服务器上的`/www/assets`文件夹为`myserver.com/assets`提供服务。如果您将文件放在这里，但它们无法访问，您可能需要用`chmod o+r`更改文件的权限，以使`其他`类别的用户可以读取它们。

保存配置文件后，运行`nginx -t && nginx -s reload`重启服务器。现在，它将把`myserver.com`的80端口代理到本地计算机的8080端口（除非你在上述配置中更改了其他端口号）。但它仍然只能运行 HTTP，要想运行 HTTPS，请运行`certbot --nginx -d myserver.com -d www.myserver.com`创建证书并更新配置文件以重定向到HTTPS。完成后，你就能在刚创建的配置文件中看到certbot添加了各种证书指令，并注册了所有需要的证书。对每个（子）域都执行这一步骤。这些证书需要不时刷新，因此你可能还需要在crontab中添加类似的内容（使用`crontab -e`）：

```
0 12 * * * /usr/bin/certbot renew --quiet
```

此条目每天中午运行一次检查，以确保证书是最新的。此外，使用`systemctl status cron` 或`systemctl enable cron && systemctl start cron`启用并使能cron服务，确保cron服务正在运行。

## 构建我们的Nim服务器

现在，Nginx已设置为只允许域上的HTTPS请求，并将所有请求传递到`localhost:8080`。现在是设置实际Nim服务器的时候了！你可能已经有了一个想要部署的Nim服务器程序，所以我不会在这里详细介绍如何编写Nim服务器。如果你还没有编写服务器程序，可以使用Jester、HappyX、Prologue或其他[HTTP 服务器](https://nimble.directory/search?query=http+server)快速入门。你也可以看看[Federico的项目制作工具](https://github.com/FedericoCeratto/nim_project_maker)，它能将一切设置为合适的`.deb`包。

如果你已经准备好了代码，但还没有一个合适的软件包，我们就需要根据我们的架构和操作系统来构建它。如果你在本地运行一台Linux机器，你或许可以直接用`scp mynimserver server@myserver.com:mynimserver` 将二进制文件复制到服务器上，否则你就必须交叉编译，或者直接在服务器上安装Nim并在那里构建网站。要安装Nim，我建议使用choosenim：

```
curl https://nim-lang.org/choosenim/init.sh -sSf | sh
```

该命令完成后，你的服务器上就安装了最新的稳定版Nim。如果以后想更新，只需运行`choosenim update stable` 即可。如果你已经在Git仓库中将项目设置为Nimble软件包，现在最简单的办法就是运行`git clone <repo URL>`，然后运行`nimble build -d:release` 即可。为了确保你的Nim服务器能在机器启动时运行，并在因某种原因失败时重启，我们可以为它添加一个服务文件。如果你的用户名为`server`，你的项目名为`mynimserver`，并且克隆到了主目录（当你SSH登录服务器时，你的主目录就在其中），这样的服务文件看起来就像下面这样

```
[Unit]
Description=Our awesome Nim server

[Service]
User=server
WorkingDirectory=/home/server/mynimserver
ExecStart=/home/server/mynimserver/mynimserver
Restart=always
RestartSec=5
 
# Sandboxing features
PrivateTmp=yes
NoNewPrivileges=yes
ProtectSystem=strict
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_DAC_READ_SEARCH
RestrictNamespaces=uts ipc pid user cgroup
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6 AF_NETLINK
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
PrivateDevices=yes
RestrictSUIDSGID=yes
IPAddressAllow=192.168.1.0/24
MemoryDenyWriteExecute=yes
LockPersonality=yes

[Install]
WantedBy=default.target
```

它将工作目录设置为`mynimserver`文件夹，这可能是你的二进制文件希望找到要提供服务的静态文件的地方，并在薄文件夹内运行`mynimserver`二进制文件。它还包含一些沙盒功能，如果你无意中增加了外部用户在你的程序中执行程序的可能性，这些功能可以帮助保护你的服务器。这里的沙箱功能对你想做的大多数事情都没有问题，但请记住，如果你要做一些更深奥的事情，你可能需要禁用其中的一些功能。如果你想让程序更加安全，你还可以添加更多的功能（参见`systemd-analyze security mynimserver` 的输出结果），但这是一个无底洞，对本文来说太深了。

要安装此服务，只需将此文件复制到`/etc/systemd/system/mynimserver.service`并运行`systemctl daemon-reload`和 `systemctl enable mynimserver && systemctl start mynimserver`。如果一切按计划进行，你的网站现在应该可以正常运行了！只需将浏览器指向`https://myserver.com`（或您的任何域名），就能欣赏到您网站的绚丽风采！

更新 13/06/2023：根据Federico Ceratto和Stefan Salewski的反馈意见改进指南