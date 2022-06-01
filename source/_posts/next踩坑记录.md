---
title: next踩坑记录
date: 2022-04-21 00:54:14
categories: hexo-next5.14踩坑记录 
---



# TOC 锚点 undefined

**步骤一:**

安装 markdown-it-named-headings 插件

这个插件是另外一个大佬针对这个 bug 做的一个修复

首先安装它：

```shell
npm install markdown-it-named-headings --save
```

然后进入你项目的根目录，修改根目录下 `node_modules\hexo-renderer-markdown-it\lib\renderer.js` 文件，在 `renderer.js` 中添加一行以使用此插件：

```shell
parser.use(require('markdown-it-named-headings'))
```

具体修改文件详情：

```
'use strict';

const MarkdownIt = require('markdown-it');

class Renderer {

  /**
   * constructor
   *
   * @param {*} hexo context of hexo
   */
  constructor(hexo) {
    this.hexo = hexo;

    let { markdown } = hexo.config;

    // Temporary backward compatibility
    if (typeof markdown === 'string') {
      markdown = {
        preset: markdown
      };
      hexo.log.warn(`Deprecated config detected. Please use\n\nmarkdown:\n  preset: ${markdown.preset}\n\nSee https://github.com/hexojs/hexo-renderer-markdown-it#options`);
    }

    const { preset, render, enable_rules, disable_rules, plugins, anchors } = markdown;
    this.parser = new MarkdownIt(preset, render);

    if (enable_rules) {
      this.parser.enable(enable_rules);
    }

    if (disable_rules) {
      this.parser.disable(disable_rules);
    }
	this.parser.use(require('markdown-it-named-headings'));

    if (plugins) {
      this.parser = plugins.reduce((parser, pugs) => {
        if (pugs instanceof Object && pugs.name) {
          return parser.use(require(pugs.name), pugs.options);
        }
        return parser.use(require(pugs));
      }, this.parser);
    }

    if (anchors) {
      this.parser.use(require('./anchors'), anchors);
    }
  }

  render(data, options) {
    this.hexo.execFilterSync('markdown-it:renderer', this.parser, { context: this });
    return this.parser.render(data.text);
  }
}

module.exports = Renderer;
```

