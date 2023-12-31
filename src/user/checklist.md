# 自查清单

本页面给出了在使用服务器过程中建议遵循的最佳实践。这些最佳实践可以帮助你更好地使用服务器，也可以保证服务器的安全和可持续运行。

规则分为三个等级：

- **必须**：必须遵守的规则，违反规则可能导致服务器故障、数据丢失等严重后果；
- **建议**：建议遵守的规则，违反规则可能导致管理混乱等后果；
- **可以**：可选遵守的规则，违反规则可能会影响使用体验。

## **必须**使用个人 Linux 账户。
必须使用个人 Linux 账户，不得共用账户或使用 root（包括管理员账户，如 dell、admin 等）账户。

**原因**

不同用户的权限不同，共用账户会导致权限混乱，不利于追踪和管理。

由于每个账户仅有一个 HOME 目录，不同用户的文件和配置（例如 Shell、编辑器、tmux 终端等）会相互影响，造成使用体验不佳，也不方便管理员进行维护（例如备份、迁移用户数据等）。

root 账户拥有最高权限，使用 root 账户可导致误操作（例如不小心删除全盘），造成严重后果。

**对策**

向管理员请求创建个人账户。

如果要自己创建，可参考[如何从公共账户迁移到个人账户](migration_from_public.md)。

## **必须**使用 SSH 密钥登录。

**原因**

SSH 密钥登录比密码登录更安全（理论上无法被暴力破解），且更方便（每次登录无需输入密码）。

**对策**

参考[如何使用 SSH 密钥登录](ssh.md)。

## **必须**使用安全的密码。

**原因**

即便使用了 SSH 密钥登录，使用弱密码（例如 123456、password、abc123 等）仍可能会导致账户的 sudoer 权限被他人获取，造成严重后果。

**对策**

使用强密码，密码长度不少于 8 位，且包含大小写字母、数字和特殊字符。此外，还需要妥善保管密码，不得将密码泄露给他人。

## **必须**禁止直接编辑无权限编辑的文件。

禁止使用 `sudo` 命令编辑无权限编辑的文件。

例如（下面是**错误示例**）：

```bash
$ sudo vim /etc/sudoers
$ sudo nano /etc/hosts
```

**原因**

一些文件（例如 `/etc/` 目录下的文件）仅 root 用户有权限编辑，或者需要特殊的权限配置，直接编辑可能会破坏文件的权限设置，造成无法预料的后果。

**对策**

在终端配置文件（如 `.bashrc`、`.zshrc`）中设置默认编辑器，如：

```bash
export EDITOR=nano
```

然后依据以下原则修改：

- **通常建议不碰无权限的文件，除非你非常清楚自己在做什么！有问题建议优先联系管理员**；
- 对于 `/etc/sudoers` 文件，使用 `sudo visudo` 命令编辑；
```admonish error title="/etc/sudoers 警告"
任何情况下禁止使用其他任何方式编辑 `/etc/sudoers` 文件或者改变其权限、属性，这可能会直接破坏 `sudo` 的功能！
```

- 对于其他文件，使用 `sudoedit` 命令编辑：
    ```bash
    $ sudoedit /etc/hosts
    ```

## **建议**不使用 root 权限执行命令。

**原因**

root 权限拥有最高权限，使用 root 权限可导致误操作，造成严重后果。

另外，需要 root 权限才能修改的文件通常是全局生效的（例如 `/etc/` 目录下的文件），贸然修改可能会影响其他用户的使用。

> 这类问题在配置镜像源时尤其常见，也是弄乱很多服务器的主要原因之一。

**对策**

尽可能不使用和 root 权限相关的命令，例如 `sudo`。如果一定要使用，**务必确认自己知道自己在做什么**。


~~~admonish error title="重要提醒"
**严禁从网上复制粘贴不了解的、需要 `sudo` 才能执行的命令**，这些命令可能会造成严重后果！

严禁私自用以下命令之一改变服务器的电源状态：

- `shutdown`
- `reboot`
- `poweroff`
- `halt`
- `init`
- `systemctl poweroff`
- `systemctl reboot`
- `systemctl halt`
- `systemctl suspend`
- `systemctl hibernate`
- `systemctl hybrid-sleep`
~~~

