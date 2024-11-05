# 管理 Python 环境的最佳实践

## 1. 为什么 Python 项目环境需要被管理？

我们在平时的工作中，经常会遇到这样的问题：

- 项目 A 需要 Python 3.6，项目 B 需要 Python 3.7，项目 C 需要 Python 3.8。**多个项目之间的 Python 版本不一致，导致我们需要频繁地切换 Python 版本**；
- 项目 A 需要使用 PyTorch 1.6，项目 B 需要使用 PyTorch 2.5。**同一个包不能同时安装多个版本**；
- 需要将项目 A 的环境迁移到另一台机器上或者分享到 GitHub 仓库中，**以便他人能够快速地复现项目环境和实验结果**。

为了实现这些目标，我们需要管理 Python 项目环境。项目环境管理的本质就是记录如何重现整个环境的配置信息，包括 Python 版本、依赖包、系统环境等。

传统上很多人喜欢直接用 `requirements.txt` 来记录依赖包，`pip` 来安装依赖包。但是这种方式有很多问题，相当过时。现在有很多更好的工具可以帮助我们管理 Python 项目环境，下面是笔者摸索出的一套最佳实践，与读者们分享。

## 2. 更好的非 Python 包环境管理：Miniforge 和 `environment.yml`

一提到 Python 本身的环境管理，大部分人的第一反应都是 Anaconda，然而却没有明确过 Anaconda 为什么好。

Anaconda 之所以好，是因为它提供了一个非常好的环境管理工具 `conda`，可以方便地管理 Python 环境和非 Python 包环境。然而 **Anaconda 并不等于 `conda`**，`conda` 是一个独立的环境管理工具，可以单独安装。

Miniforge 是一个轻量级的 `conda` 发行版，使用 `conda` 和 `mamba` 作为包管理器。它的环境和 `conda`（以及 Anaconda）完全兼容，但是更轻量、更快。

```admonish tip title="mamba 相对于 conda 的优势"
- ⚡ **运行更快**：mamba 用 C++ 编写，而 conda 用 Python 编写。启动速度上 mamba 几乎没有延迟，而 conda 总有 1-2 秒的延迟。
- ⚡ **解析依赖更快**：mamba 自行编写的解析器非常快，不会像 conda 一样卡在 `Solving environment` 半小时。甚至 conda 都要向 mamba 取经，[在 Anaconda 22.11 版本中引入了 mamba 的解析器](https://www.anaconda.com/blog/conda-is-fast-now)。
- ⚡ **下载包更快**：mamba 支持多线程下载。尽管 conda 也支持多线程下载，但是得益于 C++ 良好的多线程，mamba 下载速度实际上更快。
- 🤝 **完全兼容 conda**：mamba 的命令、参数和 conda 基本一致，只需要将 `conda install xxx` 替换为 `mamba install xxx` 即可开箱即用。
```

```admonish tip title="Miniforge 相对于 Anaconda 的优势"
- ⚡ **使用 mamba**：Miniforge 同时使用 mamba 和 conda 作为包管理器。在享受速度更快的 mamba 的同时，即使有的命令 mamba 不支持，也可以随时求助于 conda。
- 🪶 **更轻量**：Miniforge 只包含 `conda` 和 `mamba`，不包含其他无用的包。Anaconda 包含了几 GB 的包，而 Miniforge 只有几十 MB。
- 🌐 **统一社区源**：相比于 Anaconda 将数个维护质量良莠不齐、速度忽快忽慢、甚至可能相互冲突的企业源放在一起，Miniforge 从安装开始就只使用 conda-forge 社区源，彻底杜绝 “同一个包出现在多个源中、conda 随机选择一个源中的版本” 的问题。
```

