

> 记录GitHub使用过程中的事情



| 版本号 | 作者                         | 修改时间   | 修改说明 |
| ------ | :--------------------------- | ---------- | -------- |
| v1.0   | Joey zhang 2298667492@qq.com | 2021-11-26 | 初始版本 |



## 问题

### 问题1 - ssh key 添加失败提示 Key already in use

场景：我有两个 github 账号，一台 pc ，新的账号需要添加 ssh pub key 时提示 Key already in use 错误。

[官网对该错误的说明](https://docs.github.com/en/authentication/troubleshooting-ssh/error-key-already-in-use)

测试了一下发现，当前的 key 正在被旧的账号使用。

```shell
zy@zy:local_test$ ssh -T -ai ~/.ssh/id_rsa git@github.com
Hi pidansolo! You've successfully authenticated, but GitHub does not provide shell access.
```

所以解决方法应该是新建一个 key 对来给新账号使用。

1、先按照 github 官网上的教程新加一个 [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)

```shell
zy@zy:sshkey$ ssh-keygen -t rsa -b 4096 -C "2298667492@qq.com"
Generating public/private rsa key pair.
Enter file in which to save the key (/home/zy/.ssh/id_rsa): ./joey_github_sshkey

zy@zy:sshkey$ ls -lt
总用量 8
-rw------- 1 zy zy 3243 11月 26 14:42 joey_github_sshkey
-rw-r--r-- 1 zy zy  743 11月 26 14:42 joey_github_sshkey.pub
```

2、修改配置文件 /etc/ssh/ssh_config 配置文件，如下

```
Host pidansolo.github.com
	User git
	Hostname github.com
	IdentityFile /home/zy/.ssh/id_rsa

Host Joey-Zhang-0A0.github.com
	User git
	Hostname github.com
	IdentityFile /media/zy/gs_work/myblog/joey_github/sshkey/joey_github_sshkey
```

3、重启 ssh 服务，并加上新的 key 

```shell
sudo service ssh restart

# 这一步非常关键，如果不执行则无法生效。
ssh-add /media/zy/gs_work/myblog/joey_github/sshkey/joey_github_sshkey

zy@zy:myblog$ ssh-add -l
4096 SHA256:21DdQf846qAjeiN8pKEYIME5iHqqkGUTFH8dDpK2oKg /media/zy/gs_work/myblog/joey_github/sshkey/joey_github_sshkey (RSA)
2048 SHA256:WeTW/jgybwEhCu4tXrpmNOJOa9yh6WyPurWmnHczKy4 zy@zy (RSA)
```

4、尝试 ssh 连接是否正常。

```shell
zy@zy:myblog$ ssh -T Joey-Zhang-0A0.github.com
Warning: the ECDSA host key for 'github.com' differs from the key for the IP address '192.30.255.112'
Offending key for IP in /home/zy/.ssh/known_hosts:106
Matching host key in /home/zy/.ssh/known_hosts:199
Hi Joey-Zhang-0A0! You've successfully authenticated, but GitHub does not provide shell access.

zy@zy:myblog$ ssh -T pidansolo.github.com
Warning: the ECDSA host key for 'github.com' differs from the key for the IP address '192.30.255.112'
Offending key for IP in /home/zy/.ssh/known_hosts:106
Matching host key in /home/zy/.ssh/known_hosts:199
Hi pidansolo! You've successfully authenticated, but GitHub does not provide shell access.
```

5、连接正常后基本就可以用了。如果不能用可能和仓库创建有关，github添加新的ssh key后再创建仓库即可。



### 问题2 - 下载生成补丁文件

直接在commit 链接后面加上.patch 即可，如下。

```url
https://github.com/bluekitchen/btstack/commit/0f2a810173d1d70943d1c915bffd6f9b1171e8f6.patch
```





## 资料

感谢以下链接对本文章的帮助！
