# Python 安装方式全面指南

## 概述

在 Linux/Unix 系统上安装 Python 有多种方式，选择哪种取决于你的场景：开发环境、生产环境、多版本并存、离线安装等。本文详细介绍每种方式的步骤、优缺点和适用场景。

---

## 一、各方式速览对比

| 方式 | 版本自由度 | 多版本并存 | 需要 root | 需要外网 | 推荐场景 |
|------|-----------|-----------|----------|---------|---------|
| yum/apt 系统包管理器 | 低 | 否 | 是 | 是 | 快速使用，不要求最新版本 |
| 下载源码编译 | 高 | 是 | 否（可自定义路径） | 否 | 老系统、定制编译选项 |
| pyenv | 极高 | 是 | 否 | 首次安装需要 | 开发环境，多项目多版本 |
| conda / miniconda | 高 | 是 | 否 | 是 | 数据科学，复杂 C 依赖 |
| 预编译二进制包 | 中 | 是 | 否 | 否 | 离线部署 |
| docker 容器 | 极高 | 是 | 是 | 首次拉镜像需要 | 生产环境，完全隔离 |

---

## 二、方式一：系统包管理器（yum / apt / dnf）

### 原理

直接从 Linux 发行版的官方软件仓库安装预先编译好的 Python 包。安装快，自动处理依赖。但版本受发行版策略限制，通常不是最新。

### CentOS / RHEL 7

```bash
# 安装 EPEL 和 SCL（Software Collections，提供较新版本）
yum install -y epel-release centos-release-scl

# 查看可用 Python 版本
yum list available rh-python*

# 安装 Python 3.9
yum install -y rh-python39

# 启用（不会替换系统默认 Python）
scl enable rh-python39 bash
python3 --version

# 永久启用：写入 bashrc
echo 'source /opt/rh/rh-python39/enable' >> ~/.bashrc
source ~/.bashrc
```

### Ubuntu / Debian

```bash
# 查找可用版本
apt-cache search python3 | grep "^python3"

# 安装
apt install -y python3.10 python3.10-pip python3.10-venv

# 多版本管理可以用 update-alternatives
update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1
```

### 优势
- 一条命令搞定，依赖自动处理
- 与系统集成好，包管理器统一管理更新

### 劣势
- 版本通常不是最新的
- 不能细粒度控制编译选项（如启用/禁用某些模块）
- 需要 root，影响系统级 Python
- CentOS 7 的 SCL 源已下线，需切到 vault.centos.org
- 多版本并存需要额外工具（alternatives 或 scl）

### 适用场景
快速部署，不追求最新版本，系统级脚本，学习环境。

---

## 三、方式二：源码下载编译安装

### 原理

从 python.org 下载官方源码包，在本机编译。完全掌控编译选项（如启用优化、指定 OpenSSL 路径、启用/禁用模块）。编译出的二进制天然匹配本机 glibc 和内核版本，对老系统兼容性最好。

### 步骤

```bash
# 1. 安装编译依赖
yum install -y gcc make openssl-devel bzip2-devel libffi-devel zlib-devel readline-devel sqlite-devel

# 2. 下载源码
cd /usr/src
curl -O https://www.python.org/ftp/python/3.10.14/Python-3.10.14.tar.xz
tar xJf Python-3.10.14.tar.xz
cd Python-3.10.14

# 3. 配置（--enable-optimizations 会跑测试优化，慢但性能更好）
./configure \
  --prefix=/usr/local/python3.10 \
  --enable-optimizations \
  --enable-shared

# 4. 编译（-j 并行编译，$(nproc) 用满所有 CPU 核）
make -j$(nproc)

# 5. 安装（用 altinstall 防止覆盖系统 python）
make altinstall

# 6. 使用
/usr/local/python3.10/bin/python3.10 --version
/usr/local/python3.10/bin/pip3.10 install <包名>

# 7. 可选：创建软链接
ln -s /usr/local/python3.10/bin/python3.10 /usr/local/bin/python3.10
```

### CentOS 7 / RHEL 6 等老系统特别处理

老系统 OpenSSL 太老，必须装新版本并在编译时指定：

```bash
# CentOS 7
yum install -y openssl11-devel

# 编译时指定 OpenSSL 路径
CFLAGS="-I/usr/include/openssl11" \
LDFLAGS="-L/usr/lib64/openssl11 -lssl -lcrypto" \
./configure --prefix=/usr/local/python3.10 --enable-optimizations

make -j$(nproc)
make altinstall
```

### 优势
- 完全掌控编译选项
- 可以装到任意路径，不需要 root（选用户目录即可）
- 对老系统兼容性最好
- 不依赖外网（源码包可以离线拷贝）

### 劣势
- 编译耗时（3-10 分钟）
- 需要手动解决编译依赖（gcc、openssl 等）
- 没有版本管理功能，多版本需手动管理路径

### 适用场景
老系统（CentOS 6/7、RHEL 6）、需要定制编译选项、离线环境。

---

## 四、方式三：pyenv

### 原理

