---
description: BaseMetas Fileview 自定义水印、嵌入模式等界面设置，支持 word/excel/ppt/pdf 等格式文档。
---
# 界面高级设置


BaseMetas Fileview 支持传递参数控制水印及模式等等，传参方式遵照标准集成参数，使用 query 方式传参。

支持的参数如下：

| 配置项   | 参数名    | 参数类型                                                 |
| -------- | --------- | -------------------------------------------------------- |
| 文字水印 | watermark | string, 支持使用 `\n` 换行。支持 word/excel/ppt/pdf 格式 |
| 显示模式 | mode      | string, 普通: `normal`，嵌入：`embed`。支持全部格式      |


## 自定义水印

展示自定义水印，支持 word/excel/ppt/pdf 等格式文档的文字水印。


```js
// 构造参数
const url = encodeURIComponent("https://mydomain.com/myfiles/sample.docx"); // 网络文件地址，支持 http/https/ftp
const fileName = encodeURIComponent("sample.docx"); // 真实文件名，用作文件类型判断，如果文件地址中没有正确文件后缀，则必须手动传递
const watermark = encodeURIComponent("BaseMetas Fileview\nwatermark"); // 文字水印，不要超过两行

// 构造预览地址
const previewUrl = `https://yourPreviewService/preview/view?url=${url}&fileName=${fileName}&watermark=${watermark}`;
window.open(previewUrl, "_blank");
```

## 嵌入模式

支持嵌入模式，即没有顶部菜单的模式，便于嵌入其他系统做区域展示。

```js
// 构造参数
const url = encodeURIComponent("https://mydomain.com/myfiles/sample.docx"); // 网络文件地址，支持 http/https/ftp
const fileName = encodeURIComponent("sample.docx"); // 真实文件名，用作文件类型判断，如果文件地址中没有正确文件后缀，则必须手动传递
const mode = "embed"; // normal: 普通，embed：嵌入，无菜单栏

// 构造预览地址
const previewUrl = `https://yourPreviewService/preview/view?url=${url}&fileName=${fileName}&mode=${mode}`;
window.open(previewUrl, "_blank");
```