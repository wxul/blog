---
title: Cmder安装记录
date: 2017-07-26 16:45:13
tags: ['terminal']
categories: [开发工具]
---

### 下载

<a href="http://cmder.net/" target="_blank">Cmder下载</a>

建议下载full版本

### 安装

解压后把目录加入到PATH即可,使用管理员权限打开cmd运行`cmder /register all`可以将cmder添加到右键菜单

### 其它坑

- 如果不喜欢默认的提示符`λ`,可以找到`{安装目录}/vendor/clink.lua`并编辑.修改`local cmder_prompt = "\x1b[1;32;40m{cwd} {git}{hg} \n\x1b[1;30;40m{lamb} \x1b[0m"`中的`{lamb}`为`$`,再修改`clink.prompt.value = string.gsub(cmder_prompt, "{lamb}", lambda)`中的`{lamb}`为`$$`  
> 不同版本处理方式不同

- alias别名可能会失效,打开cmd然后右击左上角小图标选属性,勾选`使用旧版本控制台`,自定义alias配置文件在`{安装目录}/config/user-aliases.cmd`

- 文字重复网上说的很多,`win+alt+P`打开设置,选`Main`然后取消勾选右边的`Monospace`

- 将所有cmd提示都设为yes,可以修改`{安装目录}/config/settings`中的`terminate_autoanswer`设为1