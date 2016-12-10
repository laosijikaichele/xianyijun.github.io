---
title: VSCode The mode has failed while tokenizing the input问题修复
date: 2016-07-09 21:29:23
tags: 开发环境
---

# ** VSCode The mode has failed while tokenizing the input问题修复** #

- - -


- 问题描述：
	将VS Code从1.0升级到1.3的时候，重启VSCode之后，在编辑器顶部The mode has failed while tokenizing the input错误
代码无法高亮，将VS code卸载之后重新安装同样出现相同问题。

- 开发环境
	- os Linux Mint 17.0
	- VS Code 1.30

- 解决方法
	由于将软件卸载之后重新安装，也出现了相同问题，而同学使用相同的安装包确无法出现该问题，即不是软件安装包的问题，突然想到自己在升级VS Code的时候没有将1.0版本的卸载，而是直接安装1.3版本，可能是没有完全卸载清除的问题。
    因此决定将VS Code的全部相关配置删除。

- - -


>	- sudo apt-get remove --purge code
>	- sudo rm -rf ~/.vscode
>	- sudo rm -rf /usr/share/code

- 问题解决
	将之前所有VS code相关的文件卸载删除，重新安装之后，错误信息没有显示，问题解决。