Miniforge 由 Conda Forge 社区维护，直接从 [Miniforge 官网](https://conda-forge.org/miniforge) 安装即可。过程和安装 Anaconda 类似，不再赘述。

至于有人喜欢 Miniconda，我只能说：**你都知道 Miniconda 了，为什么不用 Miniforge？**

接下来就是环境的导出了。你可以使用 conda 自己的 `environment.yml` 文件来分享环境：

```bash
# 导出环境
conda env export > environment.yml
# 导入环境
conda env create -f environment.yml
```

非常简单轻松，不再赘述。

最后需要注意的是，本文一直在强调**非 Python 包环境**，因为 **conda 们并不擅长管理 Python 包，笔者也非常不推荐使用 conda 来安装、管理任何 Python 包**。如果你看到某个包提供了 `conda install xxx` 的安装方式，**请不要使用**。

一般来说，你的 conda 环境里只需要 `python`、`cuda-toolkit`、`gcc` 等非 Python 包，Python 包的管理交给下面的工具。

## 3. 闪电般的 Python 包环境管理：uv 和 `pyproject.toml`

Python 的包管理一直是一个老大难问题。老在于 Python 自出现以来，包括 `pip`、`pipenv`、`poetry` 等工具可以说是层出不穷，前赴后继地去解决这个问题；难在于 Python 将自己的包依赖做得过于灵活，以至于每个工具都有一些考虑不到的 Caveat。

关于环境的描述文件也是逐代更迭，目前的版本是 `pyproject.toml`，出自 [PEP 518](https://peps.python.org/pep-0518/)。而 `requirements.txt` 作为 `pip` 自己定义的 “非官方” 格式，也仍在广泛应用中，甚至在很多 Python 初学者和项目维护者中仍然是首选。

支持 `pyproject.toml` 的工具有 `poetry`、`pdm`、`uv` 等。其中 `poetry` 和 `pdm` 都是比较完善的工具，但是笔者认为它们都有一些问题，不够优雅。而 `uv` 作为一个新兴包管理器，解决了这些问题，是目前使用的首选。

```admonish tip title="uv 相对于 poetry 和 pdm 的优势"
- ⚡ **运行更快**：uv 用 Rust 编写，其启动速度只能用 “瞬间” 来形容。而 poetry 和 pdm 用 Python 编写，运行命令时总要卡数秒钟才会开始执行。
- ⚡ **解析依赖更快、下载更可靠**：uv 的解析机制非常快，并发进行、有缓存且对弱网络友好。以我自己的项目为例，uv 解析依赖只需要不到半分钟，而 pdm 则花了将近 2 小时在反复下载同一个 URL 包（[flash-attn](https://github.com/Dao-AILab/flash-attention/releases)）上，反复报错和失败；至于 poetry，笔者为了写这篇文章而进行的测试中，已经过去了 7 小时多，它仍卡在没有任何输出的 `Locking dependencies` 阶段。
- ❤️ **进度友好**：pdm 和 poetry（以及 conda）出于某些完全反人体工学、反人类的考虑，几乎隐藏了所有耗时长的过程的进度信息，让人无法知道当前进度和状态，只能干等着急。而 uv 会在终端输出所有下载和解析过程的详细进度信息，让人一目了然。
- 🤝 **兼容性强**：uv 本身支持直接作为 pip 的平替，只需将 `pip install xxx` 换成 `uv pip install xxx` 即可。另一方面，它也支持项目模式，即直接接管整个 conda 环境内的包管理。poetry 和 pdm 都只支持接管整个环境。
```

```admonish tip title="uv 相对于 pip 的优势"
- ⚡ **运行更更快**：pip 也是用 Python 编写的，所以……你懂的。
- 💪 **解析依赖更可靠**：pip 只会象征性地考虑现在已安装包的版本依赖关系，在出现包依赖破损时简单提示一些 warnings，然而实际上这些 warnings 代表整个项目环境的依赖不完备。这样的消息不仅容易被新手忽略（直到他们发出 “我的 Python 环境又被我搞烂了” 的哀叹），而且对修复依赖关系没有任何帮助。相对地，uv 会严格地校验整个环境中所有包的依赖关系，保证不多装、不少装、不错装。
- 🗑️ **支持 “完整” 卸载包**：pip 秉持 “只管装、不管卸” 的哲学，其卸载功能基本是半残废。例如安装了 A 包和它的 100 个依赖包，那么 `pip uninstall A` 实际上只会卸载 A，而完全不管这 100 个包，最终导致环境被各种无用的包填满，并出现各种依赖冲突问题。uv 则会严格检查卸载后哪些包不再需要，并及时卸载它们；
- 🆙 **支持升级包**：由于 pip 功能孱弱的包依赖解析器，其至今不支持安全地升级环境内的包。唯一保证升级成功的办法就是删掉整个环境，用 `requirements.txt` 或 `pip install` 手动重新安装所有包。uv 中则可以简单地使用 `uv lock --upgrade` 来获取所有包的更新，同时不 “搞烂” 任何依赖关系。
```

安装 uv 的推荐做法是在基础环境（Miniforge 的 base 环境或系统环境）中安装 pipx，然后使用 pipx 安装 uv：

```bash
# 安装 pipx。注意：不要在项目环境中执行这个命令！务必 mamba deactivate 退出当前环境再执行
pip install pipx
# 配置 pipx 到 PATH
pipx ensurepath
# 安装 uv
pipx install uv
```

对于还没有 `pyproject.toml` 的项目，可以使用 `uv init` 来初始化一个项目。之后就可以用以下命令来安装、卸载、升级包：

```bash
# 安装包
uv add xxx
# 卸载包
uv remove xxx
# 升级包
uv lock --upgrade
# 将当前环境和 pyproject.toml 中的包同步
uv sync
```

此外还有一些别的命令，以及关于 pip 中常见的附加 index（如 `pip install xxx -i https://xxx.com`）参数等功能，请在 [uv 的文档](https://docs.astral.sh/uv/reference/cli/#uv)中查看对应的做法。

## 4. 如何分享 Python 项目环境？

当你需要和他人分享你的 Python 项目环境时，只需要将 `environment.yml` 和 `pyproject.toml`（有必要的话，还有 `uv.lock`）一起分享。他人只需要安装 Miniforge 和 uv，然后执行以下命令：

```bash
# 导入环境
conda env create -f environment.yml
# 安装项目 Python 依赖
uv sync
```

这样就可以快速地复现你的项目环境了。