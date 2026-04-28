---
description: >-
  BaseMetas Fileview 完整部署对接指南，面向开发人员，涵盖 Docker 部署、Nginx 反向代理、API 集成、iframe
  嵌入、安全配置、高级功能等全流程技术指导。
title: BaseMetas Fileview 部署对接指南：从零到生产环境的完整集成手册
datetime: '2026-04-12 22:44:08'
permalink: /posts/fileview-deployment-integration-guide
outline: deep
category: 文章
tags:
  - fileview
  - 部署
  - 集成
  - 开发指南
prev:
  text: Nextcloud 文件预览困局与破局：集成 BaseMetas Fileview 实现全格式在线预览
  link: /posts/nextcloud-integrate-fileview
next:
  text: Kodbox 文件预览进化之路：集成 BaseMetas Fileview 打造私有化全格式预览
  link: /posts/kodbox-integrate-fileview
---

# BaseMetas Fileview 部署对接指南

> 本文面向需要将文件预览能力集成到自有系统的开发人员，覆盖从 Docker 部署、反向代理配置、API 对接到生产环境最佳实践的完整流程。读完本文，你将能够在自己的业务系统中完成 Fileview 的部署与集成。

---

## 一、了解 Fileview

### 1.1 它是什么

BaseMetas Fileview 是一款通用型在线文档预览引擎，支持超过 200 种文件格式的在线预览，覆盖 Office 文档、PDF、OFD、CAD 图纸、3D 模型、代码文件、流程图、思维导图、压缩包、音视频、图片等全格式类型。

它的设计目标是：**作为独立的预览服务，通过标准 HTTP 接口集成到任意业务系统中**。你的系统只需要把"文件地址"告诉 Fileview，它负责下载、转换、渲染，最终在浏览器中呈现预览结果。

### 1.2 架构概览

Fileview 内部由两个服务组成，Docker 镜像已将它们打包在一起，对外只暴露一个 HTTP 端口：

```
┌─────────────────────────────────────────┐
│           Docker 容器 (端口 80)           │
│                                         │
│  ┌─────────────┐    ┌─────────────────┐ │
│  │  预览服务     │───▶│   转换服务       │ │
│  │ (HTTP API)  │    │ (格式转换引擎)    │ │
│  └──────┬──────┘    └────────┬────────┘ │
│         │                    │          │
│     ┌───┴────────────────────┴───┐      │
│     │   RocketMQ + Redis + 存储   │      │
│     └────────────────────────────┘      │
└─────────────────────────────────────────┘
```

- **预览服务**：对外提供 `/preview/api/**` 等 HTTP 接口，负责请求编排、文件下载、权限校验、结果缓存、长轮询响应
- **转换服务**：内部服务，通过订阅 MQ 事件被动触发，负责文件格式转换（Office → PDF/HTML、CAD → SVG 等）
- **Redis**：统一的状态与缓存中心，存储下载/转换任务状态、缓存预览结果
- **RocketMQ**：事件总线，承载预览相关事件，负责下载任务和转换任务的异步投递

作为集成方，你不需要关心内部实现，只需要：**部署 → 配置代理 → 调用 API**。

### 1.3 集成交互流程

```
你的业务系统                        Fileview 预览服务
    │                                    │
    │  1. 用户点击文件                      │
    │──────────────────────────────────▶ │
    │  2. 构造预览URL                      │
    │     (包含文件下载地址)                  │
    │  3. 浏览器打开预览URL                  │
    │  ─────────────────────────────────▶│
    │                                    │ 4. 下载文件
    │                                    │ 5. 格式转换
    │                                    │ 6. 返回渲染结果
    │  ◀─────────────────────────────────│
    │  7. 用户看到预览效果                    │
```

---

## 二、环境准备

### 2.1 硬件要求

| 规格 | 最低配置 | 推荐配置 | 高并发场景             |
| ---- | -------- | -------- | ---------------------- |
| CPU  | 2 核     | 4 核     | 8 核+                  |
| 内存 | 2GB      | 4GB      | 8GB+                   |
| 磁盘 | 10GB     | 50GB     | 100GB+（取决于文件量） |

