# 如何使用 SSH 密钥登录

## 1. 检查是否已有 SSH 密钥
在你的（**注意，不是在服务器上找，而是在自己的电脑上**）HOME 目录下寻找 `.ssh` 目录，如果其中包含除了 `known_hosts`、`config` 和 `authorized_keys` 之外的一对名为 `<名称>` 和 `<名称>.pub` 的文件，例如 `id_rsa` 和 `id_rsa.pub`，则说明已有 SSH 密钥，可跳过下一步。

```admonish info title="不同系统的 HOME 目录位置"
- macOS/Linux：`$HOME`
- Windows: `%USERPROFILE%`
```

## 2. 生成 SSH 密钥对
在本地终端中执行以下命令：

```bash
$ ssh-keygen
```

程序会交互式地要求输入文件名、密码等信息，直接按回车键即可使用默认值。可能的输出如下：

```
[root@host ~]$ ssh-keygen  <== 建立密钥对
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): <== 按 Enter
Created directory '/root/.ssh'.
Enter passphrase (empty for no passphrase): <== 输入密钥锁码，或直接按 Enter 留空
Enter same passphrase again: <== 再输入一遍密钥锁码
Your identification has been saved in /root/.ssh/id_rsa. <== 私钥
Your public key has been saved in /root/.ssh/id_rsa.pub. <== 公钥
The key fingerprint is:
0f:d3:e7:1a:1c:bd:5c:03:f1:19:f1:22:df:9b:cc:08 root@host
```

## 3. 将公钥上传到服务器
直接打开公钥文件（即以 `.pub` 结尾的文件）并复制其中的内容，内容一般形为：

```
ssh-rsa AAAAB3NzaC1yc2EA... myname@mypc
```

```admonish warning title="保护私钥！"
请不要复制私钥文件（即**不**以 `.pub` 结尾的文件），这实际上相当于你的密码。如果泄露，可能会导致他人能够登录你的账户。
```

然后在服务器上执行以下命令：
```bash
$ echo "<刚刚复制的公钥内容>" >> ~/.ssh/authorized_keys
```

```admonish warning
这里的 `>>` 是重定向操作符，表示将内容追加到文件末尾，而不是覆盖。

如果你已经有了 `authorized_keys` 文件，不要换成 `>`，否则会覆盖原有内容。
```

~~~admonish tip
如果报告找不到文件，请先创建 `~/.ssh/` 目录：
```bash
$ mkdir -p ~/.ssh
```
~~~

```admonish example
`authorized_keys` 的内容应该类似于你的公钥，每行一个。
```

为了确保连接成功，请在服务器上保证以下文件权限正确： 
```bash
$ chmod 600 ~/.ssh/authorized_keys
$ chmod 700 ~/.ssh
```



## 4. 结束

完成，你现在可以尝试在本地使用 SSH 密钥登录服务器了：

```bash
$ ssh <用户名>@<服务器地址>
```