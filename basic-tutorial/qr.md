# 生成小程序二维码

## 功能概述

本文对应实现例子为[tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)中的 `小程序码` 功能，可以生成包括小程序码、小程序二维码。

## 体验功能

<p align="center">
    <img src="https://main.qcloudimg.com/raw/f36ab01f3fd9e0f899c879f71d11fdff.png" width="500px">
    <p align="center">扫码体验</p>
</p>

## 体验 DEMO

本章的案例代码，是在 [tcb-demo-basic](https://github.com/TencentCloudBase/tcb-demo-basic)。

1.新建 `config` 目录，复制 `example.js` 到 `config` 下，并改名为 `index.js`，打开并补齐如下字段内容
```js
module.exports = {
  appId: '', // 微信小程序 AppID
  secret: '', // 微信小程序 AppSecret
  bucketPrefix: '' // 云存储地址前缀，形如cloud://tcb-demo-u8s8ec.7463-tcb-demo-u8s8ec/
};
```
其中 `bucketPrefix` 为文件存储 `fildID` 的前半部分。比如下面这个文件：

<p align="center">
    <img style="width:650px;" src="https://main.qcloudimg.com/raw/156c91171d059f1ab557949725c8ad7d.png">
</p>

其 `fileID` 为：`cloud://tcb-demo-u8s8ec.8888-tcb-demo-u8s8ec/qr/square.png`，则 `bucketPrefix` 为 `cloud://tcb-demo-u8s8ec.8888-tcb-demo-u8s8ec/`

2.到控制台文件存储中顶级目录中创建一个名为 `qr` 的目录

3.上传云函数到服务器并部署服务

至此，服务就可以跑起来了。编译后，在列表中选择 `小程序码生成` 进入页面。

## 源码介绍

### 二维码API

小程序有了云能力之后，我们来了解一下借助云能力的二维码生成体验是怎样的。

在开始了解本例子之前，先来了解一下小程序官方提供的生成二维码api：
> 1. createWXAQRCode
>  `获取小程序二维码，适用于需要的码数量较少的业务场景。通过该接口生成的小程序码，永久有效，有数量限制 `
> 2. getWXACode
>  `获取小程序码，适用于需要的码数量较少的业务场景。通过该接口生成的小程序码，永久有效，有数量限制`
> 3. getWXACodeUnlimit
>  `获取小程序码，适用于需要的码数量极多的业务场景。通过该接口生成的小程序码，永久有效，数量暂无限制`

详细的说明请移步[官网](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/qr-code/getWXACode.html)查看


### 生成流程

按以往传统的开发模式，需要额外搭建服务器部署一个后台服务，然后通过这个后台服务去请求API，拿到二维码图片之后再返回给小程序使用。

那么在有了云能力之后，我们就可以借助云函数来代替后台发起API请求，再借助云文件存储来保存二维码图片，最后页面通过访问云存储文件的CDN地址来展示。整个流程如下：

<p align="left">
    <img style="background:white;padding:20px;" src="https://main.qcloudimg.com/raw/7147d3349c92090ad6fac59863936a86.png" width="483px">
</p>

为了更方便地在云函数里使用二维码服务，我们在[wx-js-utils](https://www.npmjs.com/package/wx-js-utils)这个npm包里封装了上面官方提供的3个api。

### 主要代码介绍

二维码生成服务是一个非常独立的小功能，逻辑也很简单。如果使用我们提供的[wx-js-utils](https://www.npmjs.com/package/wx-js-utils)npm包的话，只需要简单几行代码就可以实现了。
下面是云函数实现的核心代码：
```js
  const {
    appId,
    secret,
    bucketPrefix
  } = require('./config');
  const {
    WXMINIUser,
    WXMINIQR
  } = require('wx-js-utils');

  //获取access_token
  let wXMINIUser = new WXMINIUser({
    appId,
    secret
  });
  let access_token = await wXMINIUser.getAccessToken();

  //生成方形的二维码
  let wXMINIQR = new WXMINIQR();
  let qrResult = await wXMINIQR.getQR({
    scene: '?code=123',
    access_token,
    path,
    is_hyaline: true
  });

  //上传云存储，返回结果是供小程序使用的fileID
  return await cloud.uploadFile({
    cloudPath: fileID,
    fileContent: qrResult
  })
```

页面调用代码如下：
```js
  wx.cloud.callFunction({
    name: 'wxaqrcode',
    data: {}
  }).then(res => {
    const result = res.result

    this.setData({ qrSource: result.fileID})
  })
```

### 小程序码的分类

这里有必要区分一下小程序二维码和小程序码这两个名词
1. 小程序二维码。方形的最常见的二维码
2. 小程序码。最大的特点是圆形的。
效果分别如下图所示：

<p align="center">
    <img style="width:250px;" src="https://main.qcloudimg.com/raw/307285cc762d10add331631a5229f7e1.png">
    <img style="width:250px;" src="https://main.qcloudimg.com/raw/f2115ab956a752c3a76bad96a7957c3e.png">
</p>

官方提供的三个API中，第一个是生成小程序二维码，即方形的；第二、第三个是生成小程序码的，即圆形的。