> 磁盘空间主要用于存储下载的源文件和转换后的结果文件。Fileview 内置临时文件清理机制，会定期清理过期文件。

### 2.2 软件要求

- **Docker**：20.10+（必需）
- **Nginx**：生产环境推荐使用反向代理（非必需，但强烈建议）

### 2.3 支持的 CPU 架构

- AMD64 (x86_64)
- ARM64 (aarch64)

---

## 三、部署

### 3.1 拉取镜像

```bash
# Docker Hub（首选）
docker pull basemetas/fileview:latest

# 国内镜像加速
docker pull docker.1ms.run/basemetas/fileview:latest
# 或
docker pull dockerproxy.net/basemetas/fileview:latest
```

### 3.2 启动服务

**最简启动（开发/测试）：**

```bash
docker run -itd \
    --name fileview \
    -p 9000:80 \
    --restart=always \
    basemetas/fileview:latest
```

容器内部监听 `80` 端口，映射到宿主机 `9000` 端口。

**挂载数据目录（生产推荐）：**

```bash
docker run -itd \
    --name fileview \
    -p 9000:80 \
    -v /data/fileview/storage:/opt/fileview/data \
    -v /data/fileview/logs:/opt/fileview/logs \
    --restart=always \
    basemetas/fileview:latest
```

这样可以将文件存储和日志持久化到宿主机，避免容器重建后数据丢失。

### 3.3 验证部署

浏览器访问 `http://<你的服务器IP>:9000/`，如果看到 Fileview 欢迎页，说明部署成功。

也可以通过命令行验证：

```bash
curl -I http://localhost:9000/preview/welcome
# 应该返回 HTTP 200
```

---

## 四、反向代理配置（生产环境必做）

生产环境中不建议直接暴露 Docker 端口，应通过 Nginx 反向代理统一管理。以下提供两种典型部署模式。

### 4.1 独立域名部署

Fileview 使用一个独立的域名或子域名：

```nginx
server {
    listen 443 ssl;
    server_name preview.example.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:9000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Host $host;
    }
}
```

此时预览服务的 Base URL 为：`https://preview.example.com`

### 4.2 子路径部署（与业务系统同域）

Fileview 部署在业务系统的一个子路径下（如 `/fileview/`），这种方式可以避免跨域问题：

```nginx
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # 你的业务系统
    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Fileview 预览服务
    location /fileview/ {
        proxy_pass http://127.0.0.1:9000/;

        # 关键：告知 Fileview 自己处于子路径下
        proxy_set_header X-Forwarded-Prefix /fileview;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Host $host;
    }
}
```

> `X-Forwarded-Prefix` 是子路径部署的关键请求头。Fileview 会根据它自动调整生成的预览 URL 前缀，确保返回给浏览器的地址是正确可访问的。

此时预览服务的 Base URL 为：`https://app.example.com/fileview`

### 4.3 代理头说明

| 请求头               | 作用                       | 是否必需         |
| -------------------- | -------------------------- | ---------------- |
| `X-Forwarded-Proto`  | 告知原始协议（http/https） | 是（HTTPS 场景） |
| `X-Forwarded-Host`   | 告知原始访问域名           | 推荐             |
| `X-Forwarded-Prefix` | 告知子路径前缀             | 子路径部署时必需 |
| `X-Forwarded-Port`   | 告知原始访问端口           | 推荐             |
| `REMOTE-HOST`        | 告知原始客户端 IP          | 推荐             |
| `Host`               | 标准 Host 头               | 是               |

