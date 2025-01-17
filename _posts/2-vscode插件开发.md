---
layout: post
title: vscode插件开发
subtitle: vscode的插件开发
date: 2020-12-09
author: Qi
header-img: img/404-bg.jpg
catalog: true
tags:
  - vscode
---

# 环境安装

```
npm install -g yo generator-code

```

- 然后运行 yo code 就能生成项目文件
- 按下 F5 能新建窗口进行调试
- 用 Common+shift+P 输入启动命令就能测试

# 入口文件 extension.ts

extension.ts 是入口文件，主要有两个生命周期

- active: 插件唤醒的时候执行
- deactivate: 插件卸载的时候执行
- 在 active 的时候给命令绑定一个函数

```
//当运行helloword.helloWorld的时候可以执行下面的函数
let disposable = vscode.commands.registerCommand('helloword.helloWorld', () => {

 vscode.window.showErrorMessage(new Date().toLocaleString());
});


```

# package.json 相关字段说明

- activationEvents 运行哪些命令唤醒插件
- contributes.commands 命令的描述

```
 "commands": [{
        "command": "helloword.helloWorld",//命令
        "title": "Clock"//命令的名称，运行Clock就能运行上述该命令
    }]

```

- menu 控制命令在何时显示

有时候我们希望命令只在特定场景下作展示，比如只在特定语言或指定配置下展示命令给用户

```

   "menus": {
            "commandPalette": [{
                "command": "extension.sayHello",
                "when": "editorLangId == markdown"//制作markDown时显示
            }]
        }


```



```
const HtmlWebpackPlugin = require('html-webpack-plugin');
const pluginName = 'HtmlAfterPlugin';

const assetsHelp = (data) => {
  const js = [];
  const css = [];
  const getAssetsName = {
    css: (item) => `<link rel="stylesheet" href="${item}">`,
    js: (item) => `<script class="lazyload-js" src="${item}"></script>`,
  };
  for (let jsitem of data.js) {
    js.push(getAssetsName.js(jsitem));
  }
  for (let cssitem of data.css) {
    css.push(getAssetsName.css(cssitem));
  }
  return {
    js,
    css,
  };
};

class HtmlAfterPlugin {
  constructor() {
    this.jsarr = [];
    this.cssarr = [];
  }
  apply(compiler) {
    // compilation 创建之后执行。
    compiler.hooks.compilation.tap(pluginName, (compilation) => {
      // 在标签生成之前
      HtmlWebpackPlugin.getHooks(compilation).beforeAssetTagGeneration.tapAsync(
        pluginName,
        (htmlPligunData, cb) => {
          const { js, css } = assetsHelp(htmlPligunData.assets);
          this.cssarr = css;
          this.jsarr = js;
          cb(null, htmlPligunData);
        }
      );
      // 插入标签内容
      HtmlWebpackPlugin.getHooks(compilation).beforeEmit.tapAsync(
        pluginName,
        (data, cb) => {
          let _html = data.html;
          _html = _html.replace('<!--injectjs-->', this.jsarr.join(''));
          _html = _html.replace('<!--injectcss-->', this.cssarr.join(''));
          _html = _html.replace(/@components/g, '../../../components');
          _html = _html.replace(/@layouts/g, '../../layouts');
          data.html = _html;
          cb(null, data);
        }
      );
    });
  }
}

module.exports = HtmlAfterPlugin;


```

