# 如何从公共账户迁移到个人账户

如果你已经在使用公共账户了，你可能希望在创建新账户后能继续原来的工作流，这里是一份简明（但不完全的）教程。

```admonish note
步骤看起来很多，但实际上每一步只有一两条命令。Take it easy! 🤗
```

~~~admonish tip
开始之前，可确定当前账户拥有以 root 身份执行命令的权限：

```shell
$ sudo echo hello
```

如果能够正常执行，应该会提示输入密码（如果刚刚输入过，则不会提示）。若输出 `hello`，则说明当前账户拥有以 root 身份执行命令的权限。
~~~


```admonish warning
本文中，`#` 开头的命令**必须**以 root 身份执行（即：用 `sudo` 运行），而 `$` 开头的命令则**必须不**以 root 身份执行。

`<>` 中的内容需要被替换为实际内容，而不是按照尖括号原样输入，例如 `ls <你的 HOME 目录>` 可能替换成 `ls /home/myname`。
```

## 1. 创建新账户
用以下命令创建一个新账户：

```shell
# useradd --home-dir <新账户的 HOME 目录位置，如 /data/myname 或 /home/myname> \
    --comment "<任意备注文字，例如你的真实姓名全称>" \
    --create-home \
    --shell <你偏好的默认 Shell，例如 /bin/bash 或 /bin/zsh> \
    <账户名>
```

~~~admonish warning
请不要在 `--home-dir` 处填入已有的目录，例如你原来的实验代码目录。这样做可能会导致权限错误。

如果一定要用已有的路径，可将原来的路径重命名（假设原来的代码目录为 `/data/myname`）：

```shell
# mv /data/myname /data/myname.bak
```
~~~

## 2. 设置密码
用以下命令设置新账户的密码。创建密码时请遵循基本安全原则。

```shell
# passwd <账户名>
```

## 3. 设置用户组
如果你不需要在新账户中使用 root 权限，可跳过本步骤。

如果你需要在新账户中使用 root 权限，可用以下命令将新账户加入具有 sudo 权限的相关用户组：

```shell
# usermod --append --groups <用户组名> <账户名>
```

~~~admonish tip title="如何确定「具有 sudo 权限的相关用户组」的名称？"
尽管**以下这些名称可能在网上的不靠谱教程中很常见，但盲目地应用它们都是错误的**，因为在不同发行版和服务器上用户组的配置很有可能完全不同：

- `sudo`
- `wheel`
- `admin`
- `adm`
- `root`
- `rootadm`
- ...

正确的做法是，先查看 `/etc/sudoers` 文件中的内容：

```shell
# cat /etc/sudoers
```

其中可能包含这样的行：

```
%abc ALL=(ALL) ALL
```

或者：

```
%abc ALL=(ALL:ALL) ALL
```

它们的意思是：组 `abc` 的成员，可以在任何位置（`ALL=`）以任何用户身份（`(ALL)`）执行任何命令（`ALL`）。

那么组 `abc` 就是你要找的一个「具有 sudo 权限的用户组」。

如果这样的组有多个，优先考虑听起来名称更加合理的组，例如 `wheel`、`sudo` 等。
~~~

## 4. 停止原程序
如果你此前在公共账户中运行了程序，请自行检查并停止这些程序。

容易忽略的点包括：

- `tmux` 或 `screen` 中运行的程序
- `nohup` 或 `&` 后台运行的程序
- `vscode-server` 进程

## 5. 切换到新账户
用以下命令切换到新账户：

```shell
如果你想进入新账户，但停在当前目录中：
$ su <账户名>
如果你想进入新账户，并切换到新账户的 HOME 目录：
$ su - <账户名>
```

## 6. 迁移数据

```dmonish tip
以下是对不同种类的数据和配置的迁移方法的简要介绍。

与其他节不同的是，**你不一定要按列出的顺序挨个执行！**
```

### Conda 支持
如果你此前使用 conda 管理环境，可用以下命令在新账户中初始化 conda 支持：

```shell
$ conda init
```

~~~admonish tip
如果报错找不到 conda，说明此前的 conda 安装不在新账户的 PATH 中，需要寻找原先的 conda 程序的位置。

输入 `exit` 回到**原先的账户**下，执行：
```shell
$ which conda
```

输出的路径就是 conda 程序的位置。再次到**新账户**中，用以下命令替代上面的 `$ conda init`：

```shell
$ <conda 程序的位置> init
```
~~~

用 `$ exit` 退出新账户，然后重新进入新账户，输入 `$ conda info` 确认 conda 是否正常工作。

### Conda 环境迁移

如果你此前已经有环境（如果没有，可跳过本步骤），可能在使用时发现出现了一大堆权限问题：

```shell
(my_env) myname@server:~$ pip install test
Defaulting to user installation because normal site-packages is not writeable
...
```

这是因为原先的安装在公共账户下，而新账户没有权限修改。解决方法是，将原先的环境迁移到新账户下。

遗憾的是，conda 并没有一个适用于所有情况的迁移环境方法，因此你不得不从下面几个迁移方法中做出选择（按推荐程度排序）：

```admonish info
更多方法可参考 [Moving Conda Environments](https://www.anaconda.com/blog/moving-conda-environments)。
```


#### a. `requirements.txt` 文件
如果你有在原环境中维护一个 `requirements.txt` 文件的良好习惯，可用以下命令创建新环境：

```shell
$ conda create -n <新环境名> python=<原环境中的 Python 版本>
$ conda activate <新环境名>
$ pip install -r <原环境中的 requirements.txt 位置>
```

#### b. `environment.yml` 文件
切换到原来的 conda 环境，用以下命令导出环境：

```shell
$ conda env export > environment.yml
```

然后用以下命令创建新环境：

```shell
$ conda create -n <新环境名> -f environment.yml
```

#### c. `--clone` 参数
用以下命令创建新环境：

```shell
$ conda create -n <新环境名> --clone <原环境名>
```

---

迁移完成后，用 `$ exit` 回到**原公共账户**，用以下命令删除原环境：

```shell
$ conda env remove -n <原环境名>
```

### 代码和数据迁移
如果你的代码和数据都在公共账户的 HOME 目录下，可用以下命令在 **原公共账户** 下将它们迁移到新账户的 HOME 目录下：

```shell
$ cp -r <原来的代码位置> <新账户的 HOME 目录位置>
$ rm -rf <原来的代码位置>
```

如果你的代码和数据不在公共账户的 HOME 目录下（例如，在 `/data` 目录下），更简单的方法是更换原来的代码和数据的所有者：

```shell
# chown -R <新账户名>:<新账户名> <原来的代码位置>
```

有时你的数据集可能缓存在 `$HOME/.cache` 目录下（例如，用 Huggingface Hub 下载的代码）。这部分的迁移比较复杂，友情建议是重新下载数据集（也就是什么都不管，直接在新账户下跑原来的代码。由于 `$HOME` 已经发生了变化，所以程序理应会找不到原来的缓存，继而重新下载数据集）。

## 6. 配置 SSH 登录
你应该在新账户中使用 SSH 登录。参考[如何使用 SSH 密钥登录](ssh.md)。

---
恭喜完成！你现在可以用新账户登录服务器了。

```admonish note
**全部完成后，请丢弃原来的公共账户登录方式，包括密钥、密码等。** 

这是安全的，因为你的新账户也具有 sudo 权限。
```
