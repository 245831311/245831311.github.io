---
title: st3使用配置
date: 2017-01-22 11:16:39
categories: 
- 笔记
- 配置
tags:
- 笔记
---

---
该编文章主要是记录下使用ST3和在ST3编辑markdown
---

### 1. 安装ST3
安装过程及其简单，直接go to anything 就可以了。

### 2. 下载支持插件
1. 下载package control插件(网上一堆信息，不做详细说明)
主要参考：[https://www.cnblogs.com/luoshupeng/archive/2013/09/09/3310777.html/](https://www.cnblogs.com/luoshupeng/archive/2013/09/09/3310777.html/)
2. 下载markdownEditing(编辑markdown,提供快捷键)
*流程:ctrl+Shift+P->install package->下载markdownEditing插件->在multimarkdown setting user下设置*
```
{
      "enable_table_editor": true,
      "highlight_line": true,
      "line_numbers": true,   // 显示行号
      "tab_size": 4,          // tab宽度
      "wrap_width": 300,
      "word_wrap": true,      // 自动换行
      "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Dark.tmTheme",
      // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor.tmTheme",
      // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Dark.tmTheme",
      // "color_scheme": "Packages/MarkdownEditing/MarkdownEditor-Yellow.tmTheme",
      "mde.keep_centered": true,// 可以保持你正在编辑的行始终处于屏幕的中间
      "extensions":
      [
          "mmd",
          "md"
      ]
 }
```
3. 下载Markdown Preview或MarkdownLivePreview.(预览markdown)
_markdown preview下载流程:ctrl+Shift+P->install package->输入插件名称markdown preview->key bindings插入下面代码_
```
{ "keys": ["alt+m"], "command": "markdown_preview", "args": {"target": "browser", "parser":"markdown"} },   
```


### 3. 记录st3常用的快捷键
```java
Ctrl+← 向左单位性地移动光标，快速移动光标。
Ctrl+→ 向右单位性地移动光标，快速移动光标。
shift+↑ 向上选中行。
shift+↓ 向下选中行。
Ctrl+Shift+K 删除整行。
Ctrl+/ 注释单行。
Ctrl+Shift+/ 注释多行。
Ctrl+K+U 转换大写。
Ctrl+K+L 转换小写。
Ctrl+F 打开底部搜索框，查找关键字。
Ctrl+P 打开搜索框。提供：1、输入当前项目中的文件名，快速搜索文件，2、Ctrl+G功能，3、Ctrl+R功能，4、Ctrl+：功能
Ctrl+G 自动带：，输入数字跳转到该行代码
Ctrl+R 自动带@，查找文件中的函数名
Ctrl+： 自动带#，查找文件中的变量名、属性名等
Ctrl+Shift+P 打开命令框。场景栗子：打开命名框，输入关键字，调用sublime text或插件的功能，例如使用package安装插件。
Esc 退出搜索框，命令框等。
Alt+Shift+1 窗口分屏1-4(水平),5（等分）,89（垂直）
Ctrl+K+B 开启/关闭侧边栏。
```

### 4. markdownEditing常用的快捷键
```java
Ctrl+Win+V 选中的内容将自动转换为行内式超链接，链接到剪贴板中的内容
Ctrl+Win+R 选中的内容将自动转换为参考式超链接，链接到剪贴板中的内容
Ctrl+Alt+R 弹出提示框插入一个参考式超链接，在提示框中输入链接内容和定义参考ID[^3]
Ctrl+Win+K 插入一个标准的行内式超链接
Win+Shift+K 插入一个标准的行内式图片（此快捷键可能与输入法有冲突）
Ctrl+1 至 Ctrl+6 插入一级至六级标题
FN+Alt+i 选中的内容转换为斜体
FN+Alt+b 选中的内容转换为粗体[^1]
Ctrl+Shift+6 自动插入一个脚注，并跳转到该脚注的定义中。
Alt+Shift+F 查找没有定义的脚注并自动添加其定义链接
Alt+Shift+G 查找没有定义的参考式超链接并自动添加其定义链接
Ctrl+Alt+S 脚注排序
Ctrl+Shift+. 缩进当前内容
Ctrl+Shift+, 提前当前内容
```

__PS__ 参考:
1. 快捷键总结:[http://blog.csdn.net/u012771929/article/details/30030249](http://blog.csdn.net/u012771929/article/details/30030249)
2. st3使用配置:[http://www.jianshu.com/p/62241c7ecec9](http://www.jianshu.com/p/62241c7ecec9)
ssda