---
layout:     post
title:      "ST使用记录"
subtitle:   "平常使用sublime text3 的一些记录"
date:       2017-03-31 00:00:00
author:     "zfl"
header-img: "img/post-st-use-guide.png"
header-mask: 0.6
catalog:    true
tags:
    - python
    - sublime text
    - 工具
---

# ST使用指南
> 经常使用这个神器，有些快捷键和插件使用过很溜，做个记录。
## 编辑区常用快捷键
* Ctrl+Enter 光标下方插入新的一行
* Ctrl+Shift+Enter 光标上方插入新的一行
* Ctrl + ←/→进行逐词移动
* Ctrl + Shift + ←/→进行逐词选择
* Ctrl + ↑/↓移动当前显示区域
* Ctrl + Shift + ↑/↓移动当前行
* Ctrl + D 选择当前光标所在的词并高亮该词所有出现的位置，再次Ctrl + D选择该词出现的下一个位置
* Ctrl + Shift + L可以将当前选中区域打散，然后进行同时编辑
* Ctrl + J可以把当前选中区域合并为一行
* Ctrl + H 替换
* Ctrl + p 搜索文件
* Ctrl + g 搜索行号
* Ctrl + b 运行python文件
* Ctrl + PageUp\PageDown 切换tab
## 常用插件
### Markdown相关
* Markdown Preview 用来在浏览器中浏览md文件，快捷键Ctrl + Alt + o直接打开
* Markdown Editing 用来编辑md文件
### python相关
* SublimeREPL 安装后才可以支持python，这个插件支持多种语言
* Anaconda python**究极神器**，是的ST变成了一款真正意义上的**ide**，自动补全，跳转到定义，跳转到方法，找到调用此方法的，还可以支持PEP8格式化，由于使用了AutoPep8，这个功能被我禁止，为了避免代码中出现白框，在setting中加入{"anaconda_linting": false}
* Djaneiro 提供对django的支持
* AutoPep8 将python代码按照PEP8格式化,设置之后save自动格式化
{
    "format_on_save": true,
}
* SublimeLinter 可以检测到所写代码的语法错误,并高亮显示错误，但是对于python来说，首先要有flake8的环境，然后安装Sublime​Linter-flake​8这个插件，其他的也需要先安装环境
### 其他
* Package Control 必装，用来安装\更新\删除sublime插件，快捷键ctrl+shift+p,输入pci
* Terminal Ctrl+Shift+t调出windows的powershell
* Side Bar 扩展侧边栏 可以删除放到回收站，也可以浏览器打开，很强大
* Bracket Highlighter  高亮括号，本身的太不显眼了
* AutoFileName 自动补全文件、文件夹很方便

