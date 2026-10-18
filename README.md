# CoolapkReviver

让旧版酷安通过服务端版本校验，恢复评论区显示

## 解决什么问题

酷安服务端对 v6 接口做了版本白名单。客户端版本过低时，请求统一返回 403，App 弹出
「评论接口异常，请尝试升级到新版酷安」，评论区、私信直接不可用。

官方解法是升级 App。本模块的解法是让旧版客户端在请求时自称新版，通过服务端的版本校验，
从而继续使用原有功能，不需要升级、不需要换包。

模块只改 HTTP 请求头里的版本标识，不动 APK、不改界面、不删广告、不加权限，
对 App 的其他行为没有影响。

## 原理

旧版酷安的所有版本标识由 `com.coolapk.market.AppConfig` 生成，再由
`HttpClientFactory$CoolMarketHeaderInterceptor` 逐对写入 OkHttp 请求：

```java
List<String> headers = appConfig.createHeaders();
for (int i = 0; i < headers.size(); i += 2) {
    builder.header(headers.get(i), headers.get(i + 1));
}
```

服务端靠这些头判断客户端版本。模块 hook `okhttp3.Request$Builder#header` 与 `#addHeader`，
在写入前替换版本相关的头，让服务端看到一个符合白名单的版本号。

因为 hook 的是 OkHttp 而不是酷安的混淆类，客户端小版本升级一般不需要改模块。

## 已知限制

- 只处理 OkHttp 出站请求头。评论、私信、点赞等 v6 接口都走这条链路；若有走 WebSocket 的通道则不在覆盖范围内。
- 若酷安日后混淆 `okhttp3`，`Class.forName` 会失败并在日志中报错，此时需改为 hook `AppConfig` 的混淆方法。
- 改写的是请求头里的版本号，客户端代码仍是旧版。若服务端将来在 v6 接口上新增字段，旧版解析可能不完整。

## 免责声明

仅供学习与研究使用。使用本模块产生的任何后果由使用者自行承担。
