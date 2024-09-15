# 如何安装不同版本的 CUDA

CUDA 是 NVIDIA 公司推出的并行计算平台和编程模型，用于利用 NVIDIA GPU 的并行计算能力。

**NVIDIA 公司混乱的术语表达，使人常常对 CUDA 的概念辨析和安装产生困惑**。本文将介绍如何在 Linux 上正确安装不同版本的 CUDA。

## 1. CUDA 的相关术语辨析

```admonish tip
本节仅仅做术语和概念的介绍，不感兴趣可以直接跳到下一节。

**太长不看：**

- CUDA 和驱动是不同的，根本不存在「CUDA 驱动」这个糅合的概念。
- CUDA 就是普通的库，更换版本不需要重新安装驱动。
- 但不同版本的驱动支持的 CUDA 版本范围不同。
- `nvidia-smi` 命令里显示的是「驱动最高支持的 CUDA 版本」，不是「当前的 CUDA 版本」。因为同一个电脑上可以安装零到多个版本的 CUDA，所以「当前的 CUDA 版本」这样的概念根本不存在。
```

我们常说的「CUDA」，有三个主要组成部分：

- **NVIDIA Graphics Driver**: NVIDIA 显卡驱动程序，用于内核和 GPU 之间的通信，是下面所有模块的基础。它由两部分组成：一些内核模块（主要是 `nvidia.ko`）和一些用户空间库（主要是 `libnvidia-xxx.so`）。版本号目前是 `560.63`。
- **CUDA Runtime**: CUDA 运行时库，用于将 CUDA 函数调用转换为 GPU 指令。它是一些用户空间库（主要是 `libcudart.so`）。版本号目前是 `12.6`。
- **CUDA Toolkit**: CUDA 工具包，用于开发、编译和测试 CUDA 程序。它包含了 CUDA Runtime 和一些开发工具，如 `nvcc` 编译器。版本号目前是 `12.6`。

可以看出，在一台使用 NVIDIA 显卡的机器上，NVIDIA Graphics Driver 是必须的，而 CUDA Runtime 和 CUDA Toolkit 是可选的。实际上，用 pip 或 conda 安装 PyTorch 时，都会自动安装 CUDA Runtime。**因此，只要安装了 NVIDIA Graphics Driver，就可以在机器上直接安装 PyTorch 等深度学习框架**。

另一方面可以看出，**「CUDA 驱动」的说法是非常不严谨的**。上面三个部分中，只有 NVIDIA Graphics Driver 才是真正的「驱动」，后两个都是普通的用户空间库文件。

然而，NVIDIA 在发布驱动时，会笼统地称之为「CUDA 驱动」，把三个部分全部捆绑在一个 `.run` 安装包中。这就导致了很多人产生了**错误的观念，以为想要改变 CUDA 的版本，就必须重新安装驱动。实际上，CUDA 版本和驱动版本是相互独立的，根本不是同一个东西。**

一定要说两者间有关系的话，那就是**不同版本的驱动支持的 CUDA 版本范围不同**。例如，`460` 驱动支持最高 `11.2` 版本的 CUDA，`470` 驱动支持最高 `11.4` 版本的 CUDA（数字是胡诌的，请勿当真）。这个版本号对应关系可以在[这里](https://docs.nvidia.com/deploy/cuda-compatibility/index.html)查到，也可以通过 `nvidia-smi` 命令查看。

## 2. 安装另一个版本的 CUDA

由于 CUDA Toolkit 包含了 CUDA Runtime，所以我们只需要安装 CUDA Toolkit 即可。下面以安装 CUDA 12.1 旧版本为例。

1. 目前，只有 conda 提供了方便的安装方式，可以在自己的环境中通过下面的命令安装：

    ```bash
    (my_env) $ conda install conda-forge::cuda-toolkit=12.1
    ```

```admonish question title="为什么用 conda-forge？"
anaconda 中目前有三个主要的源提供了 CUDA Toolkit：main、nvidia 和 conda-forge。三者之间的区别是：

- **main**：Anaconda 公司维护的源，其 CUDA Toolkit 版本更新非常不及时（实际上，main 源里绝大部分包都是非常老的版本，因此我们根本不推荐使用 main 源）。
- **nvidia**：NVIDIA 公司维护的源，其 CUDA Toolkit 版本更新较为及时，可以使用。
- **conda-forge**：社区维护的源，其 CUDA Toolkit 版本更新最及时，常常比 nvidia 源还要快。

因此推荐使用 conda-forge 源。
```

2. 安装完成后，可以通过下面的命令查看安装的 CUDA 版本：

    ```bash
    (my_env) $ nvcc --version
    ```

    如果看到输出中包含 `release 12.1`，则说明安装成功。否则，可能是因为 conda 的环境变量没有设置正确，可以尝试重启 Shell 或者手动检查环境变量：
    
    ```bash
    (my_env) $ echo $CONDA_PREFIX # 检查 conda 的根目录
    (my_env) $ echo $PATH # 检查 PATH 是否包含了 conda 的 bin 目录
    ```

    如果 PATH 没有包含 conda 的 bin 目录，或者在 bin 目录前面还有别的有关 cuda 的路径，这意味着 PATH 被覆盖了。通常是因为在 Shell 配置文件中写了类似下面的代码：

    ```bash
    export PATH=/usr/local/cuda/bin:$PATH
    ```

    这通常不是你想要的，不管你是从什么地方复制的这行糟糕的配置，都应该删除它。

```admonish info title="不同 Shell 配置文件的路径"
- bash: `~/.bashrc`
- zsh: `~/.zshrc`
```

3. 如果你只需要运行该版本的 `nvcc`（也就是只需要编译器的功能），到这里已经全部结束了。但是，如果你还希望程序能够优先使用这个版本的 CUDA Runtime，可以通过下面的命令设置环境变量：

    ```bash
    (my_env) $ export LD_LIBRARY_PATH=$CONDA_PREFIX/lib:$LD_LIBRARY_PATH
    ```

    这样，当你运行的程序需要 CUDA Runtime 时，会优先使用这个 12.1。

```admonish question title="为什么上面说不要用 export，这一步又推荐用了？"
conda 目前的默认行为是会设置 `PATH`，但不会设置 `LD_LIBRARY_PATH`（可以用 `$ conda info --system` 查看）：[set LD_LIBRARY_PATH when activate](https://github.com/conda/conda/issues/12800)。

因此，如果你不设置 `LD_LIBRARY_PATH`，那么你的程序在运行时就可能找不到正确的 CUDA Runtime 库文件。所以这里推荐手动设置 `LD_LIBRARY_PATH`。

当然，`export` 永远是临时设置环境变量的方法，如果你希望永久设置，推荐的方法有两个：

1. **`$ conda env config vars set LD_LIBRARY_PATH=$CONDA_PREFIX/lib`**：这样会将 `LD_LIBRARY_PATH` 写入到当前环境的配置文件中。每次激活这个环境时，都会自动设置这个环境变量。优点是不会总是设置这个环境变量，缺点是设置时会覆盖原有的 `LD_LIBRARY_PATH`。
2. **在 Shell 配置文件中写入 `export LD_LIBRARY_PATH=<你 $CONDA_PREFIX 的值>/lib:$LD_LIBRARY_PATH`**：这样会将 `LD_LIBRARY_PATH` 写入到 Shell 配置文件中。每次打开 Shell 时，都会自动设置这个环境变量。优缺点和上面的方法相反。
```