pyenv 是一套 bash 脚本，在 `~/.pyenv/versions/` 下编译和存储多个 Python 版本。通过环境变量 `PATH` 和 shims 机制，在不同目录自动切换 Python 版本。每个项目可以用 `.python-version` 文件锁定版本，进目录自动切。

### 架构

```
~/.pyenv/
├── bin/          # pyenv 命令
├── shims/        # 代理脚本，拦截 python/pip 调用
├── versions/     # 各 Python 版本安装目录
│   ├── 3.9.21/
│   └── 3.10.14/
└── cache/        # 源码包缓存（预放文件可跳过下载）
```

### 安装与使用

```bash
# 1. 安装 pyenv
git clone https://github.com/pyenv/pyenv.git ~/.pyenv

# 2. 配置 shell
echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
exec $SHELL

# 3. 安装编译依赖
yum install -y gcc make openssl-devel bzip2-devel libffi-devel zlib-devel \
  readline-devel sqlite-devel

# 4. 查看可安装的版本
pyenv install --list

# 5. 安装 Python（会从 python.org 下载源码并编译）
pyenv install 3.10.14

# 6. 全局设置默认版本
pyenv global 3.10.14

# 7. 项目级设置（在项目目录下）
cd /path/to/myproject
pyenv local 3.10.14
# 会生成 .python-version 文件，进目录自动切换
```

### 加速下载

如果 python.org 直连慢，提前把包下好放入缓存目录：

```bash
# 从国内镜像下载源码包
cd /tmp
curl -O https://mirrors.huaweicloud.com/python/3.10.14/Python-3.10.14.tar.xz

# 放入 pyenv 缓存目录
mkdir -p ~/.pyenv/cache
cp Python-3.10.14.tar.xz ~/.pyenv/cache/

# pyenv install 发现缓存有包就跳过下载，直接编译
pyenv install 3.10.14
```

### 优势
- **多版本管理的最佳方案**：安装、切换、隔离都很方便
- 按项目自动切换版本，不需要手动 source
- 不碰系统 Python，不需要 root
- 每个版本在独立目录，删掉目录就彻底卸载
- `.python-version` 可跟着项目 git 提交，团队统一版本

### 劣势
- 首次装需要从 GitHub clone（约 7MB）
- 本质还是源码编译，老系统仍需要处理编译依赖（GCC、OpenSSL）
- 安装速度取决于编译速度（首次 3-5 分钟）

### 适用场景
**目前最主流的开发环境方案**。适合多项目、需要多版本并存、团队协作。

---

## 五、方式四：conda / Miniconda / Anaconda

### 原理

conda 是一个不依赖系统库的独立 Python 发行版。它自带 Python 解释器、glibc、openssl 等依赖库，与系统完全隔离。通过 `conda` 命令管理环境和包，不仅能装 Python 包，还能装 C 库和非 Python 工具。

### conda vs Anaconda vs Miniconda

| 名称 | 大小 | 预装包 | 适合 |
|------|------|--------|------|
| Anaconda | ~500MB | 150+ 数据科学包 | 数据科学家，开箱即用 |
| Miniconda | ~70MB | 仅 conda + Python | 开发者，按需装包 |
| miniforge | ~40MB | 仅 conda + Python | 同上，社区维护，conda-forge 优先 |

### 安装 Miniconda

```bash
# 1. 下载安装脚本
curl -L -O https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 2. 安装（-b 静默，-p 指定路径）
bash Miniconda3-latest-Linux-x86_64.sh -b -p ~/miniconda3

# 3. 初始化 shell
~/miniconda3/bin/conda init bash
exec $SHELL

# 4. 创建 Python 3.10 环境
conda create -n proxy_browser python=3.10 -y

# 5. 激活环境
conda activate proxy_browser

# 6. 装包（conda 装装不上的再用 pip）
conda install requests lxml
pip install DrissionPage
```

### 常用命令

```bash
# 查看所有环境
conda env list

# 创建环境（指定 Python 版本）
conda create -n myenv python=3.11

# 克隆环境
conda create -n newenv --clone oldenv

# 导出环境
conda env export > environment.yml

# 从文件恢复环境
conda env create -f environment.yml

# 删除环境
conda remove -n myenv --all
```

### 优势
- 自带 glibc、openssl 等系统依赖，不依赖主机系统库
- 环境隔离彻底：不同项目不同环境，互不影响
- 装 C 库很方便：`conda install gcc`、`conda install cuda`
- 数据科学/机器学习生态最完善（numpy、scipy、pytorch 等预编译优化）
- 跨平台：Linux、macOS、Windows 用法一致

### 劣势
- 安装包大（Miniconda ~70MB，Anaconda ~500MB）
- 需要 glibc 版本匹配（现代 conda 需要 glibc ≥ 2.17，RHEL 6 只有 2.12 用不了）
- 装包慢（conda 的依赖解析比 pip 慢）
- 商业使用有许可限制（Anaconda 2024 年后收费，Miniconda 仍免费）

### 适用场景
数据科学、机器学习、需要复杂 C 依赖项目、跨平台开发。

---

## 六、方式六：预编译二进制包（python-build-standalone）

### 原理

有些项目提供已经编译好的、静态链接的独立 Python 二进制包，不必在目标机器上编译，下载解压即用。适合离线或老系统环境。

