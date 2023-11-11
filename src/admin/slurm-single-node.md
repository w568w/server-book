# Slurm（单机）部署

```admonish warning
本文假定服务器操作系统的发行版为 Ubuntu 20.04 LTS，其他版本的 Ubuntu 或者其他 Linux 发行版可能需要做一些调整。

本文进行的是**最小安装**，依照安全最佳实践，你可能还需要对涉及的各个部件配置认证、防火墙等。
```

## 1. 安装 Slurm

需要安装的软件包：
- `slurmctld`：Slurm 控制器守护进程
- `slurmdbd`：Slurm 数据库守护进程。数据库主要用于存储作业历史数据，显示统计信息等
- `slurmd`：Slurm 计算节点守护进程
- `munge`：Munge 认证服务，用于 Slurm 组件之间的权限认证
- [`mariadb`](https://mariadb.org/)：MariaDB 数据库，也可以使用 [MySQL](https://www.mysql.com/)

~~~admonish tip
可以运行 mariadb 的附带脚本配置数据库安全，如：

```
# mysql_secure_installation
```
~~~
安装后先不急着启动，先做配置。

## 2. 配置数据库

Slurm 的账户信息需要存储在数据库中，需要先创建数据库和用户。

```admonish note
注意，在配置过程中涉及到三种「账户」：

- **数据库账户**：slurmdbd 用于连接数据库的账户，需要在数据库中创建
- **Linux 账户**：用于组件间最小权限管理（通常安装时会自动创建，分别名为 `slurm`、`munge`）
- **Slurm 账户**：用于分组限制使用配额和记账统计数据的用户，需要在 Slurm Accounting Manager 中创建
```

在 mysql 终端（例如 `# mysql`）中执行以下命令：

```sql
-- 创建数据库
create database slurm_acct_db;
-- 创建用户，别忘了修改密码
create user 'slurm'@'localhost';
set password for 'slurm'@'localhost' = password('i_am_the_password_plz_change_me');
-- 授权
grant all on slurm_acct_db.* to 'slurm'@'localhost';
flush privileges;

exit
```

## 3. 配置 `slurmdbd.conf`

需要告诉 Slurm 的数据库守护进程 `slurmdbd` 数据库的位置和用户信息。

在 `/etc/slurm-llnl/slurmdbd.conf` 中添加以下内容：

```ini
# 数据库连接信息，和 MySQL/MariaDB 的配置一致
StorageType=accounting_storage/mysql
StorageHost=localhost
StoragePort=3306
StorageLoc=slurm_acct_db
StorageUser=slurm
StoragePass=i_am_the_password_plz_change_me

# 配置其他组件连接到 SlurmDBD 的信息
AuthType=auth/munge
DbdHost=server # SlurmDBD 所在主机的主机名
DbdPort=6819
SlurmUser=slurm # 作为数据库管理员的用户名，务必和下面的 slurm.conf 保持一致，否则 slurmctld 无法修改数据库

# 日志位置
LogFile=/var/log/slurm-llnl/slurmdbd.log
```



## 4. 配置 `slurm.conf`

`slurm.conf` 是 Slurm 所有组件的配置文件，需要放在 `/etc/slurm-llnl/` 目录下。

```admonish note
其他发行版安装后可能在其他目录位置，例如 `/etc/slurm/`。
```

默认情况下附带了一个 `configurator.html`，可以在安装位置找到，推荐使用它来生成初始配置文件。

```admonish example
在我的情况下，安装位置为 `/usr/share/doc/slurmctld/slurm-wlm-configurator.html`。
```

可以将该网页文件复制到本地，然后在浏览器中打开，按照提示进行配置。

配置项较多，请耐心一个一个检查。下面列出一些需要注意的（不同版本的 Slurm 可能命令有所不同）：

- `SlurmctldHost`：Slurm 控制器的主机名，也就是你的主机名（`$ cat /etc/hostname`）
- `NodeName`：计算节点名，可任意设置（例如 `node`、`server` 等）
- `ProctrackType`、`TaskPlugin`：有关进程跟踪的配置，建议一律配置为 Cgroup
- `SelectTypeParameters`：需要配置哪些为可消费资源，建议配置为 `CR_Core_Memory`（CPU 核心数和内存）
- `AccountingStorageType`：设为 SlurmDBD，即使用数据库存储账户信息
- `AccountingStorageLoc`：数据库名，例如 `slurm_acct_db`
- `AccountingStorage[Host/Port/User/Pass]`：连接到 SlurmDBD 的信息，和上面的 `slurmdbd.conf` 保持一致
- `ClusterName`：集群名，可任意设置

```admonish warning
注意，`AccountingStorage[Host/Port/User/Pass]` 是**连接到 SlurmDBD** 的信息，而**不是连接到数据库**的信息。

也就是说，在上面的配置文件下，应该分别是 `server`、`6819`、`slurm`、留空（`AccountingStoragePass` 的含义**不是字面意思，请参考文档**。我们没有配置过，不要填写）。
```

配置完成后，将生成的 `slurm.conf` 保存，再添加（或更新）如下配置项：

```ini
# 追踪资源类型
AccountingStorageTRES=gres/gpu
# 在 Slurm 日志中显示 CPU 绑定和 GPU 使用情况，便于调试
DebugFlags=CPU_Bind,gres
# 使用 Cgroup 进行作业统计
JobAcctGatherType=jobacct_gather/cgroup

# ...
# 以下配置添加在节点附近

# 注册 GRes 类型
GresTypes=gpu
# 给节点声明 GRes 配置，gpu:2 表示每个节点有 2 个 GPU。按照你的实际情况修改
NodeName=node Gres=gpu:2
```

完成后，将 `slurm.conf` 写入到 `/etc/slurm-llnl/` 目录下。

## 5. 配置 `cgroup.conf`

Slurm 的 Cgroup 配置文件是单独的，需要放在 `/etc/slurm-llnl/cgroup.conf` 位置。

示例内容如下：

```ini
CgroupAutomount=yes 
```

## 6. 配置 `gres.conf`

Slurm 的 GRes 配置文件也是单独的，用于指定参与调度的 GRes 设备（在我们这里就是 NVIDIA GPU）的信息，需要放在 `/etc/slurm-llnl/gres.conf` 位置。

```admonish note
**GRes** 是 Generic Resource 的缩写，表示通用资源。在 Slurm 中，GRes 用于表示除了 CPU 和内存以外，其他可管理的调度资源。
```

示例内容如下：

```ini
# 有几个 GPU 就写几个，请按照实际情况修改
Name=gpu File=/dev/nvidia0
Name=gpu File=/dev/nvidia1
```

```admonish warning
Slurm 对 GPU 的控制非常弱，实际上只是通过设置环境变量「建议」程序使用哪些 GPU，而不会真的限制程序使用。

可参考 [Slurm 极简手册](/user/slurm-quickstart.md) 中的相关介绍。

一些相关问答包括：

1. [Slurm - GPU enforcement with cgroups](https://superuser.com/questions/1480883/slurm-gpu-enforcement-with-cgroups)
2. [SLURM: restrict GPU access only to SLURM](https://unix.stackexchange.com/questions/437589/slurm-restrict-gpu-access-only-to-slurm)
```

## 7. 启用服务

用以下命令启用并启动服务：

```
$ systemctl enable --now slurmdbd
$ systemctl enable --now slurmctld
$ systemctl enable --now slurmd
```

第一次启动时还要将当前节点的状态设为 IDLE（默认值是 UNKNOWN）：

```
# scontrol update nodename=node state=IDLE
```

## 8. 测试

在节点上运行以下命令：

```
$ srun --gres=gpu:<要请求的显卡数> env | grep CUDA
```

显示 `CUDA_VISIBLE_DEVICES=<显卡位置>` 说明配置成功。

```admonish tip

接下来，你可以使用 `sbatch` 提交作业，使用 `squeue` 查看作业状态，使用 `sacct` 查看作业统计信息。

具体教程可以直接参考 [Slurm 作业调度系统使用指南](https://zhuanlan.zhihu.com/p/356415669)。
```

## 参考文档

1. Slurm 资源管理与作业调度系统安装配置：<https://hmli.ustc.edu.cn/doc/linux/slurm-install/>
