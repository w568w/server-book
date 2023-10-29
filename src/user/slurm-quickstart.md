# Slurm 极简手册
本文是面向普通用户的 Slurm 速查手册，主要介绍 Slurm 的常用命令。

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
如果显示出来的表格太宽以至于有跨行，可以适当缩减列宽，通过减少 `%` 后面的数字来实现。
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