### 项目

- [python-build-standalone](https://github.com/indygreg/python-build-standalone)：提供各版本静态编译的 Python
- [python-for-android](https://github.com/kivy/python-for-android)：Android 交叉编译

```bash
# 下载
curl -L -O https://github.com/indygreg/python-build-standalone/releases/download/20240107/cpython-3.10.13+20240107-x86_64-unknown-linux-gnu-pgo+lto-full.tar.zst

# 解压
tar --zstd -xf cpython-3.10.13+20240107-x86_64-unknown-linux-gnu-pgo+lto-full.tar.zst

# 使用
./python/bin/python3 --version
```

### 优势
- 不需要编译，下载解压即用
- 自带依赖库，不碰系统库

### 劣势
- 社区维护，非官方，版本更新可能滞后
- 静态编译的二进制对内核版本有最低要求
- pip 包的 C 扩展可能编译不了（缺头文件）

### 适用场景
快速尝鲜、临时使用、对版本要求不高的离线环境。

---

## 七、方式七：Docker 容器

### 原理

在容器里运行 Python，利用 Docker 镜像提供的完整 Linux 用户空间，与主机内核共享但库完全隔离。镜像里预装了 Python 和依赖，一行命令即可运行。

### 使用

```bash
# 拉取官方 Python 镜像
docker pull python:3.10-slim

# 交互式运行
docker run -it --rm python:3.10-slim python

# 映射本地目录运行脚本
docker run -it --rm \
  -v /path/to/project:/app \
  python:3.10-slim \
  python /app/run.py

# 用 Dockerfile 构建自定义镜像
cat > Dockerfile << 'EOF'
FROM python:3.10-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "run.py"]
EOF

docker build -t myapp .
docker run -d myapp
```

### 优势
- 完全隔离：容器里 Python、glibc、系统库都是独立的
- 环境可复制：同样的 Dockerfile 在任何机器上产出一样的环境
- 生态丰富：任何版本都有官方镜像

### 劣势
- **需要主机内核 ≥ 3.10**（docker 依赖 cgroups 和 namespace）
- 需要安装 Docker 引擎
- 镜像体积大（slim 版 ~150MB，完整版 ~900MB）
- 需要 root 或加入 docker 组

### 适用场景
生产部署、微服务、CI/CD、本地开发环境隔离。

---

## 八、各种方式的适用场景总结

| 场景 | 推荐方式 | 原因 |
|------|---------|------|
| 个人开发机，多个项目 | **pyenv** | 按项目自动切换版本，管理最方便 |
| 数据科学/机器学习 | **conda** | C 库预编译优化，cuda/numpy/pytorch 支持好 |
| 老旧系统（CentOS 6/7、RHEL 6） | **源码编译** | GCC + OpenSSL 本地编译，不依赖系统库版本 |
| 快速尝鲜一个脚本 | **系统 yum/apt** | 一条命令搞定 |
| 生产环境部署 | **Docker** | 环境完全一致，回滚方便 |
| 离线环境 | **conda** 或 **源码编译** | 提前下载包，拷过去装 |
| CI/CD 流水线 | **Docker** 或 **pyenv** | 环境可复现 |

---

## 九、当前主流趋势

**开发环境：pyenv + poetry 是最流行的组合。**

- pyenv 管理 Python 版本
- poetry 或 pipenv 管理项目依赖和虚拟环境

典型工作流：

```bash
# 在项目目录下
pyenv local 3.10.14        # 锁定 Python 版本
python -m venv .venv       # 创建项目虚拟环境
source .venv/bin/activate
pip install -r requirements.txt
```

**生产环境：Docker 容器化是标准做法。**

应用打包成镜像，CI/CD 构建后推到镜像仓库，部署时拉取运行。版本、依赖、系统库全部在 Dockerfile 里声明。

**数据科学：conda 仍是首选。**

Jupyter、NumPy、PyTorch 等重度 C 库优化的生态让 conda 无可替代。

---

## 十、常见问题

### Q: pip install 和 conda install 有什么区别？
- `pip` 从 PyPI 下载 Python 包，纯 Python 或带 C 扩展，C 扩展会在本地编译
- `conda` 从 conda-forge / anaconda 仓库下载预编译的二进制包（含 C 库），不需要本地编译器
- 混用建议：先用 conda 装，conda 装不了的用 pip

### Q: 多个 Python 版本可以并存吗？
可以。pyenv 和 conda 天生支持多版本并存。源码编译装到不同路径也可以，修改 PATH 即可切换。

### Q: 为什么老系统装新 Python 这么麻烦？
新 Python 需要较新的 GCC 编译器和 OpenSSL。老系统自带的版本不够，需要手动装新版本。此外 pip 包的 C 扩展编译也可能需要新 glibc。

### Q: requirements.txt 里的包版本怎么定？
- `==` 精确锁定，生产环境必须用
- `>=` 最低版本，开发环境可以用
- 推荐用 `pip freeze > requirements.txt` 导出精确版本

### Q: pip 下载慢怎么办？
```bash
# 临时用国内镜像
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple <包名>

# 永久配置
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```
