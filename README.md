### webpack-backbone

#### 目录

> > webpack-backbone
> >
> > >build
> > >
> > >config
> > >
> > >src
> > >
> > >.babelrc
> > >
> > >package.json
> > >
> > >.editorconfig
> > >
> > >.eslint.json



##### build

devser.js

```js
'use strict'

const webpack = require('webpack')
const config = require('../config/index.js')
const webpackConfigDev = require('./webpack.config.dev.js')
const webpackDevServer = require('webpack-dev-server')
const chalk = require('chalk')
const openBrowser = require('opn')
const utils = require('./utils.js')
const { resolve } = utils;

const complie = webpack(webpackConfigDev)

const server = new webpackDevServer(complie, {
    contentBase: resolve('dist'),
    publicPath: '/public/',
    watchContentBase: true,
    hot: true,
    overlay:{
        errors:true
    },
    historyApiFallback:{
        index:'/public/'
    }
})

const host = config.dev.host || '127.0.0.1';
const port = config.dev.port || 3000;
server.listen(port, host, (err) => {
    if (err) {
        throw new Error('devServer start Fail!')
    } else {
        const uri = `http://${host}:${port}${config.dev.publicPath}`
        console.info(chalk.green(`devServer starting at port:[${port}] ip:[${host}],opening uri:${uri} in [${config.dev.broswer}] browser`), '\n');
        openBrowser(uri, { app: config.dev.broswer })
    }
})

```

webpack.config.base.js

```js
'use strict'

const path = require('path');
const webpack = require('webpack');
const htmlPlugin = require('html-webpack-plugin')
function resolve(dir) {
    return path.join(__dirname, '..', dir)
}
module.exports = {
    entry: {
        app: resolve('src/index.js')
    },
    output: {
        filename: '[name][hash:8].js',
        path: resolve('dist'),
    },
    resolve: {
        extensions: ['.js', '.css', '.html']
    },
    module: {
        rules: [{
            test: /\.html$/,
            use: ['html-loader']
        }, {
            test: /\.js$/,
            enforce: 'pre',
            loader: 'eslint-loader',
            include: resolve('src'),
            options: {
                formatter: require('eslint-friendly-formatter')
            }
        }, {
            test: /\.js$/,
            use: ['babel-loader'],
            exclude: [
                resolve('node_module')
            ]
        },{
            test: /\.less$/,
            use: ['style-loader','css-loader','less-loader'],
            exclude: [
                resolve('node_module')
            ]
        },{
            test: /\.css$/,
            use: ['style-loader','css-loader','less-loader']
        },{
            test: /\.(png|jpe?g|gif|svg)$/,
            loader: 'url-loader',
            options:{
                limit:10000
            }
        },{
            test: /\.(woff|woff2|eot|ttf|otf)$/,
            loader: 'url-loader',
            options:{
                limit:10000
            }
        }]
    },
    plugins: [
        new webpack.ProvidePlugin({
            '$': 'jquery',
            'jQuery':'jquery',
            '_': 'underscore',
            'Backbone': 'backbone'
        }),
        new htmlPlugin({
            template: resolve('src/index.html'),
            favicon: resolve('src/assets/icons/replace.ico'),
            thunks:['vendors','jquery','underscore','backbone','app']
        })
    ],
    optimization: {
        splitChunks: {
            cacheGroups: {
                // 第三方组件
                vendors: {
                    test: /[\\/]node_modules[\\/]/,
                    priority: -10,
                    chunks: 'initial',
                    name: 'vendors',
                    enforce: true
                }
            }
        }
    },
}
```

webpack.config.dev.js

```js
'use strict'
const webpack = require('webpack')
const webpackBaseConfig = require('./webpack.config.base.js')
const merge = require('webpack-merge')
const config = require('../config/index.js')

const webpackDevConfig = merge(webpackBaseConfig, {
    mode: 'development',
    output: {
        publicPath: config.dev.publicPath,
    },
    plugins: [
        new webpack.DefinePlugin({
            'process.env': JSON.stringify(config.dev),
        }),
        new webpack.HotModuleReplacementPlugin()
    ]
});

module.exports = webpackDevConfig;

```

webpack.config.prod.js

```js

'use strict'
const webpack = require('webpack')
const webpackBaseConfig = require('./webpack.config.base.js');
const merge = require('webpack-merge');
const cleanPlugin = require('clean-webpack-plugin');
const htmlPlugin = require('html-webpack-plugin');
const config = require('../config/index');
const { resolve } = require('./utils.js');