有的教程习惯使用 `$ sudo kill`，容易给人造成「`kill` 必须使用 `sudo`」的错觉，实际上 `kill` 自己的进程，不需要 `sudo`。**仅当你确定进程无法被普通用户权限杀死时，才需要使用 `$ sudo kill`**。

有关配置镜像源等问题，可以让管理员协助。

## **建议**使用 conda 管理 Python 环境。

**原因**

conda 是一个开源的软件包管理系统和环境管理系统，可以用于安装 Python 环境和第三方库。

系统的 Python 解释器已经被锁定在固定的旧版本，仅作为一些系统工具和服务的正常运行所必需的运行库而存在，不适合实验和开发。使用 conda 可以方便地安装自己需要的 Python 版本。

**对策**

在进行实验时，使用 conda 创建虚拟环境。使用方法可参考 [Conda Cheat Sheet](https://docs.conda.io/projects/conda/en/latest/user-guide/cheatsheet.html)。

## **建议**在 conda 中仅使用一种包安装机制。

**原因**

在 conda 环境中，用 `conda install` 和 `pip install` 安装包的底层机制完全不同，混用可能导致依赖关系错误，安装的包不能正常使用。

**对策**

在 conda 环境中，仅使用 `conda install` 或 `pip install` 安装包。由于 `pip` 支持的 Python 包通常是 `conda` 的超集，建议优先使用 `pip` 安装包。

```admonish tip
推荐用一个 `requirements.txt` 文件记录所有需要安装的包，然后用 `pip install -r requirements.txt` 一次性安装所有包。

参考 [`requirements.txt` 用法](https://pip.pypa.io/en/stable/user_guide/#requirements-files)。
```

## **建议**在实验结束后检查残留进程。

**原因**

一些进程在实验结束后可能仍在运行，例如 Jupyter Notebook 等。这些进程会占用系统资源（例如 GPU、内存、端口、硬盘文件等），造成大量资源浪费。

**对策**

每次运行以下程序后，留意是否有残留进程，如果有，手动杀死。

- **VSCode 中打开 ipynb 文件**时，会自动启动 ipykernel 进程，关闭 VSCode 后，ipykernel 进程有时仍会继续运行。
- **VSCode 非正常断开**（例如在笔记本上写代码后直接合盖休眠）后，vscode-server 进程有时仍会继续运行。
- **Jetbrains Gateway** 连接到服务器后，即使关闭了客户端，后端进程（例如 pycharm）有时仍会继续运行。
- 后台运行程序（即使用 `&`）时，脚本运行结束后，该后台进程可能仍会继续运行。

如果不能注意，至少做到在每次实验结束后，完全退出编辑器再关机。


## **可以**使用 tmux 管理长时间运行实验的会话。

「会话」是 Linux 中的一个概念，可以理解为一个终端窗口。例如你在服务器上打开了两个可以输入命令的终端窗口，那么这两个窗口就是两个会话。

**原因**

tmux 是一个终端复用工具，可以在同一个终端窗口中创建多个会话，这些会话在用户断开后仍能继续运行，并且可以在多个终端窗口中连接到同一个会话。

**对策**

参考 [Tmux Cheat Sheet & Quick Reference](https://tmuxcheatsheet.com/) 学习 tmux 的基本用法，也可以阅读下面的极简指南。

类似的工具还包括 [GNU Screen](https://www.gnu.org/software/screen/)。

~~~admonish tip
可以配置 tmux 为支持鼠标操作，这样就可以通过鼠标点击来切换窗口、滚动屏幕等：

```bash
echo "set -g mouse on" >> ~/.tmux.conf
```

配置后需要重新连接会话才能生效。
~~~

~~~admonish example
开启会话：

```bash
$ tmux new -s my-session-name
```

断开连接：

`Ctrl + b`，然后按 `d`（意为 detach）。

重新连接：

```bash
$ tmux a -t my-session-name
```

新建窗口：

`Ctrl + b`，然后按 `c`（意为 create）。

切换窗口：

直接用鼠标点击下面的 Tab 栏即可。

关闭整个会话和所有窗口：

`Ctrl + b`，然后按 `&`，再输入 `y` 回车确认。
~~~
