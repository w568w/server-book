# 代理配置指南

由于众所周知的原因，许多服务（例如 Hugging Face Hub）在国内无法直接访问。为了解决这个问题，我们提供了一个简明的代理配置指南。

## 1. 启动代理服务
推荐使用 [clash](https://github.com/Dreamacro/clash) 管理代理。clash 是一个开源的代理软件，支持多种代理协议，例如 SOCKS5、HTTP、Shadowsocks、VMess 等。

许多代理提供者也提供了 clash 配置文件（或 URL），便于直接导入使用。下面假定你已经有了一个可用的 clash 配置文件（通常是一个 `.yaml` 文件）。

### 获取配置文件
首先需要将文件导入到服务器上，下面分三种情况：

#### a. 从 URL 导入
如果你的配置文件是一个 URL，可以在服务器上直接使用 `wget` 下载：

```bash
$ wget https://example.com/config.yaml
```

#### b. 从本地导入，使用 scp
如果你的配置文件在本地，可以在本地使用 `scp` 命令将其上传到服务器上：

```bash
$ scp /path/to/config.yaml <用户名>@<主机>:/path/to/config.yaml
```

其中 `<用户名>` 和 `<主机>` 分别是你的服务器的用户名和主机名，例如 `root@123.456.789.0`。

#### c. 从本地导入，使用 Visual Studio Code
如果你使用的是 Visual Studio Code 连接到服务器，也可以在 VSCode 的资源管理器中直接将文件从本地复制粘贴到服务器上。

### 检阅配置文件
在导入配置文件之后，你可以使用 `cat` 命令检阅一下配置文件的内容：

```bash
$ cat /path/to/config.yaml
```

~~~admonish tip
需要注意的是前几行的内容，尤其是端口号，通常形如：

```yaml
mixed-port: 7890
external-controller: 0.0.0.0:9090
```

通常的默认值如上，但如果你在后续启动服务时发现报错「端口冲突」，应该同时将两者的端口号都改为其他值，直到不再报错为止。
~~~

### 启动服务
```admonish tip
由于服务需要持续运行，建议用一个 tmux 会话来运行以下命令。

tmux 的使用方法见[这里](checklist.md#可以使用-tmux-管理长时间运行实验的会话)。
```

启动服务非常简单，只需要在服务器上执行：

```bash
$ clash -f /path/to/config.yaml
```

其中 `/path/to/config.yaml` 是你的 clash 配置文件的实际路径，推荐将其放置在 HOME 目录下，例如 `~/.config/clash/config.yaml`。

如果报错，请回到上一步检查配置文件的内容。

## 2. 配置代理
启动 clash 代理后，需要配置应用程序或 Python 代码使用代理。

目前大部分 Linux 网络请求库、框架和应用程序都遵循同一惯例，即：读取 `http_proxy` 和 `https_proxy` 环境变量的值来作为代理，Python 也不例外。因此，只需要在启动应用程序之前设置这两个环境变量即可。

```admonish info
[这篇文章](https://about.gitlab.com/blog/2021/01/27/we-need-to-talk-no-proxy/)和[这篇问答](https://superuser.com/questions/944958/are-http-proxy-https-proxy-and-no-proxy-environment-variables-standard)讨论了这个惯例的由来和标准化情况。
```

有[很多种方法配置环境变量](https://wiki.archlinux.org/title/Environment_variables)，这里我们只介绍三种：

### a. 临时设置
在启动应用程序之前，可以在命令行中设置环境变量：

```bash
$ export http_proxy=http://127.0.0.1:7890
$ export https_proxy=http://127.0.0.1:7890
# 接下来的命令都会使用代理
$ python my_lab.py
```

这样设置仅会在当前 Shell 会话中接下来的命令生效，新开窗口或者重启后都会失效。

或者可以在启动应用程序的命令前加上环境变量：

```bash
$ https_proxy=http://127.0.0.1:7890 http_proxy=http://127.0.0.1:7890 python my_lab.py
```

这样设置仅对本条命令生效。

### b. 永久设置
如果你希望永久设置这两个环境变量，可以将其写入你的 Shell 配置文件中。

```admonish info title="不同 Shell 配置文件的路径"
- bash: `~/.bashrc`
- zsh: `~/.zshrc`
```

在文件尾部添加：

```sh
# >>> Proxy settings >>>
export http_proxy=http://127.0.0.1:7890
export https_proxy=$http_proxy
# <<< Proxy settings <<<
```

然后重启 Shell。

```admonish warning
一些远古和错误的教程会建议将这些环境变量写入下面的位置之一，对于普通用户来说，这些全部是不正确的配置方法：

- `~/.bash_profile`
- `~/.profile`
- `/etc/environment`
- `/etc/profile`
- `/etc/profile.d/xxx.sh`
```

### c. 在 Python 方面设置
也可以仅在特定的 Python 代码中设置环境变量，这样不会影响其他程序。

```python
# 在文件/笔记本的最开头执行：
import os
os.environ['http_proxy'] = 'http://127.0.0.1:7890'
os.environ['https_proxy'] = os.environ['http_proxy']
# ...余下的其他 import...
```
