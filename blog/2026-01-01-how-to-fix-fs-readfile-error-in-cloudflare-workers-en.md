---
slug: how-to-fix-fs-readfile-error-in-cloudflare-workers-en
title:  Fixing "[unenv] fs.readFile is not implemented yet!" When Deploying Next.js to Cloudflare Workers
authors: baiying
tags: [cloudflare, workers, next.js, fs, readfile, node.js]
---

If you deploy a Next.js app (especially using `@opennextjs/cloudflare`) to Cloudflare Workers and use the AWS SDK v3 (`@aws-sdk/client-s3`) for R2, you may run into this runtime error:

```
Error: [unenv] fs.readFile is not implemented yet!
```

This can occur even if you configured `webpack.resolve.fallback = { fs: false }` in `next.config.ts` or enabled `nodejs_compat` in `wrangler.toml`. The error typically appears while calling `getSignedUrl` or initializing an `S3Client`.

This post explains the root cause and presents a proven solution using the lightweight, Workers-friendly `aws4fetch` library.

## Background: Upgrading from Cloudflare Pages to Workers

As teams migrate from Cloudflare Pages (older `@cloudflare/next-on-pages` flows) to the more flexible Workers architecture (via `@opennextjs/cloudflare` / OpenNext), they gain better cold starts and finer control. However, the Edge Runtime constraints become stricter.

Some Node.js polyfills that previously hid `fs` usage no longer apply in the newer OpenNext build/runtime. As a result, code that worked on Pages can suddenly fail with `fs.readFile` errors after upgrading to Workers.

## Why this happens

Cloudflare Workers run on a V8-based Edge Runtime, not a full Node.js environment. While Workers provide a `nodejs_compat` mode, it does not fully emulate Node's core modules. The AWS SDK v3 is primarily designed for Node.js and, in certain code paths, will attempt to read local credentials or config files using `fs.readFile`.

OpenNext's build tooling and environment emulation (via `unenv`) may not implement `fs.readFile`, so when the SDK tries to call it at runtime, you get the error above.

## Approaches that usually don't fully fix it

- Changing Webpack fallbacks (`fs: false`) helps resolve build-time resolution but won't stop runtime calls inside the SDK.
- Adding `export const runtime = 'edge'` to API routes can conflict with OpenNext's bundling model (and OpenNext may require edge functions to be split differently).
- Polyfilling `fs` is brittle and often brings more compatibility issues in Workers.

## The robust solution: use aws4fetch

Abandoning `@aws-sdk` and using `aws4fetch`—a tiny, Fetch-native signer—solves the problem. `aws4fetch` is implemented using Fetch and Web Crypto APIs, making it compatible with Cloudflare Workers and other edge runtimes.

### Install / remove packages

Remove the incompatible AWS packages and add `aws4fetch`:

```bash
npm uninstall @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
npm install aws4fetch
```

### Example: Generate a presigned PUT URL for uploading

Old (problematic) code with `@aws-sdk`:

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

// ...initialize S3 client...
const command = new PutObjectCommand({ Bucket: '...', Key: '...', ContentType: '...' });
const url = await getSignedUrl(S3, command, { expiresIn: 600 });
```

New (Workers-compatible) code with `aws4fetch`:

```typescript
import { AwsClient } from 'aws4fetch';

const r2 = new AwsClient({
  accessKeyId: env.R2_ACCESS_KEY_ID,
  secretAccessKey: env.R2_SECRET_ACCESS_KEY,
  service: 's3',
  region: 'auto',
});

const url = new URL(`https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`);
url.searchParams.set('X-Amz-Expires', '600');

const signed = await r2.sign(new Request(url, {
  method: 'PUT',
  headers: { 'Content-Type': contentType },
}), { aws: { signQuery: true } });

return NextResponse.json({ url: signed.url });
```

### Example: Presigned GET URL for downloads (with filename)

```typescript
const url = new URL(`https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`);
url.searchParams.set('X-Amz-Expires', '3600');
url.searchParams.set('response-content-disposition', `attachment; filename="${encodeURIComponent(filename)}"`);

const signed = await r2.sign(new Request(url, { method: 'GET' }), { aws: { signQuery: true } });

return NextResponse.json({ url: signed.url });
```

### Example: Stream file from R2 inside Worker

If you need to fetch and stream the file directly from R2 within a Worker endpoint:

```typescript
const url = `https://${env.R2_BUCKET_NAME}.${env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com/${key}`;
const response = await r2.fetch(url);
if (!response.ok) return new NextResponse('File not found', { status: 404 });
return new NextResponse(response.body, { headers: { 'Content-Type': fileMime } });
```

## Notes about build/runtime and OpenNext

- OpenNext (and its Cloudflare adapter) bundles the Next.js app into a Worker. Some patterns like using `runtime = 'edge'` per-route can complicate bundling; prefer Fetch-native libraries where possible.
- Replacing `@aws-sdk` with `aws4fetch` reduces bundle size and avoids Node-specific code paths that trigger `fs` usage.

## Summary

- The `fs.readFile` error is caused by Node-oriented libraries attempting to access the filesystem in an Edge runtime.
- The practical and robust fix is to switch to a Fetch-native signer such as `aws4fetch` for generating presigned URLs and interacting with R2.
- This change resolves the `fs.readFile` runtime error and reduces your Worker bundle size and runtime complexity.

---

**References**

- The Docs Are Wrong! Here's How to Implement Presigned URLs using Cloudflare R2 and Workers — https://gebna.gg/blog/fix-fs-readFile-is-not-implemented-yet
- aws4fetch GitHub — https://github.com/mhart/aws4fetch
