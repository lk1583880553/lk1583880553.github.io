---
layout:     post
title:      mini-css-extract-plugin
subtitle:   webpack4.0之css提取插件
date:       2018-10-11
author:     Mr Kang
header-img: img/post-bg-building.jfif
catalog: true
tags:
    - webpack 4.0
    - css提取插件配置
    - mini-css-extract-plugin
---


## 前言

## 关于mini-css-extract-plugin的配置

   将CSS提取为独立的文件的插件，对每个包含css的js文件都会创建一个CSS文件，支持按需加载css和sourceMap

   只能用在webpack4中，对比另一个插件 extract-text-webpack-plugin有点:
   
- 异步加载
- 不重复编译，性能更好
- 更容易使用
- 只针对CSS

目前缺失功能，HMR。

安装：

```
yarn add -D mini-css-extract-plugin

```

使用:


```
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

module.exports = {
  plugins: [
    new MiniCssExtractPlugin({
      // 类似 webpackOptions.output里面的配置 可以忽略
      filename: '[name].css',
      chunkFilename: '[id].css',
    }),
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          {
            loader: MiniCssExtractPlugin.loader,
            options: {
              // 这里可以指定一个 publicPath
              // 默认使用 webpackOptions.output中的publicPath
              publicPath: '../'
            },
          },
          'css-loader',
        ],
      }
    ]
  }
}

```

高级配置示例:

这个插件应该只用在 production 配置中，并且在loaders链中不使用 style-loader, 特别是在开发中使用HMR，因为这个插件暂时不支持HMR。

```
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const devMode = process.env.NODE_ENV !== 'production';

module.exports = {
  plugins: [
    new MiniCssExtractPlugin({
      filename: devMode ? '[name].css' : '[name].[hash].css',
      chunkFilename: devMode ? '[id].css' : '[id].[hash].css',
    })
  ],
  module: {
    rules: [
      {
        test: /\.(sa|sc|c)ss$/,
        use: [
          devMode ? 'style-loader' : MiniCssExtractPlugin.loader,
          'css-loader',
          'postcss-loader',
          'sass-loader',
        ],
      }
    ]
  }
}

```

## production 阶段进行压缩

webpack5可能会内置CSS 压缩器，webpack4需要自己使用压缩器，可以使用 optimize-css-assets-webpack-plugin 插件。 设置 optimization.minimizer 覆盖webpack默认提供的，确保也指定一个JS压缩器

```
const UglifyJsPlugin = require('uglifyjs-webpack-plugin');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
const OptimizeCSSAssetsPlugin = require('optimize-css-assets-webpack-plugin');

module.exports = {
  optimization: {
    minimizer: [
      new UglifyJsPlugin({
        cache: true,
        parallel: true,
        sourcMap: true
      }),
      new OptimizeCSSAssetsPlugin({}),
    ],
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: "[name].css",
      chunkFilename: "[id].css"
    }),
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader'
        ]
      }
    ]
  }
}

```
>将所有的CSS提取到一个文件中

和 extract-text-webpack-plugin 类似，可以使用 optimization.splitChunks.cacheGroups 将css提取到一个CSS中

```
const MiniCssExtractPlugin = require('mini-css-extract-plugin');
module.exports = {
  optimization: {
    splitChunks: {
      cacheGroups: {
        styles: {
          name: 'styles',
          test: /\.css$/,
          chunks: 'all',
          enforce: true,
        },
      },
    },
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: '[name].css',
    }),
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          'css-loader'
        ]
      }
    ]
  }
}

```

>根据 entry提取CSS

可以根据webpack 的entry name来提取CSS，这对你动态引入路由，却想依据entry保存打包的CSS的情况十分有用。这也解决了ExtractTextPlugin中CSS重复的问题

```
const path = require('path');
const MiniCssExtractPlugin = require('mini-css-extract-plugin');

function recursiveIssuer(m) {
  if (m.issuer) {
    return recursiveIssuer(m.issuer);
  } else if (m.name) {
    return m.name;
  } else {
    return false;
  }
}

module.exports = {
  entry: {
    foo: path.resolve(__dirname, 'src/foo'),
    bar: path.resolve(__dirname, 'src/bar')
  },
  optimization: {
    splitChunks: {
      cacheGroups: {
        fooStyles: {
          name: 'foo',
          test: (m,c,entry = 'foo') => m.constructor.name === 'CssModule' && recursiveIssuer(m) === entry,
          chunks: 'all',
          enforce: true
        },
        barStyles: {
          name: 'bar',
          test: (m,c,entry = 'bar') => m.constructor.name === 'CssModule' && recursiveIssuer(m) === entry,
          chunks: 'all',
          enforce: true
        }
      }
    }
  },
  plugins: [
    new MiniCssExtractPlugin({
      filename: "[name].css",
    })
  ],
  module: {
    rules: [
      {
        test: /\.css$/,
        use: [
          MiniCssExtractPlugin.loader,
          "css-loader"
        ]
      }
    ]
  }
}

```