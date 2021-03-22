# NeoVim 配置
* [NeoVim](https://neovim.io/)官网
* 安装

```shell
brew install --HEAD luajit
brew install --HEAD neovim
```
* 插件

能在 neovim 中使用的插件管理工具有不少，这里介绍的是 [vim-plug](https://github.com/junegunn/vim-plug)。
　　安装方法很简单，只要一条命令：
　　
```shell
sh -c 'curl -fLo "${XDG_DATA_HOME:-$HOME/.local/share}"/nvim/site/autoload/plug.vim --create-dirs \
       https://raw.githubusercontent.com/junegunn/vim-plug/master/plug.vim'
```

[vim-easy-align](https://github.com/junegunn/vim-easy-align)是一个用来对齐指定符号的工具，还是比较有用的：
`Plug 'junegunn/vim-easy-align'`