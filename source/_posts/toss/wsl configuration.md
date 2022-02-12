---
title: wsl 开发环境配置
categories: [linux, arch, configuration]
date: 2022-2-12 10:47:45
---

## 1. WSL2 安装

### 1.1 安装和启动 wsl

首先使用管理员权限(windows + R)打开 powershell/cmd/windows terminal，输入下面的指令，也就是打开 `windows subsystem for linux` 的功能

```powershell
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
wsl --set-default-version 2
```

重启电脑，这一步就完成了

### 1.2 安装 Linux 发行版

这一步骤也极其简单：在 `Microsoft Store` 中下载 ubuntu 或者 kali 发行版，就可以打开使用了。

不过今天要在这里着重强调的，不是 ubuntu 这种“入门”级别的 Linux 发行版的 wsl 配置，而是 `Arch Linux.yyds` 的配置。

详细的配置可以参考[github文档](https://github.com/yuk7/ArchWSL/blob/master/i18n/README_zh-cn.md)，这里简要说明一下，分为几个步骤：

1. 去 github 项目下载 Arch.zip，解压（文件夹位置慎重存放，因为一旦开启 `arch linux wsl`，该文件夹位置将无法移动）；
2. 双击 Arch 程序，此时会启动一个命令行窗口，完成注册表等信息的更改，等待安装完成，此时会提示 `Installation Complete!`
3. 启动 Arch 程序，可以通过 Windows Terminal（因为此时已经注册到系统当中了），也可以双击刚刚的 Arch 程序打开命令行窗口

注意，如果你已经安装了 Ubuntu 等别的发型版，会出现多个 Wsl 系统，虽然它们可以共存，但有一个会默认打开，可以通过 powershell 中的 `wslconfig` 命令修改默认版本，（如果使用 vscode wsl 连接，需要将 Arch 设置为默认的 wsl，因为该扩展暂时不支持 arch 的识别`

## 2. Arch Linux 配置

### 2.1 导入密钥

```bash
pacman-key --init && pacman-key --populate archlinux && pacman-key --refresh-keys
```

### 2.2 更新软件源

```bash
echo 'Server = https://mirrors.tuna.tsinghua.edu.cn/archlinux/$repo/os/$arch' >> /etc/pacman.d/mirrorlist
pacman -Syy
```

### 2.3 安装常用开发软件

```bash
pacman -yS openssh zsh which wget curl git aria2 bison flex texinfo gperf libtool patchutils bc zlib expat vi vim neovim rsync fd exa ripgrep tree cloc man python2 # 基本软件
pacman -yS clang llvm gdb clang-format cmake make ccls # 编译必备
pacman -yS ltrace libelf libbcc strace pahole bpftrace perf netmap trace-cmd liburing base-devel devtools # 开发必备
```

### 2.4 zsh 配置

首先将默认 shell 修改为 zsh

```bash
chsh -s `which zsh` # 上一步已经安装了 zsh & which 
```

下载 `oh-my-zsh`（若这一步中无法连接 github，可以使用 hub.fastgit.org 等国内镜像作为替代）

```bash
git clone git://github.com/robbyrussell/oh-my-zsh.git ~/.oh-my-zsh
cp ~/.oh-my-zsh/templates/zshrc.zsh-template ~/.zshrc
```

</br></br>

配置 zsh 插件

```bash
git clone https://github.com/paulirish/git-open.git $ZSH_CUSTOM/plugins/git-open
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions
```

配置 `~/.zshrc` 中的 `plugins` 配置为 `plugins=(其他的插件 git-open zsh-autosuggestions zsh-autosuggestions z extract python)`（可以参考 `~/.oh-my-zsh/plugin` 中的插件目录，自行取舍）

</br></br>

配置 zsh 主题

查看 [github wiki](https://github.com/ohmyzsh/ohmyzsh/wiki/Themes)，找到属于你自己的主题，之后在 `~/.zshrc` 中修改 `ZSH_THEME='xxx'` 中的主题名称。

</br></br>

配置 zsh alias

在 `~/.zshrc` 中配置下面信息（个人习惯），当然你也可以根据个人喜好配置属于自己的快捷键，或者你可以采用 oh-my-zsh 已经配置好的 **zsh alias**

```bash
alias ls=exa # 需要安装 exa
alias clo="git clone "
alias rm=trash   # 需要安装 trash-cli
```

### 2.5 python 配置

安装 pip3/pip2:

```bash
curl https://bootstrap.pypa.io/get-pip.py -o get-pip3.py # get pip3
python3 get-pip3.py
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple # set pip source
curl https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip2.py # get pip2 
sudo python2 get-pip2.py
```

### 2.6 nvim(vim) 配置

通过 curl 下载 `vim-plug` 这个插件管理工具（这里使用了国内的源，你也可以自己找合适的源，建议通过 hub.fastgit.org/gitclone.com/github.com.cnpmjs.org 这几个网站查找）

```bash
curl -fLo ~/.vim/autoload/plug.vim --create-dirs \
    https://ghproxy.fsou.cc/https://github.com/junegunn/vim-plug/blob/master/plug.vim
```

```bash
git clone https://github.com/hsqStephenZhang/neovim-config.git ~/.config/nvim
```

接着修改 `~/.config/nvim/init.vim` 并自行修改配置文件中的一些选项，比如插件设置，快捷键设置

打开 nvim，命令模式下输入 `:PlugInstall`，完成插件的安装

### 2.7 Rust 配置

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

然后选择 custom 安装，将默认 rust 发行版修改为 nightly，其余的按照 default 选项安装即可，

配置cargo 国内源，修改/创建 `~/.cargo/config` 为下面的内容：

```toml
[source.crates-io]
registry = "https://github.com/rust-lang/crates.io-index"

# 替换成你偏好的镜像源
replace-with = 'ustc'

# 清华大学
[source.tuna]
registry = "https://mirrors.tuna.tsinghua.edu.cn/git/crates.io-index.git"

# 中国科学技术大学
[source.ustc]
registry = "git://mirrors.ustc.edu.cn/crates.io-index"

# 上海交通大学
[source.sjtu]
registry = "https://mirrors.sjtug.sjtu.edu.cn/git/crates.io-index"

# rustcc社区
[source.rustcc]
registry = "git://crates.rustcc.cn/crates.io-index"
```

### 2.8 git 配置

配置用户名和邮箱

修改配置文件 `~/.gitconfig`，或者通过 `git config --global` 来设置 email,name 等属性，甚至可以设置 git 镜像网站，用来加速项目下载，比如在 `~/.gitconfig` 添加下面这条配置，可以加速国内 clone 速度

```bash
[url "https://hub.fastgit.org"]
    insteadOf = https://github.com
```

</br></br>

配置 ssh 密钥

通过下面这条命令，创建 ssh 的密钥，并且上传到 github，gitee，gitlab 等 git 托管网站中
`ssh-keygen -t rsa -m PEM -b 4096 -C "email"`

如果取的名字比较特殊（因为 git 会默认查找一些特殊命名的密钥文件），需要在 `~/.ssh/config` 当中指明，针对某一个 host，到底用的是哪个配置文件

```bash
Host github.com
  HostName github.com
  IdentityFile ~/.ssh/id_rsa_xxx
```

## 3. WSL 管理

如果是单个 WSL，还算比较容易管理，但如果存在多个 WSL 实例，这时候就要用到 wslconfig 这个命令来管理所有的 Linux subsystems

详细信息可以参考 [docs](https://docs.microsoft.com/en-us/windows/wsl/wsl-config) [小白版 docs](https://dowww.spencerwoo.com/4-advanced/4-3-wslconfig.html)

由于我在安装 arch linux 之前，还安装了 ubuntu 20.04，因此后者是默认的 WSL，也就用到了下面的命令修改默认的发行版本（vscode wsl 插件都可以识别）：

```powershell
wslconfig /list # 确认名称叫 'Arch'
wslconfig /s Arch # 设置默认发行版本
```