const webpackDevConfig = merge(webpackBaseConfig, {
    plugins: [
        //清除dist目录下
        new cleanPlugin('dist/*', {
            root: resolve()
        }),
        new webpack.DefinePlugin({
            'process.env': JSON.stringify(config.build),
        }),
        new htmlPlugin()
    ]
});

module.exports = webpackDevConfig;
```

utils.js

```js
'use strict'

const path = require('path')

module.exports = {
    resolve(dir = '') {
        return path.join(__dirname, '..', dir)
    },
}
```

##### config

index.js

```js
'use strict'

module.exports = {
    dev:{
        port:3000,
        publicPath:'/public/',
        host:'127.0.0.1',
        NODE_ENV:'"development"',
        broswer:'chrome',
    },
    build:{

    }
}
```

##### src

###### assets

###### router

router.js

```js

import mainpage from '../views/main/controller/main.js';
export default Backbone.Router.extend({
    routes: {
        '': 'loginPage',
        'login': 'loginPage',
        'main': 'mainPage'
    },
    initialize(param) {
        console.log('routerInitialize-->:', param)
    },
    before(param) {
        console.log('routerBefore-->:', param)
    },
    after(param) {
        console.log('routerAfter-->:', param)
    },
    loginPage(param) {
        console.log('loginPage-->:', param);
        new mainpage()
    },
    mainPage(param) {
        console.log('mainPage-->:', param);
        console.log(mainpage)
    }
})
```

###### views

main/controller/main.js

```js
const temlpate = require('../html/main.html');
import '../../../assets/less/main.less';
console.log('temlpate:',temlpate)
const mainpage = Backbone.View.extend({
    el: '#app',
    initialize() {
        console.log('页面初始化',_.template)
        this.template = _.template(temlpate);
        this.render();
    },
    render(){
        this.$el.html(this.template({name:'wangyy'}));
        return this;
    }
})
export default mainpage;
```



main/html/main.html

```html
<div class="mainContainer">
    <div class="header">
        <button class="btn btn-large btn-primary" type="button">Large button</button>
        <button class="btn btn-large" type="button">Large button</button>
    </div>
    <div class="content">
        <div class="left"></div>
        <div class="right"></div>
    </div>
</div>
```



index.html

```html

```

index,..js

```js

```



.babelrc

```js
{
    "presets":["@babel/preset-env"],
    "plugins":[]
}
```

.editorconfig

```js
# backbone project editorconfig
root = true
[*.js]

charset = utf-8
indent_style = space
indent_size = 4
trim_trailing_whitespace = true
insert_final_newline = true
```

.eslintrc.json

```json
{
    "parser":"babel-eslint",
    "env": {
        "browser": true,
        "es6": true,
        "node":true
    },
    "extends": "eslint:recommended",
    "parserOptions": {
        "ecmaVersion": 2015,
        "sourceType":"module"
    },
    "rules": {
        "no-unused-vars":[0],
        "no-console":[0],
        "indent": [
            "error",
            4
        ],
        "linebreak-style": [
            "error",
            "windows"
        ],
        "quotes": [
            "error",
            "single"//double
        ],
        "semi": [0]
    },
    "globals":{
        "Backbone":true,
        "_":true,
        "bootstrap":true
    }
}
```

package.json

```json
{
  "name": "webpack-backbone",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "dev": "node build/dev-server.js",
    "build": "webpack --config build/webpack.config.prod.js --mode production"
  },
  "author": "yangyangwang",
  "license": "ISC",
  "devDependencies": {
    "@babel/cli": "^7.2.3",
    "@babel/core": "^7.2.2",
    "@babel/preset-env": "^7.2.3",
    "babel-eslint": "^10.0.1",
    "babel-loader": "^8.0.5",
    "babel-preset-env": "^1.7.0",
    "bootstrap": "^3.4.0",
    "clean-webpack-plugin": "^1.0.0",
    "css-loader": "^2.1.0",
    "eslint": "^5.12.0",
    "eslint-friendly-formatter": "^4.0.1",
    "eslint-loader": "^2.1.1",
    "extract-text-webpack-plugin": "^3.0.2",
    "file-loader": "^3.0.1",
    "html-loader": "^0.5.5",
    "html-webpack-plugin": "^3.2.0",
    "less": "^3.9.0",
    "less-loader": "^4.1.0",
    "opn": "^5.4.0",
    "style-loader": "^0.23.1",
    "url-loader": "^1.1.2",
    "webpack": "^4.28.3",
    "webpack-cli": "^3.2.0",
    "webpack-dev-server": "^3.1.14",
    "webpack-merge": "^4.2.1"
  },
  "dependencies": {
    "backbone": "^1.3.3",
    "chalk": "^2.4.1",
    "jquery": "^3.3.1",
    "underscore": "^1.9.1"
  }
}

```



