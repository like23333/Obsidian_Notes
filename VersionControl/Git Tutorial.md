# Git安装配置 
---
[Git安装地址](https://git-scm.com/install/)
## Linux平台安装
各大 Linux 平台可以使用包管理器（apt-get、yum 等）进行安装。
### Debian/Ubuntu 
```bash
apt-get install git
```
### Centos/RedHat
```bash
yum -y install git-core
```
### Fedora
```bash
# yum install git #(Fedora 21 及之前的版本)
# dnf install git #(Fedora 22 及更高新版本)
```
### FreeBSD
```bash
pkg install git
```
### Alpine
```bash
apk add git
```
### 源码安装
#### Git官网源码包下载地址
><https://mirrors.edge.kernel.org/pub/software/scm/git/>
#### Github源码包克隆地址
><https://github.com/git/git>
#### 安装源码包
```bash
tar -zxf git-1.7.2.2.tar.gz
cd git-1.7.2.2
make prefix=/usr/local all
sudo make prefix=/usr/local install
```
---
## Windows 平台安装
### 安装包下载地址
><https://git-scm.com/download/win>
### Winget安装
```bash
winget install --id Git.Git -e --source winget
```
---
## Mac平台安装
### Homebrew安装
#### 安装指令
```bash
brew install git
```
#### (可选)git-gui和gitk安装
```bash
brew install git-gui
```
### 图形Git安装工具下载地址
><https://sourceforge.net/projects/git-osx-installer/>
## Git配置
Git通过**`git config`**命令配置或读取相应的工作环境变量
环境变量存放于以下三处：

- **`/etc/gitconfig`** 文件：系统中对所有用户都普遍适用的配置。若使用**`git config`**时用**`--system`**选项，读写该文件。
- `~/.gitconfig` 文件：用户目录下的配置文件只适用于该用户。若使用**`git config`**时用**`--global`**选项，读写该文件。
- 当前项目的 Git 目录中配置文件（工作目录中 `.git/config` 文件）：这里的配置仅仅针对当前项目有效。每一个级别的配置都会覆盖上层的相同配置，所以**`.git/config`**里的配置会覆盖**`/etc/gitconfig`**中的同名变量。
在 Windows 系统上，Git 会找寻用户主目录下的`.gitconfig`文件。主目录即`$HOME`变量指定的目录，一般都是**`C:\Documents and Settings\$USER`**。

此外，Git 还会尝试找寻 `/etc/gitconfig` 文件，只不过看当初 Git 装在什么目录，就以此作为根目录来定位。
### 用户信息
配置个人的用户名称和电子邮件地址，保证代码提交时个人信息被记录：
```bash
git config [--global] user.name "username"
git config [--global] user.email user@mail.com
```
若携带 **`--global`**参数，所有项目将当前设置个人信息为默认。如需要为某项目特殊设置信息，则去除**`--global`**参数
### 文本编辑器
Git默认文本编辑器一般为 Vi 或 Vim，如需更改：
```bash
git config --global core.editor "value"
```
### 差异分析工具
更换解决合并冲突的差异分析工具：
```bash
git config --global merge.tool value
```
