---
title: electron 入门学习
categories: 大前端
tags: electron
date: 2017-12-16 14:48:05
---

![](https://raw.githubusercontent.com/yuhanle/blogbag/master/uploads/images/electron.jpeg)

## 简介

目前，使用前端技术开发桌面应用越来越成熟，所以前端同学也可以参与桌面应用的开发。目前类似的工具有electron，NW.js等。刚好最近在学习Vue，简单做了些入门页面，B/S 的方式虽然能够跨平台使用，但是对于不折腾会死的周末来说，尝试着踩坑才是最充实的。

这里我们简单介绍下 [Electron](https://electron.org.cn/)的使用。

Electron 中文网上对于他的功能描述的很直观，也很吸引众多开发者的探索。


<center> 
<h4>比你想象的更容易</h4>
</center>

> 如果您能够搭建一个网站，那么您就可以搭建一个桌面应用。 Electron是这样一个框架，它可以帮助您使用JavaScript/Html/Css等网站相关技术，非常快速而容易地搭建一个原生应用。这样，您就可以聚焦于您的业务系统本身了，然后把剩下的那些难题交给我们好了。

## 如何开发

简单来说，Electron 是基于Node.js 和Chromium 做的一个工具，通过将前端页面加壳的方式，实现桌面开发，并且支持跨平台。

他是一个工具，除了官方文档上的基础配置，开发相关的只是主要是是前端业务相关，对于工具的使用看文档，论坛交流和更多Demo 的学习足够。

### 项目结构

一个最简单的Electron 项目包含三个文件，package.json, index.html, main,js。

```
|-- package.json Node.js 项目的配置文件
|-- index.html 桌面应用的业务
|-- main.js 应用的启动入口文件
```

其中，核心的文件是index.html 和main.js。

回到最近写的小工具，目录结构如下

```
|-- voicepkgtool 
|	|-- config webpack配置文件
|	|	|-- dev.env.js 
|	|	|-- index.js
|	|-- dist 发布
|	|-- node_modules
|	|-- samples Node.js Server
|	|-- src 源码目录
|	|-- main.js Electron 应用入口
|	|-- package.json	配置文件

```

## 安装使用

推荐查阅官网的教程[Electron-vue](https://electron.org.cn/vue/index.html)

我们这个简单的Web 项目部署以后，希望可以通过Electron 打包跨屏体运行，所以只需要简单几步配置就搞定！

### 安装Electron

在使用前，需要先安装相关模块，然后在项目中引入使用

```
// 核心模块
npm install electron -s
// 打包模块 | 也可以使用electron-builder
npm install electron-packager -s
```

### 入口文件

在执行Electron 相关命令并启动应用时，需要一个入口文件，就是上面提到的main.js，文件包含整个应用的生命周期回调，同时也用来存放应用相关配置信息。

以下便是我们为目前这个小项目编辑的main.js 文件，API 可以参考官方教程。

注释也可以清楚的了解每一行的含义

```
const electron = require('electron')
// Module to control application life.
const app = electron.app
// Module to create native browser window.
const BrowserWindow = electron.BrowserWindow

const path = require('path')
const url = require('url')

// Keep a global reference of the window object, if you don't, the window will
// be closed automatically when the JavaScript object is garbage collected.
let mainWindow

function createWindow () {
  // Create the browser window.
  mainWindow = new BrowserWindow({width: 800, height: 600})

  // and load the index.html of the app.
  mainWindow.loadURL(url.format({
    pathname: path.join(__dirname, 'dist/index.html'),
    protocol: 'file:',
    slashes: true
  }))

  // Open the DevTools.
  // mainWindow.webContents.openDevTools()

  // Emitted when the window is closed.
  mainWindow.on('closed', function () {
    // Dereference the window object, usually you would store windows
    // in an array if your app supports multi windows, this is the time
    // when you should delete the corresponding element.
    mainWindow = null
  })
}

// This method will be called when Electron has finished
// initialization and is ready to create browser windows.
// Some APIs can only be used after this event occurs.
app.on('ready', createWindow)

// Quit when all windows are closed.
app.on('window-all-closed', function () {
  // On OS X it is common for applications and their menu bar
  // to stay active until the user quits explicitly with Cmd + Q
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

app.on('activate', function () {
  // On OS X it's common to re-create a window in the app when the
  // dock icon is clicked and there are no other windows open.
  if (mainWindow === null) {
    createWindow()
  }
})

// In this file you can include the rest of your app's specific main process
// code. You can also put them in separate files and require them here.

```

### 编辑脚本

入口文件的路径，我们需要通过键值对的方式，新增在package.json 文件里，同时可以在Scripts 里添加启动、编译以及打包的脚本。

```
{
  "name": "voicepkgtool",
  "description": "A Vue.js project",
  "author": "shibo.wen@ti-link.com.cn",
  "main": "main.js",	// 入口文件路径 不可缺少 否则执行时会报错
  "scripts": {
    "start": "electron .",		// 启动应用
    "dev": "webpack-dev-server --inline --hot --env.dev",
    "build": "rimraf dist && webpack -p --progress --hide-modules",
    "build:all": "electron-packager . voicepkgtool --platform=all --arch=all -version=0.0.1 --icon=app.ico --out=Staging --version-string.ProductName=Telar Converter --version-string.ProductVersion=0.5.0",
    "package": "electron-packager ./ voicepkgtool --all --out ./ --version 0.0.1 --overwrite"
  },
  "dependencies": {
    "axios": "^0.17.1",
    "checksum": "^0.1.1",
    "electron": "^1.7.9",
    "electron-packager": "^10.1.0",
    "element-ui": "^2.0.0",
    "file-saver": "^1.3.3",
    "jszip": "^3.1.5",
    "qs": "^6.5.1",
    "querystring": "^0.2.0",
    "vue": "^2.5.2",
    "vue-simple-uploader": "^0.3.0",
    "webpack-merge": "^4.1.1"
  },
  "engines": {
    "node": ">=6"
  },
  "devDependencies": {
    "autoprefixer": "^6.6.0",
    "babel-core": "^6.24.1",
    "babel-loader": "^6.4.0",
    "babel-preset-vue-app": "^1.2.0",
    "css-loader": "^0.27.0",
    "file-loader": "^0.10.1",
    "html-webpack-plugin": "^2.24.1",
    "postcss-loader": "^1.3.3",
    "rimraf": "^2.5.4",
    "style-loader": "^0.13.2",
    "url-loader": "^0.5.8",
    "vue-loader": "^13.3.0",
    "vue-template-compiler": "^2.5.2",
    "webpack": "^2.4.1",
    "webpack-dev-server": "^2.4.2"
  }
}

```

接着我们只需要执行

```
npm run start
```

一个跨屏台的应用就展现在桌面上。

### 构建打包

进阶教程参考[Electron构建打包](https://electron.org.cn/build.html)

```
electron-packager . --overwrite --platform=darwin --arch=x64 --out=out --icon=assets/app-icon/mac/app.icns --osx-sign.identity='Developer ID Application: GitHub' --extend-info=assets/mac/info.plist

electron-packager . --overwrite --platform=win32 --arch=ia32 --out=out --icon=assets/app-icon/win/app.ico

electron-packager . --overwrite --platform=linux --arch=x64 --out=out
```

## 注意事项

### Electron 不支持XP 系统

如果有考虑在XP 系统上使用，不用考虑了

参考：[https://js3.org/question/43](https://js3.org/question/43)

### 安装包大小

一般情况下，通过packager或者builder打包完毕后，exe、dll、asr等文件总和的大小为100M左右。而通过builder制作的nsis安装包，一般为32M左右。通过innosetup生成的安装包，一般为31M左右。总体来说，体积较大。但是您通过一系列的手段可以有效的减少它的体积，到一个可接受的范围。

### 调试模式

浏览器窗口的开发工具仅能调试渲染器的进程脚本（比如 web 页面）。为了提供一个可以调试主进程 的方法，Electron 提供了 --debug 和 --debug-brk 开关。

使用如下的命令行开关来调试 Electron 的主进程：

--debug=[port]

当这个开关用于 Electron 时，它将会监听 V8 引擎中有关 port 的调试器协议信息。 默认的 port 是 5858。

--debug-brk=[port]

就像 --debug 一样，但是会在第一行暂停脚本运行。

## Electron 常见问题

如何解决下载源的时候timeout问题
对于国内的网络用户，请切换npm源为淘宝的cnpm源，切换方法为：

```
npm config set registry https://registry.npm.taobao.org/
npm config set electron_mirror http://npm.taobao.org/mirrors/electron/
```

另外，对于文件构建时的timeout问题，需要您下载对应的文件到本地缓存文件夹。具体的下载切换办法，可以点击这里查看：[https://js3.org/article/7](https://js3.org/article/7)

更多问题参考：[Electron 常见问题](https://electron.org.cn/doc/faq.html)
