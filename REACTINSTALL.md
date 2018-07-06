title: ReactJS安装问题
author: qing
date: 2018-07-05
description: ReactJS安装问题
tags:
category:

# ReactJS安装问题

**不能在tmux中运行下面命令**

    npx create-react-app my-app

会出现如下错误：

    Error: Cannot find module '/root/go/src/github.com/winglq/site/src/frontend/smarttag/smarttag/node_modules/uglifyjs-webpack-plugin/lib/post_install.js'
        at Function.Module._resolveFilename (internal/modules/cjs/loader.js:581:15)
        at Function.Module._load (internal/modules/cjs/loader.js:507:25)
        at Function.Module.runMain (internal/modules/cjs/loader.js:742:12)
        at startup (internal/bootstrap/node.js:236:19)
        at bootstrapNodeJSCore (internal/bootstrap/node.js:560:3)
    npm ERR! code ELIFECYCLE
    npm ERR! errno 1
    npm ERR! uglifyjs-webpack-plugin@0.4.6 postinstall: `node lib/post_install.js`
    npm ERR! Exit status 1
    npm ERR!
    npm ERR! Failed at the uglifyjs-webpack-plugin@0.4.6 postinstall script.
    npm ERR! This is probably not a problem with npm. There is likely additional logging output above.
