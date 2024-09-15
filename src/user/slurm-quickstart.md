# Slurm 极简手册
本文是面向普通用户的 Slurm 速查手册，主要介绍 Slurm 的常用命令。

```admonish info title="小广告"
我正在开发一个 Slurm 的竞品调度系统，[TurboSched](https://github.com/w568w/TurboSched)，目前仍在原型阶段。欢迎关注！
```

```admonish note title="其他 Slurm 学习资源"
- [Slurm 官方文档（英文）](https://slurm.schedmd.com/documentation.html)
- [Slurm 作业调度系统使用指南](https://zhuanlan.zhihu.com/p/356415669)
```

## 什么是 Slurm
Slurm 是一个开源的作业调度系统，用于管理集群中的计算资源，实现作业的提交、调度、执行和管理。它可以解决以下场景中的问题：

- GPU「僧多粥少」，如何保证公平分配？
- 计算资源已占满时，新的任务如何在有空闲资源时立即执行？
- 如何了解其他用户的排队作业情况，避免资源冲突？
- 如何统计用户的作业信息，方便后续的资源分配和管理？

## 提交任务

~~~admonish error title="常见误区必读！"
1. **Slurm 不会真正追踪 GPU 的使用状态**：它认为的「GPU 空闲」是指**它之前没有把这张卡分配给其他任务**。例如，你直接在 Slurm 外运行了一个使用 GPU 0 的任务，哪怕 GPU 使用率 100%，Slurm 仍然会认为 GPU 0 是空闲的，可以分配给其他任务。只有你使用 Slurm 调度任务，然后 Slurm 把这张卡分配给了你的任务，它才会认为这张卡是「被占用」的；
2. **Slurm 不会强制让你的程序只能使用指定的 GPU**：Slurm「分配」给你的方法是在运行开始时设置环境变量 `CUDA_VISIBLE_DEVICES`，除此之外，它什么都没有做：你的程序可以看到所有 GPU，也可以通过重写环境变量的方式使用其他 GPU。

因此，如果你想遵守 Slurm 的游戏规则，让你的程序只使用 Slurm 分配给你的 GPU，需要做到：

- 你的程序中不可以显式地修改 `CUDA_VISIBLE_DEVICES` 环境变量；
- 设置了 `CUDA_VISIBLE_DEVICES` 时，GPU 枚举顺序会被重新编号（例如 `CUDA_VISIBLE_DEVICES=3,7` 时，`cuda:1` 指的是 7 号 GPU），所以你的程序中只可以使用 `cuda`、`cuda:0`、`cuda:1` 等这样的从 0 开始的编号。
~~~

### a. 我要运行一行 python 命令
直接在原命令前加上 `srun` 即可，例如：

```sh
$ srun --gres=gpu:1 python train.py
```

`--gres=gpu:1` 是指申请 1 个 GPU 资源，如果需要多个 GPU，可以写成 `--gres=gpu:<GPU 数量>`。

### b. 我运行的命令里需要输入交互，例如 `bash`, `pdb` 等

```sh
$ srun --gres=gpu:1 --pty bash
```

### c. 我要运行一个脚本文件
使用 `sbatch` 命令，例如：

```sh
$ sbatch --gres=gpu:1 train.sh
```

