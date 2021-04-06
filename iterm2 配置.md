# iterm2 配置
下载 [iterm2](https://iterm2.com/)
## 设置Oh-My-Zsh
[Oh-My-Zsh](https://ohmyz.sh/)网站
安装方式两种：
* curl方式：

```shell
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```
* wget方式：

```shell
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```
### 配置 brew
M1之后，不能放到 /usr/local/bin目录下了。要放到/opt目录下 所以新建/opt目录，执行:
`cd /opt && sudo chmod -R 777 ./`
授权给所有用户所有权限。
* 安装
进入/opt/路径，在终端中执行:

```shell
mkdir homebrew && curl -L https://github.com/Homebrew/brew/tarball/master | tar xz --strip 1 -C homebrew
```
在～/.zshrc 尾部，插入一行：

```.zshrc
path=('/opt/homebrew/bin' $path)
export PATH
```
#### brew 报错解决方案
##### 执行 brew install 命令长时间卡在 Updating Homebrew 的解决方法
* 使用 Alibaba 的 Homebrew 镜像源进行加速

平时我们执行 brew 命令安装软件的时候，跟以下 3 个仓库地址有关：

```shell
brew.git
homebrew-core.git
homebrew-bottles
```
通过以下操作将这 3 个仓库地址全部替换为 Alibaba 提供的地址

* 替换 / 还原 brew.git 仓库地址

```shell
# 替换成阿里巴巴的 brew.git 仓库地址:
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git

#=======================================================

# 还原为官方提供的 brew.git 仓库地址
cd "$(brew --repo)"
git remote set-url origin https://github.com/Homebrew/brew.git
```

* 替换 / 还原 homebrew-core.git 仓库地址

```shell
# 替换成阿里巴巴的 homebrew-core.git 仓库地址:
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

#=======================================================

# 还原为官方提供的 homebrew-core.git 仓库地址
cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://github.com/Homebrew/homebrew-core.git
```
*  替换 / 还原 homebrew-bottles 访问地址


```shell
# 替换成阿里巴巴的 homebrew-bottles 访问地址:
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.zshrc
source ~/.zshrc

#=======================================================

# 还原为官方提供的 homebrew-bottles 访问地址
vi ~/.zshrc
# 然后，删除 HOMEBREW_BOTTLE_DOMAIN 这一行配置
source ~/.zshrc
```
### oh-my-zsh 插件

##### 配置命令自动提示
在终端输入
```shell
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
```
安装完成后使用 vim 打开 .zshrc 配置文件，添加到plugins列表中plugins=(zsh-autosuggestion)
##### 配置语法高亮
推荐 [brew](https://brew.sh/) 安装
终端中输入
`brew install -s zsh-syntax-highlighting`
在～/.zshrc 尾部，插入一行：

```shell
ZSH_DISABLE_COMPFIX=true
source /usr/local/share/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
```
生效配置
`source ~/.zshrc`
##### 配置目录跳转
在终端输入
`brew install autojump`
在.zshrc` 文件，添加到plugins配置列表并在尾部追加如下内容：
```shell
[ -f /usr/local/etc/profile.d/autojump.sh ] && . /usr/local/etc/profile.d/autojump.sh
```
plugins中别忘了添加上autojump：

```.zshrc
plugins=(
 git
 zsh-autosuggestion
 autojump
)
```
> PS:在brew安装完会有提示输出，按照提示来配置就好

```
Add the following line to your ~/.bash_profile or ~/.zshrc file:
  [ -f /opt/homebrew/etc/profile.d/autojump.sh ] && . /opt/homebrew/etc/profile.d/autojump.sh

If you use the Fish shell then add the following line to your ~/.config/fish/config.fish:
  [ -f /opt/homebrew/share/autojump/autojump.fish ]; and source /opt/homebrew/share/autojump/autojump.fish

Restart your terminal for the settings to take effect.

zsh completions have been installed to:
  /opt/homebrew/share/zsh/site-functions
```

##### bat 代替 cat

cat 某个文件，可以在终端直接输出文件内容，bat 相比 cat 增加了行号和颜色高亮 👍

直接上个效果
![](http://qq0g5pggg.hb-bkt.clouddn.com/16157035255815.jpg)
[官网](https://github.com/sharkdp/bat)
安装在终端输入
`brew install bat`

##### you-should-use -ZSH 插件，可提醒您使用定义的别名
在终端运行

```shell
git clone https://github.com/MichaelAquilina/zsh-you-should-use.git $ ZSH_CUSTOM / plugins / you-should-use
```
在`plugins=()`中添加`you-should-use`

#### antigen 插件管理工具
[antigen](https://github.com/zsh-users/antigen)安装
`brew install -s antigen`
