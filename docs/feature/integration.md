---
description: Fileview文件预览服务集成指南，支持HTTP API/iframe/SDK等多种方式快速集成，助力企业文档管理系统搭建
---
# 服务集成

安装并启动成功后，简单几步就可以集成文件预览服务到自己的业务系统。

## 预览文件

Fileview 支持两种传参方式，你可以任意选择其中一种

#### 使用 query 参数

```js
// 构造参数
const url = encodeURIComponent("https://mydomain.com/myfiles/sample.docx"); // 网络文件地址，支持 http/https/ftp
const fileName = encodeURIComponent("sample.docx"); // 真实文件名，用作文件类型判断，如果文件地址中没有正确文件后缀，则必须手动传递
const displayName = encodeURIComponent("网络示例文档"); // 用于标题栏等展示的文件名，非必需

// 构造预览地址
const previewUrl = `https://yourPreviewService/preview/view?url=${url}&fileName=${fileName}&displayName=${displayName}`;
window.open(previewUrl, "_blank");
```

#### 使用 data 传递 json 数据

预览参数需要做序列化后 base64 编码，所以会用到 `js-base64` 库。该集成方式对参数有一定隐藏作用。

```html
<!-- cdn 方式引入 -->
<script src="https://cdn.jsdelivr.net/npm/js-base64@3.7.8/base64.min.js"></script>
```

```js
// esm 方式引入
import { Base64 } from "js-base64";
```

```js
// 构造参数
const opts = {
  url: "https://mydomain.com/myfiles/sample.docx", // 网络文件地址，支持 http/https/ftp
  fileName: "sample.docx", // 真实文件名，用作文件类型辅助判断，如果文件地址中没有正确文件后缀，则需要手动传递
  displayName: "网络示例文档", // 用于标题栏等展示的文件名，非必需
};

// 对参数进行base64编码
const base64Data = encodeURIComponent(Base64.encode(JSON.stringify(opts)));

// 构造预览地址
const previewUrl = `https://yourPreviewService/preview/view?data=${base64Data}`;
window.open(previewUrl, "_blank");
```

## 预览本地文件

预览服务器上的本地文件方式与远程文件基本一致，只是文件路径的传参由 `url` 变为 `path`

```js
// 构造参数
const path = encodeURIComponent("/opt/myfiles/sample.docx"); // 本地文件地址
const fileName = encodeURIComponent("sample.docx"); // 真实文件名，用作文件类型判断，如果文件地址中没有正确文件后缀，则必须手动传递
const displayName = encodeURIComponent("本地示例文档"); // 用于标题栏等展示的文件名，非必需

// 构造预览地址
const previewUrl = `https://yourPreviewService/preview/view?path=${path}&fileName=${fileName}&displayName=${displayName}`;
window.open(previewUrl, "_blank");
```

## 子目录部署场景

系统支持子目录部署，具体见 [子目录部署](/docs/install/docker#子目录部署)

此时预览服务的路径则为 `https://yourPreviewService/<subpath>/preview/view?data=${base64Data}`


## 界面高级设置

#### 文字水印
文件预览支持文字水印，使用方法见 [文字水印](./interface-control.md#自定义水印)
#### 嵌入模式
文件预览支持嵌入模式，使用方法见 [嵌入模式](./interface-control.md#嵌入模式)