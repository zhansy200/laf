---
title: 云函数常见问题
---

# {{ $frontmatter.title }}

这里是云函数开发过程中可能会遇到的一些问题。欢迎 Pr～

[[toc]]

## 云函数延迟执行

```typescript
import cloud from '@lafjs/cloud'

export async function main(ctx: FunctionContext) {
  const startTime = Date.now()
  console.log(startTime)
  await sleep(5000) // 延迟 5 秒
  console.log(Date.now() - startTime)
}

async function sleep(ms) {
  return new Promise(resolve =>
    setTimeout(resolve, ms)
  );
}
```

## 云函数设置某请求超时时间

```typescript
import cloud from '@lafjs/cloud'

export async function main(ctx: FunctionContext) {
  // 如果 getData 的异步操作在 4 秒内完成并返回，则 responseText 为 getDat 的返回值
  // 如果 4 秒内未完成，则 responseText 为''，不影响 getData 的实际运行
  const responseText = await Promise.race([
    getData(),
    sleep(4000).then(() => ''),
  ]);
  console.log(responseText, Date.now())
}

async function sleep(ms) {
  return new Promise(resolve =>
    setTimeout(resolve, ms)
  );
}

async function getData(){
  // 某个异步操作，以下通过 sleep 模拟超过 4 秒的情况
  await sleep(5000)
  const text = "getData 的返回值"
  console.log(text, Date.now())
  return text
}
```

## 云函数对接公众号简单示例

以下代码只兼容明文模式

```typescript
import * as crypto from 'crypto'
import cloud from '@lafjs/cloud'

export async function main(ctx: FunctionContext) {
  const { signature, timestamp, nonce, echostr } = ctx.query;
  const token = '123456'; // 这里的 token 自定义，需要对应微信后台的配置的 token

  // 验证消息是否合法，若不合法则返回错误信息
  if (!verifySignature(signature, timestamp, nonce, token)) {
    return 'Invalid signature';
  }

  // 如果是首次验证，则返回 echostr 给微信服务器
  if (echostr) {
    return echostr;
  }

  // 处理接收到的消息
  const payload = ctx.body.xml;
  // 如果接收的是文本
  if (payload.msgtype[0] === 'text') {
    // 公众号发什么回复什么
    return toXML(payload, payload.content[0]);
  }
}

// 校验微信服务器发送的消息是否合法
function verifySignature(signature, timestamp, nonce, token) {
  const arr = [token, timestamp, nonce].sort();
  const str = arr.join('');
  const sha1 = crypto.createHash('sha1');
  sha1.update(str);
  return sha1.digest('hex') === signature;
}

// 返回组装 xml
function toXML(payload, content) {
  const timestamp = Date.now();
  const { tousername: fromUserName, fromusername: toUserName } = payload;
  return `
  <xml>
    <ToUserName><![CDATA[${toUserName}]]></ToUserName>
    <FromUserName><![CDATA[${fromUserName}]]></FromUserName>
    <CreateTime>${timestamp}</CreateTime>
    <MsgType><![CDATA[text]]></MsgType>
    <Content><![CDATA[${content}]]></Content>
  </xml>
  `
}
```

## 云函数合成图片

需要先安装依赖 `canvas`

```typescript
import { createCanvas } from 'canvas'

export async function main(ctx: FunctionContext) {
  const canvas = createCanvas(200, 200)
  const context = canvas.getContext('2d')

  // Write "hello!"
  context.font = '30px Impact'
  context.rotate(0.1)
  context.fillText('hello!', 50, 100)

  // Draw line under text
  var text = context.measureText('hello!')
  context.strokeStyle = 'rgba(0,0,0,0.5)'
  context.beginPath()
  context.lineTo(50, 102)
  context.lineTo(30 + text.width, 102)
  context.stroke()

  // Write "Laf!"
  context.font = '30px Impact'
  context.rotate(0.1)
  context.fillText('Laf!', 50, 150)
  console.log(canvas.toDataURL())
  return `<img src= ${canvas.toDataURL()} />`
}
```

## 云函数防抖

通过 Laf 云函数的全局缓存可以很方便的设置防抖

以下是一个简单的防抖例子，前端请求时，需要在 header 中带上用户 token。

```typescript
// 云函数生成 Token
const accessToken_payload = {
  // 除了示例的，还可以加别的参数
  uid: login_user[0]._id, //一般是 user 表的_id
  role: login_user[0].role, //如果没有 role，可以不要
  exp: (Math.floor(Date.now() / 1000) + 60 * 60 * 24 * 7) * 1000, //7天过期
}
const token = cloud.getToken(accessToken_payload)
console.log(token)
```

```typescript
import cloud from '@lafjs/cloud'

export async function main(ctx: FunctionContext) {
  const FunctionName = ctx.request.params.name
  const sharedName = FunctionName + ctx.user.uid
  let lastCallTime = cloud.shared.get(sharedName)
  console.log(lastCallTime)
  if (lastCallTime > Date.now()) {
    console.log("请求太快了")
    return '请求太快了'
  }
  cloud.shared.set(sharedName, Date.now() + 1000)
  // 原有逻辑

  // 逻辑完毕后删除全局缓存
  cloud.shared.delete(sharedName)
}
```

## 云函数域名验证

部分微信服务需要验证 MP 开头的 txt 文件的值，以判断域名是否有权限

可以新建一个该文件名的云函数，如：`MP_123456789.txt`

直接返回该文本的内容

```typescript
import cloud from '@lafjs/cloud'

export async function main(ctx: FunctionContext) {
  // 这里直接返回文本内容
  return 'abcd...'
}
```

## Laf 应用 IP 池

下满例子为使用的是 laf.run 的情况。使用 laf.dev 或其他，下面命令需要更换域名。

- Windows 可在 CMD 中执行 `nslookup laf.run`

- Mac 可在终端中执行 `nslookup laf.run`

可看到全部 IP 池
