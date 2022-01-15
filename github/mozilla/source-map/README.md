# mozilla/source-map

> [Source Map V3 规范][规范]

[规范]: https://docs.google.com/document/d/1U1RGAehQwRypUTovF1KRlpiOFze0b-_2gc6fAH0KY0k/edit#heading=h.1ce2c87bpj24


## base64 编码表
![base64 编码表](assets/base64.png)

## base64-vlq 编码原理
![base64 vlq 编码12345过程](assets/base64-vlq.png)

## 关于 Source map 规范
通过阅读源码，更加深入了解了Source Map的规范，Source Map分为2种，即源码中的`BasicSourceMapConsumer`和`IndexedSourceMapConsumer`，后者支持合并多个Source Map，通过`sections`字段将多份Souce Map连接起来。

> 为了避免遭受 [XSSI](https://security.googleblog.com/2011/05/website-security-for-webmasters.html) 攻击，Source Map文件的第一行可以是以`)]}'`开头的任意内容，在解析Source Map时需要忽略这样的行。
> 
> 目前各家浏览器对此的实现并不完全一致，主要是最后面的那个单引号`'`，Google Chrome不会要求必须有这个单引号，而 Firefox 则要求必须有，否则就是无效的SourceMap。
> 另外，规范规定 SourceMap 里面`version`字段是必须的，FireFox 也是严格按照这个规范做的，如果没有`version`字段，则不认为是一个合法的 SourceMap，而 Chrome 就宽松的多，可以没有`version`字段。
