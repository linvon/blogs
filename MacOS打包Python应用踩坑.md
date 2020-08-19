---
title: "MacOS 打包 Python 应用踩坑"
date: "2020-08-04"
categories: ["学习笔记"]
---

首先，在 MacOS 上打包用的是 py2app，建议使用 Python3 来运行，2.x 版本会遇到一些错误，比较难处理。

`pip install py2app` 
或者直接指定解释器
`python3 -m pip install py2app`

之后进入要打包的文件目录，比如 main.py，执行
`py2applet --make-setup main.py`
该步骤生成一个 setup.py 文件，之后运行
`python3  setup.py  py2app`

在该步骤如果使用 Python2.x 运行可能会有以下错误，因此建议使用 Python3 来运行，若还有错误可以尝试使用 sudo

> creating application bundle: win *** error: [Errno 1] Operation not permitted:   '/Users/a/studio/a/a/dist/main.app/Contents/MacOS/win'

完成打包后，在 dist 文件夹下有生成的 App，双击即可运行，如果运行报错，可以右键显示包内容，进入 `mian.app/Contents/MacOS`，找到你的 App 对应的可执行文件，进入 Terminal 调试

如果是出现依赖问题，可能是系统环境下打包没有找到对应的依赖，一般有两种方法解决：

- 使用 Anaconda 等包管理软件，整理好系统环境
- 但实际上我更推荐使用软连接，找到你使用的包的绝对路径，比如 xxx/json，执行 ln -s xxx/json json，将该包路径软连接到当前目录下即可 