Fileview 会按优先级解析这些请求头，动态生成预览 URL 的 Base URL。详细机制参见[预览地址生成机制](https://fileview.basemetas.cn/docs/product/preview-url)。

---

## 五、API 集成

Fileview 的集成核心只有一件事：**构造正确的预览 URL**。

### 5.1 预览 URL 格式

```
{baseUrl}/preview/view?url={fileUrl}&fileName={fileName}
```

其中：
- `{baseUrl}`：Fileview 服务的访问地址
- `{fileUrl}`：文件的网络下载地址（需 URL 编码）
- `{fileName}`：文件名（需 URL 编码，用于文件类型判断）

### 5.2 预览网络文件（最常用）

**方式一：query 参数（简单直接）**

```javascript
// 你的系统中，生成文件的下载链接
const fileDownloadUrl = "https://app.example.com/api/files/download?id=12345";
const fileName = "年度报告.docx";

// 构造预览 URL
const previewUrl = `https://app.example.com/fileview/preview/view?url=${encodeURIComponent(fileDownloadUrl)}&fileName=${encodeURIComponent(fileName)}`;

// 打开预览
window.open(previewUrl, "_blank");
```

**参数说明：**

| 参数          | 类型   | 必填     | 说明                                                                                  |
| ------------- | ------ | -------- | ------------------------------------------------------------------------------------- |
| `url`         | string | 是       | 文件的网络下载地址，支持 http/https/ftp                                               |
| `fileName`    | string | 条件必填 | 真实文件名（含后缀），用于类型判断。如果 `url` 中已包含正确后缀（如 `.docx`），可不传 |
| `displayName` | string | 否       | 在标题栏显示的文件名，不影响类型判断                                                  |
| `watermark`   | string | 否       | 文字水印内容，支持 `\n` 换行，建议不超过两行                                          |
| `mode`        | string | 否       | 显示模式：`normal`（默认，带菜单栏）或 `embed`（嵌入模式，无菜单栏）                  |

**方式二：data 参数（Base64 编码，隐藏参数）**

对于不希望在 URL 中暴露文件地址的场景，可以使用 `data` 参数将所有参数 Base64 编码后传递：

```javascript
// 安装 js-base64：npm install js-base64
import { Base64 } from "js-base64";

const opts = {
    url: "https://app.example.com/api/files/download?id=12345",
    fileName: "年度报告.docx",
    displayName: "2025年度报告"
};

// Base64 编码
const base64Data = encodeURIComponent(Base64.encode(JSON.stringify(opts)));

// 构造预览 URL
const previewUrl = `https://app.example.com/fileview/preview/view?data=${base64Data}`;
window.open(previewUrl, "_blank");
```

CDN 引入方式：

```html
<script src="https://cdn.jsdelivr.net/npm/js-base64@3.7.8/base64.min.js"></script>
<script>
    const opts = {
        url: "https://app.example.com/api/files/download?id=12345",
        fileName: "年度报告.docx"
    };
    const base64Data = encodeURIComponent(Base64.encode(JSON.stringify(opts)));
    const previewUrl = `https://app.example.com/fileview/preview/view?data=${base64Data}`;
    window.open(previewUrl, "_blank");
</script>
```

### 5.3 预览本地文件

如果文件已存在于 Fileview 容器可访问的磁盘路径上（比如通过 Docker 卷挂载共享目录），可以使用 `path` 参数替代 `url`：

```javascript
const filePath = "/opt/fileview/data/shared/report.docx";
const fileName = "report.docx";

const previewUrl = `https://app.example.com/fileview/preview/view?path=${encodeURIComponent(filePath)}&fileName=${encodeURIComponent(fileName)}`;
window.open(previewUrl, "_blank");
```

> 注意：`path` 是 Fileview 容器内部的文件路径，不是宿主机路径。如果使用 Docker 挂载 `-v /host/files:/opt/fileview/data/shared`，则对应容器内路径为 `/opt/fileview/data/shared/xxx`。

### 5.4 两种传参方式对比

| 特性       | query 参数方式        | data 参数方式                  |
| ---------- | --------------------- | ------------------------------ |
| 实现复杂度 | 低                    | 中（需 Base64 库）             |
| URL 可读性 | 文件地址在 URL 中可见 | 参数被 Base64 编码，不直接可见 |
| 参数安全性 | 一般                  | 有一定隐藏作用                 |
| 适用场景   | 内部系统、开发调试    | 对外系统、安全要求较高场景     |
| 后端集成   | 简单字符串拼接        | 需 JSON 序列化 + Base64 编码   |

---

## 六、前端集成模式

### 6.1 新窗口打开（最简单）

```javascript
function previewFile(fileUrl, fileName) {
    const url = encodeURIComponent(fileUrl);
    const name = encodeURIComponent(fileName);
    const previewUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${url}&fileName=${name}`;
    window.open(previewUrl, "_blank");
}
```

适合：快速集成、对 UI 无特殊要求的场景。

### 6.2 iframe 嵌入（推荐）

```html
<iframe
    id="file-preview"
    src=""
    width="100%"
    height="600px"
    frameborder="0"
    allowfullscreen
></iframe>

<script>
function previewInIframe(fileUrl, fileName) {
    const url = encodeURIComponent(fileUrl);
    const name = encodeURIComponent(fileName);
    // 使用 embed 模式去除 Fileview 自带的菜单栏
    const previewUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${url}&fileName=${name}&mode=embed`;
    document.getElementById('file-preview').src = previewUrl;
}
</script>
```

`mode=embed` 参数会隐藏 Fileview 的顶部菜单栏，使预览内容更好地嵌入你的页面。

适合：文件详情页、审批流程页面、文档管理系统等需要"内嵌预览"的场景。

### 6.3 弹窗/抽屉预览

```javascript
// 以弹窗模式预览（示例使用原生 JS）
function previewInModal(fileUrl, fileName) {
    const url = encodeURIComponent(fileUrl);
    const name = encodeURIComponent(fileName);
    const previewUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${url}&fileName=${name}&mode=embed`;

    // 创建遮罩层
    const overlay = document.createElement('div');
    overlay.style.cssText = 'position:fixed;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,0.7);z-index:9999;display:flex;align-items:center;justify-content:center;';
    
    // 创建 iframe 容器
    const container = document.createElement('div');
    container.style.cssText = 'width:90%;height:90%;background:#fff;border-radius:8px;overflow:hidden;position:relative;';
    
    // 关闭按钮
    const closeBtn = document.createElement('button');
    closeBtn.innerText = '关闭';
    closeBtn.style.cssText = 'position:absolute;top:8px;right:12px;z-index:10;padding:4px 12px;cursor:pointer;';
    closeBtn.onclick = () => document.body.removeChild(overlay);
    
    // iframe
    const iframe = document.createElement('iframe');
    iframe.src = previewUrl;
    iframe.style.cssText = 'width:100%;height:100%;border:none;';
    
    container.appendChild(closeBtn);
    container.appendChild(iframe);
    overlay.appendChild(container);
    overlay.onclick = (e) => { if (e.target === overlay) document.body.removeChild(overlay); };
    document.body.appendChild(overlay);
}
```

适合：列表页"快速预览"、不离开当前页面查看文件的场景。

---

## 七、后端集成示例

### 7.1 Java (Spring Boot)

```java
import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;

@Service
public class FilePreviewService {

    @Value("${fileview.base-url}")
    private String fileviewBaseUrl;  // 如 https://app.example.com/fileview

    /**
     * 生成文件预览 URL
     * @param fileDownloadUrl 文件下载地址
     * @param fileName 文件名
     * @return 预览 URL
     */
    public String buildPreviewUrl(String fileDownloadUrl, String fileName) {
        String encodedUrl = URLEncoder.encode(fileDownloadUrl, StandardCharsets.UTF_8);
        String encodedName = URLEncoder.encode(fileName, StandardCharsets.UTF_8);
        return String.format("%s/preview/view?url=%s&fileName=%s",
                fileviewBaseUrl, encodedUrl, encodedName);
    }

    /**
     * 生成带水印的预览 URL
     */
    public String buildPreviewUrlWithWatermark(String fileDownloadUrl, String fileName, String watermark) {
        String encodedUrl = URLEncoder.encode(fileDownloadUrl, StandardCharsets.UTF_8);
        String encodedName = URLEncoder.encode(fileName, StandardCharsets.UTF_8);
        String encodedWatermark = URLEncoder.encode(watermark, StandardCharsets.UTF_8);
        return String.format("%s/preview/view?url=%s&fileName=%s&watermark=%s",
                fileviewBaseUrl, encodedUrl, encodedName, encodedWatermark);
    }

    /**
     * 生成嵌入模式的预览 URL（无菜单栏）
     */
    public String buildEmbedPreviewUrl(String fileDownloadUrl, String fileName) {
        return buildPreviewUrl(fileDownloadUrl, fileName) + "&mode=embed";
    }
}
```

在 Controller 中使用：

```java
@RestController
@RequestMapping("/api/files")
public class FileController {

    @Autowired
    private FilePreviewService previewService;

    @GetMapping("/{fileId}/preview-url")
    public Map<String, String> getPreviewUrl(@PathVariable String fileId) {
        // 从你的业务中获取文件信息
        FileInfo file = fileService.getById(fileId);
        String downloadUrl = fileService.generateDownloadUrl(fileId);

        String previewUrl = previewService.buildPreviewUrl(downloadUrl, file.getName());
        return Map.of("previewUrl", previewUrl);
    }
}
```

### 7.2 Python (Flask/Django)

```python
from urllib.parse import quote

FILEVIEW_BASE_URL = "https://app.example.com/fileview"

def build_preview_url(file_download_url: str, file_name: str, 
                      watermark: str = None, embed: bool = False) -> str:
    """构造 Fileview 预览 URL"""
    encoded_url = quote(file_download_url, safe='')
    encoded_name = quote(file_name, safe='')
    
    preview_url = f"{FILEVIEW_BASE_URL}/preview/view?url={encoded_url}&fileName={encoded_name}"
    
    if watermark:
        preview_url += f"&watermark={quote(watermark, safe='')}"
    
    if embed:
        preview_url += "&mode=embed"
    
    return preview_url
```

### 7.3 Go

```go
package preview

import (
    "fmt"
    "net/url"
)

const FileviewBaseURL = "https://app.example.com/fileview"

func BuildPreviewURL(fileDownloadURL, fileName string) string {
    return fmt.Sprintf("%s/preview/view?url=%s&fileName=%s",
        FileviewBaseURL,
        url.QueryEscape(fileDownloadURL),
        url.QueryEscape(fileName),
    )
}

func BuildEmbedPreviewURL(fileDownloadURL, fileName string) string {
    return BuildPreviewURL(fileDownloadURL, fileName) + "&mode=embed"
}
```

### 7.4 PHP

```php
<?php
define('FILEVIEW_BASE_URL', 'https://app.example.com/fileview');

function buildPreviewUrl(string $fileDownloadUrl, string $fileName, 
                         ?string $watermark = null, bool $embed = false): string {
    $params = [
        'url' => $fileDownloadUrl,
        'fileName' => $fileName,
    ];
    
    if ($watermark !== null) {
        $params['watermark'] = $watermark;
    }
    
    if ($embed) {
        $params['mode'] = 'embed';
    }
    
    return FILEVIEW_BASE_URL . '/preview/view?' . http_build_query($params);
}

// 使用
$previewUrl = buildPreviewUrl(
    'https://app.example.com/download/report.docx',
    'report.docx',
    "内部文件\n仅供预览",
    true
);
```

---

## 八、关键功能：文字水印

Fileview 支持在预览时添加文字水印，适用于所有支持水印的文件格式，可用于防止截屏泄露。

```javascript
// 水印内容支持 \n 换行，建议不超过两行
const watermark = encodeURIComponent("内部文件 仅供预览\n2026-04-12");

const previewUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${fileUrl}&fileName=${fileName}&watermark=${watermark}`;
```

水印是在预览时动态叠加的，不会修改原始文件。

---

## 九、关键功能：嵌入模式

嵌入模式（`mode=embed`）会隐藏 Fileview 自带的顶部菜单栏（包含文件名、工具按钮等），使预览内容区域全屏展示。

```javascript
// 普通模式（默认）- 带菜单栏
const normalUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${fileUrl}&fileName=${fileName}`;

// 嵌入模式 - 无菜单栏
const embedUrl = `${FILEVIEW_BASE_URL}/preview/view?url=${fileUrl}&fileName=${fileName}&mode=embed`;
```

适用于将预览嵌入到你自己的 UI 框架中，由你的系统提供统一的顶栏和操作按钮。

---

## 十、安全配置

### 10.1 可信站点白名单（生产环境必须配置）

Fileview 在预览网络文件时，会先从指定 URL 下载文件。为防止 SSRF 攻击和非授权访问，应配置可信站点白名单，确保 Fileview 只从你的系统域名下载文件。

白名单配置写在 Fileview 的 `application.yml` 中。对于 Docker 部署，通过环境变量传入：

```bash
docker run -itd \
    --name fileview \
    -p 9000:80 \
    -e FILEVIEW_NETWORK_SECURITY_TRUSTED_SITES="app.example.com, *.internal.example.com" \
    --restart=always \
    basemetas/fileview:latest
```

或在配置文件中：

```yaml
fileview:
  network:
    security:
      # 仅允许从这些域名下载文件
      trusted-sites: app.example.com, *.internal.example.com
      # 明确禁止的域名（优先级高于白名单）
      untrusted-sites: ""
```

**规则说明：**

- 配置 `example.com` 会自动匹配其所有子域名
- 支持通配符：`*.example.com`
- 黑名单优先级高于白名单
- 大小写不敏感
- 多个规则用逗号分隔

**推荐策略：**

| 环境      | 配置建议                           |
| --------- | ---------------------------------- |
| 开发/测试 | 可不配置白名单（允许所有域名）     |
| 生产环境  | 必须配置白名单，仅允许你的系统域名 |
| 内网隔离  | 配置内网域名/IP                    |

### 10.2 文件下载地址的安全性

你的系统生成的文件下载 URL 应具备基本的安全防护：

- **临时链接**：设置过期时间（如 30 分钟）
- **签名校验**：URL 包含签名参数，防止伪造
- **权限校验**：确保只有授权用户才能生成下载链接

```javascript
// 推荐：生成带签名和过期时间的临时下载链接
const downloadUrl = generateSignedUrl(fileId, {
    expiresIn: 1800,  // 30 分钟
    signature: computeHmac(fileId + timestamp, secretKey)
});
```

### 10.3 加密文件处理

Fileview 支持预览带密码的 Office 文档（docx/xlsx/pptx）和加密压缩包（zip/rar/7z）。

处理流程：
1. Fileview 检测到文件加密，返回 `PASSWORD_REQUIRED` 状态
2. 前端弹出密码输入框
3. 用户输入密码后，Fileview 验证并解密预览
4. 密码加密存储在 Redis 中，30 分钟内同一客户端再次访问无需重复输入

---

## 十一、文件 URL 的对接要求

这是集成中最关键的一点：**你的系统需要提供一个可供 Fileview 服务端下载文件的 URL**。

### 11.1 基本要求

1. **可达性**：Fileview 容器能够通过网络访问该 URL
2. **直接下载**：URL 访问后直接返回文件二进制流（而非 HTML 页面）
3. **Content-Type**：建议返回正确的 MIME 类型（非必需，Fileview 主要靠 `fileName` 参数判断类型）

### 11.2 典型场景

**场景 A：文件服务有公开下载接口**

```
https://app.example.com/api/files/download?id=12345&token=xxx
```

直接将此 URL 作为 `url` 参数传给 Fileview 即可。

**场景 B：文件存储在 OSS/S3**

```
https://your-bucket.oss-cn-hangzhou.aliyuncs.com/files/report.docx?OSSAccessKeyId=xxx&Signature=xxx&Expires=xxx
```

使用预签名 URL，注意设置合理的过期时间。

**场景 C：文件接口需要 Cookie/Token 认证**

Fileview 的文件下载是服务端行为（不是浏览器行为），所以不会携带用户的 Cookie。

**解决方案：**

- 方案一：生成不需要认证的临时下载链接（推荐）
- 方案二：将 Token 作为 URL 参数传递（如 `?token=xxx`）
- 方案三：使用 Fileview 的本地文件预览，让你的系统先把文件写入共享目录

**场景 D：Fileview 和业务系统在同一内网**

可以使用内网地址：

```
http://192.168.1.100:8080/api/files/download?id=12345
```

但需注意 Docker 网络。如果 Fileview 在 Docker 中，需确保容器能访问宿主机或其他容器的网络：

```bash
# 方案一：使用 host.docker.internal 访问宿主机（Docker Desktop 支持）
http://host.docker.internal:8080/api/files/download?id=12345

# 方案二：使用 Docker 网络
docker network create my-net
docker run --network my-net --name fileview ...
docker run --network my-net --name my-app ...
# 此时可用容器名访问：http://my-app:8080/api/files/download?id=12345
```

---

## 十二、Docker Compose 部署（推荐）

对于与业务系统联合部署的场景，推荐使用 Docker Compose：

```yaml
version: '3.8'

services:
  fileview:
    image: basemetas/fileview:latest
    container_name: fileview
    ports:
      - "9000:80"
    volumes:
      - fileview-data:/opt/fileview/data
      - fileview-logs:/opt/fileview/logs
    restart: always
    networks:
      - app-network

  # 你的业务系统（示例）
  my-app:
    image: your-app:latest
    container_name: my-app
    ports:
      - "8080:8080"
    restart: always
    networks:
      - app-network

  # Nginx 反向代理
  nginx:
    image: nginx:alpine
    container_name: nginx
    ports:
      - "443:443"
      - "80:80"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d
      - ./nginx/ssl:/etc/nginx/ssl
    depends_on:
      - fileview
      - my-app
    restart: always
    networks:
      - app-network

volumes:
  fileview-data:
  fileview-logs:

networks:
  app-network:
    driver: bridge
```

在此配置下，Nginx 的代理配置可以使用容器名（`fileview`、`my-app`）作为上游地址：

```nginx
# nginx/conf.d/default.conf
server {
    listen 443 ssl;
    server_name app.example.com;

    ssl_certificate     /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # 业务系统
    location / {
        proxy_pass http://my-app:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Fileview
    location /fileview/ {
        proxy_pass http://fileview:80/;
        proxy_set_header X-Forwarded-Prefix /fileview;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-Host $host;
    }
}
```

---

## 十三、集成自检清单

部署和集成完成后，按以下清单逐项验证：

### 部署验证

- [ ] 访问 Fileview 欢迎页正常显示
- [ ] 如使用反向代理，通过代理地址访问欢迎页正常
- [ ] 如使用子路径部署，确认 `{baseUrl}/preview/welcome` 可访问

### 预览功能验证

- [ ] 使用 `url` 参数预览一个公开可下载的 PDF 文件
- [ ] 使用 `url` 参数预览一个 DOCX 文件（需经过格式转换）
- [ ] 使用 `url` 参数预览一个图片文件
- [ ] 如有本地文件场景，使用 `path` 参数预览本地文件
- [ ] 验证 `mode=embed` 嵌入模式生效（无菜单栏）
- [ ] 验证 `watermark` 水印参数生效

### 安全验证

- [ ] 配置可信站点白名单后，尝试预览非白名单域名的文件（应被拒绝）
- [ ] 确认文件下载 URL 有适当的鉴权和过期机制

### 网络验证

- [ ] Fileview 容器能够访问你的文件下载地址
- [ ] 如在 Docker 内，确认容器间网络互通
- [ ] HTTPS 场景下，确认预览 URL 生成为 HTTPS

---

## 十四、常见问题排查

### 问题 1：预览页面白屏或长时间加载

**排查步骤：**

1. 查看浏览器 Network 面板，确认预览 URL 请求是否成功（HTTP 200）
2. 检查 Fileview 容器日志：`docker logs fileview`
3. 确认 Fileview 能否下载文件：进入容器测试 `curl <文件下载URL>`
4. 对于 Office 文件，首次预览需要格式转换，可能需要几秒到十几秒

### 问题 2：预览 URL 中的地址不正确

**原因：** 反向代理未正确传递 `X-Forwarded-*` 请求头。

**解决：** 检查 Nginx 配置中是否包含完整的代理头设置，特别是 `X-Forwarded-Proto`、`X-Forwarded-Host`、`X-Forwarded-Prefix`。

### 问题 3：文件下载失败

**排查步骤：**

1. 确认文件下载 URL 在 Fileview 容器内可访问
2. 检查白名单配置是否包含文件所在域名
3. 检查文件下载 URL 是否已过期
4. 如使用 Docker，检查容器网络配置（DNS 解析、网络连通性）

### 问题 4：中文文件名乱码

**解决：** 确保所有参数都经过 `encodeURIComponent` / `URLEncoder.encode` 编码。

### 问题 5：OFD 文件中文显示为方框

**原因：** 缺少中文字体（思源字体）。

**解决：** 参考[常见问题文档](https://fileview.basemetas.cn/docs/product/faq)中的字体安装说明。

---

## 十五、生产环境最佳实践

### 15.1 必做事项

1. **配置反向代理**：不直接暴露 Docker 端口
2. **启用 HTTPS**：在 Nginx 层终止 SSL
3. **配置可信站点白名单**：限制文件下载来源
4. **挂载数据卷**：持久化文件存储和日志
5. **设置容器自动重启**：`--restart=always`

### 15.2 建议事项

1. **使用子路径部署**：避免跨域问题
2. **生成带签名和过期时间的文件下载 URL**
3. **监控容器资源**：关注 CPU、内存、磁盘使用率
4. **定期查看日志**：及时发现转换失败等异常

### 15.3 性能优化

对于高并发场景，可参考[性能调优指南](https://fileview.basemetas.cn/docs/product/performance)进行以下优化：

- JVM 堆内存调整（通过 Docker 环境变量）
- 转换线程池并发度控制
- Redis 连接池调优
- 长轮询策略调整

---

## 十六、支持的文件格式速查

| 类别            | 格式                                                                                         |
| --------------- | -------------------------------------------------------------------------------------------- |
| Office          | doc, docx, wps, rtf, txt, md 等文档类；xls, xlsx, csv, ods 等表格类；ppt, pptx, odp 等演示类 |
| 版式文档        | pdf, ofd                                                                                     |
| 图片            | jpg, png, gif, bmp, svg, webp, psd, tif, tga, emf, wmf                                       |
| CAD             | dwg, dxf                                                                                     |
| 3D 模型         | gltf, glb, obj, stl, fbx, ply, dae, wrl, 3ds, 3mf, 3dm                                       |
| 流程图/思维导图 | vsd, vsdm, vsdx, vssm, vssx, vstm, vstx, bpmn, drawio, xmind                                 |
| 压缩包          | zip, jar, rar, 7z, tar, tar.gz                                                               |
| 代码            | java, kotlin, scala, python, go, rust, c, cpp, js, ts, vue, php, ruby, shell 等 100+ 语言    |
| 音视频          | mp4, mp3, webm, wav, aac, ogg, flac, avi, mkv, mov, flv                                      |
| 电子书          | epub                                                                                         |
| 文本            | html, xml, json, yaml, toml, ini, conf                                                       |

完整列表参见 [支持格式](https://fileview.basemetas.cn/docs/product/formats)。

---

## 相关资料

- 产品介绍：https://fileview.basemetas.cn/docs/product/summary
- 安装部署：https://fileview.basemetas.cn/docs/install/docker
- 服务集成：https://fileview.basemetas.cn/docs/feature/integration
- 架构介绍：https://fileview.basemetas.cn/docs/product/architecture
- 安全设置：https://fileview.basemetas.cn/docs/product/security
- 性能调优：https://fileview.basemetas.cn/docs/product/performance
- 常见问题：https://fileview.basemetas.cn/docs/product/faq
- 在线体验：https://file.basemetas.cn
- GitHub：https://github.com/basemetas/fileview