~~~admonish tip
`sbatch` 运行的脚本可以包含配置参数，详细参数可参考[上海交大超算平台用户手册](https://docs.hpc.sjtu.edu.cn/job/slurm.html)。
~~~

### d. 我要运行一个 IPython 笔记本（例如 Jupyter Notebook）

```admonish warning
本部分仅适用于简单的单节点 Slurm 环境，因为它依赖于「localhost 是计算节点」的前提。

如果你在使用包含多个节点的 Slurm，本部分介绍的命令不会正常工作。
```

```admonish warning
这里介绍的办法**在 Visual Studio Code 中也可以使用，但是可能有问题**。

**请先阅读**下面「d.1. 在 Visual Studio Code 中运行」中「体验下降注意」的内容。
```

用以下命令启动一个 Jupyter Notebook 服务：

```sh
# 执行前确保你已经切换到自己的 Python 环境（例如 conda activate xxxx）
$ srun --gres=gpu:1 jupyter notebook --no-browser --ip localhost --port `python -c "import socket; s = socket.socket(); s.bind(('', 0));print(s.getsockname()[1]);s.close()"` --ServerApp.token='123'
```

这在服务器本地任意一个空闲端口上启动了一个 Jupyter Notebook 服务，密码为 `123`。

启动之后会显示**端口号**，输出类似于：

```
[I 10:00:00.000 ServerApp] http://localhost:12345/?token=...
```

记住这个端口号。

~~~admonish tip
你也可以固定端口，便于在 Visual Studio Code 中多次使用。例如，固定端口为 8888，修改密码为 456：

```sh
# 执行前确保你已经切换到自己的 Python 环境（例如 conda activate xxxx）
$ srun --gres=gpu:1 jupyter notebook --no-browser --ip localhost --port 8888 --ServerApp.token='456'
```

~~~

接下来，根据运行环境的不同，有不同的对应步骤：

#### d.1. 在 Visual Studio Code 中运行
打开任意一个笔记本文件，在右上角切换内核，选择「选择其他内核」，然后选择「已有的 Jupyter 服务器」，输入 `http://localhost:12345`（端口号替换为刚才启动的），回车，输入连接密码连接即可。

~~~admonish warning title="体验下降注意"
截至 2023 年 11 月，Visual Studio Code 的 Jupyter Notebook 插件对连接到「已有的 Jupyter 服务器」的支持不太好，**代码补全、已安装包解析等功能都不会正常工作**。

如果你很介意，这里还有一种完全不同、让你可以完全按原来方式工作的方法，但不推荐使用：

1. 安装包 [`simple_slurm`](https://github.com/amq92/simple_slurm)：

```sh
$ pip install simple_slurm
```

2. 在笔记本文件中加入以下两个代码块：

```python
# 申请 GPU
from simple_slurm import Slurm
import os

slurm = Slurm(gres=['gpu:1'])
job_id = slurm.sbatch('env > slurm.env; yes > /dev/null')

with open("slurm.env", "r") as f:
    for line in f.readlines():
        if "CUDA_VISIBLE_DEVICES" in line:
            print(line.strip())
            os.environ["CUDA_VISIBLE_DEVICES"] = line.strip().split("=")[-1]

os.remove("slurm.env")
```

```python
# 释放 GPU
import os
os.system(f"scancel {job_id}")
```

3. 每次运行笔记本时，先运行第一个代码块，然后就可以像平常一样继续运行你的代码。**关闭笔记本前，务必运行第二个代码块。如果你忘记了，请务必用以下命令手动释放 GPU**：

```sh
$ scancel <第一个代码块输出的 job_id>
```
~~~

#### d.2. 在浏览器中使用 Jupyter Notebook
首先将服务器端口映射到本地，例如**在本地执行**（端口号 12345 替换为刚才启动的，`<username>` 和 `<hostname>` 替换为你实际连接服务器时用的用户名和服务器地址）：

```sh
$ ssh -L 12345:localhost:12345 <username>@<hostname>
```

然后在浏览器中打开 `http://localhost:12345` 即可。

## 查看队列
### a. 当前正在运行和排队等待的任务
使用 `squeue` 命令，例如：

```sh
# 简单查看，不显示每个任务申请的具体资源数量
$ squeue
# 完整查看
$ squeue -o "%.18i %.9P %.40j %.8u %.8T %.6D %.4C %.5D %.6m %.13b %.10M"
```

~~~admonish tip
这个命令使用会非常频繁，可以在 Shell 配置文件中添加一个别名，例如 `sq`：

```sh
alias sq='squeue -o "%.18i %.9P %.40j %.8u %.8T %.6D %.4C %.5D %.6m %.13b %.10M"'
```
~~~

~~~admonish tip
如果显示出来的表格太宽以至于跨行显示，可以适当缩减列宽，通过减少 `%` 后面的数字来实现。
~~~


### b. 只查看某个用户的任务
```sh
$ squeue -u <Linux 用户名>
```

### c. 查看历史任务
```sh
$ sacct
```

```admonish info
普通用户只能看到自己的历史任务。
```

## 取消正在执行或排队的任务
先查看队列，得到任务号，然后

```sh
$ scancel <任务号>
```

```admonish warning
对于正在运行的任务，这会强制终止它，可能会导致数据丢失。
```