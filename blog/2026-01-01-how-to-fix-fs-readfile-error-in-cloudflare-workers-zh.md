---
slug: how-to-fix-fs-readfile-error-in-cloudflare-workers-zh
title:  Next.js 部署 Cloudflare Workers 报错 "[unenv] fs.readFile is not implemented yet!" 的终极解决方案
authors: baiying
tags: [cloudflare, workers, next.js, fs, readfile, node.js]
---

在将 Next.js 项目（特别是使用 `@opennextjs/cloudflare`）部署到 Cloudflare Workers 时，如果你使用了 AWS SDK v3 (`@aws-sdk/client-s3`) 来操作 R2 存储桶，你很可能会在运行时遇到下面这个顽固的错误：

```
Error: [unenv] fs.readFile is not implemented yet!
```

即使你在 `next.config.ts` 中配置了 `webpack.resolve.fallback = { fs: false }`，或者在 `wrangler.toml` 中开启了 `nodejs_compat`，这个错误依然可能在调用 `getSignedUrl` 或初始化 `S3Client` 时出现。

本文将分析该错误产生的原因，并提供一个经过验证的、基于 `aws4fetch` 的轻量级替代方案。

## 背景：从 Cloudflare Pages 到 Workers 的架构升级

随着 Cloudflare 对 Next.js 支持的演进，越来越多的项目开始从传统的 Cloudflare Pages (基于 `@cloudflare/next-on-pages`) 迁移到更灵活的 Cloudflare Workers 架构 (基于 `@opennextjs/cloudflare`)。

这次升级带来了更快的冷启动速度和更细粒度的控制，但也意味着我们需要更严格地遵守 Edge Runtime 的限制。在旧的 Pages 架构中，某些 Node.js polyfills 可能掩盖了 `fs` 模块的调用，但在新的 OpenNext 架构下，构建工具对环境的模拟更加精简，导致隐藏的 `fs.readFile` 调用暴露为运行时错误。

这就是为什么在升级过程中，原本工作正常的 AWS SDK 代码突然开始报错的原因。

## 问题根源

Cloudflare Workers 是一个基于 V8 的 Edge Runtime 环境，它不是标准的 Node.js 环境。虽然 Cloudflare 提供了 `nodejs_compat` 标志来模拟部分 Node.js API，但文件系统（File System, `fs`）模块在无服务器边缘环境中是无法被完整实现的。

**为什么 AWS SDK 会报错？**
官方的 AWS SDK v3 设计之初主要面向 Node.js 环境。虽然它声称支持浏览器端，但在 Workers 这种混合环境中，SDK 内部的某些逻辑（例如加载凭证文件、配置文件读取）会尝试调用 `fs.readFile`。

当使用 OpenNext 构建 Next.js 应用时，构建工具会尝试通过 `unenv` 来模拟 Node 环境，但 `fs.readFile` 的模拟实现通常只是抛出一个 "not implemented" 错误。

## 尝试过但失败的方法

在找到最终解法之前，你可能尝试过以下方案（通常无效）：

1.  **修改 Webpack 配置**：在 `next.config.ts` 中设置 `fs: false`。这只能解决构建时的模块解析错误，无法解决运行时 SDK 内部调用的问题。
2.  **强制 Edge Runtime**：在 API 路由中添加 `export const runtime = 'edge'`。这在 OpenNext 架构下可能会导致构建失败，提示 Edge Runtime 函数必须定义在独立文件中。
3.  **使用 Polyfills**：尝试引入各种 `fs` 的 polyfill，通常会因为环境差异导致更多兼容性问题。

## 终极解决方案：使用 aws4fetch

最彻底的解决办法是**放弃臃肿的 `@aws-sdk`，改用专为 Fetch API 环境（如 Cloudflare Workers）设计的轻量级库 —— `aws4fetch`**。

`aws4fetch` 只有几 KB 大小，完全基于标准的 Web Crypto API 和 Fetch API 实现，完美兼容 Cloudflare Workers。

### 第一步：安装依赖

首先，卸载导致问题的 AWS SDK 包，并安装 `aws4fetch`：

```bash
npm uninstall @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install aws4fetch
```

### 第二步：重构代码

下面是几个常见场景的代码重构对比。

#### 场景 1：生成上传文件的预签名 URL (Presigned PUT URL)

**❌ 旧代码 (使用 @aws-sdk):**

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// ... 初始化 S3Client ...
const command = new PutObjectCommand({ Bucket: '...', Key: '...', ContentType: '...' });
const url = await getSignedUrl(S3, command, { expiresIn: 600 });
```

**✅ 新代码 (使用 aws4fetch):**

```typescript
import { AwsClient } from 'aws4fetch';

// 初始化客户端
const r2 = new AwsClient({
  accessKeyId: env.R2_ACCESS_KEY_ID,
  secretAccessKey: env.R2_SECRET_ACCESS_KEY,
  service: 's3',
  region: 'auto',
});

// 手动构建 URL
const url = new URL(`https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`);
url.searchParams.set('X-Amz-Expires', '600'); // 设置过期时间

// 生成签名
const signed = await r2.sign(new Request(url, {
  method: 'PUT',
  headers: {
    'Content-Type': contentType,
  },
}), {
  aws: { signQuery: true }, // 将签名参数附加到 URL 查询字符串中
});

return NextResponse.json({ url: signed.url });
```

#### 场景 2：生成下载文件的预签名 URL (Presigned GET URL)

如果你需要支持文件下载并指定文件名（Content-Disposition），处理方式类似：

**✅ 新代码:**

```typescript
const url = new URL(`https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`);
url.searchParams.set('X-Amz-Expires', '3600');

// 添加下载时的文件名头
url.searchParams.set('response-content-disposition', `attachment; filename="${encodeURIComponent(filename)}"`);

const signed = await r2.sign(new Request(url, {
  method: 'GET',
}), {
  aws: { signQuery: true },
});

return NextResponse.json({ url: signed.url });
```

#### 场景 3：在 Worker 中直接读取文件 (Stream)

如果你需要在 API 路由中直接读取 R2 文件并流式返回给前端：

**✅ 新代码:**

```typescript
const url = `https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`;

// 直接使用 r2.fetch 发起请求
const response = await r2.fetch(url);

if (!response.ok) {
  return new NextResponse('File not found', { status: 404 });
}

// 直接透传 Body 流
return new NextResponse(response.body, {
  headers: {
    'Content-Type': 'application/octet-stream',
    'Cache-Control': 'public, max-age=31536000',
  },
});
```

## 总结

在 Cloudflare Workers 或其他 Edge 环境中，尽量避免使用依赖 Node.js 核心模块（如 `fs`, `crypto`, `net`）的库。对于 AWS S3/R2 操作，`aws4fetch` 是一个更现代、更轻量且兼容性更好的选择。

通过替换 SDK，我们不仅彻底解决了 `fs.readFile` 报错，还显著减小了 Worker 的打包体积，提升了冷启动速度。

---

**参考资料：**
*   [The Docs Are Wrong! Here's How to Implement Presigned URLs using Cloudflare R2 and Workers](https://gebna.gg/blog/fix-fs-readFile-is-not-implemented-yet)
*   [aws4fetch GitHub Repository](https://github.com/mhart/aws4fetch)


