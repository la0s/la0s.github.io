---
layout: post
title: 秒破iOS APP加密数据
key: 20150103
tags: Android Reverse
excerpt_separator: <!--more-->
---
毕业第一步，先把项目需求实现了。其实这篇文章和之前Android上frida破解加密数据思路是一致的，因为iOS作为一种闭源系统，没有Android那么多的packers和so库，iOS官方封装了自己统一的Crypto库：  
[CommonCryptor.h](https://opensource.apple.com/source/CommonCrypto/CommonCrypto-36064/CommonCrypto/CommonCryptor.h)/[CommonDigest.h](https://opensource.apple.com/source/CommonCrypto/CommonCrypto-36064/CommonCrypto/CommonDigest.h.auto.html)/[CommonHMAC.h](https://opensource.apple.com/source/CommonCrypto/CommonCrypto-36064/CommonCrypto/CommonHMAC.h.auto.html)，所以我们hook起来也很方便。<!--more-->拿我们之前搞过的一个投资APP(v4.2.9)为例，点击登录
![](https://raw.githubusercontent.com/la0s/la0s.github.io/master/screenshots/20181207.1.png)

request包
![](https://raw.githubusercontent.com/la0s/la0s.github.io/master/screenshots/20181207.2.png)

response包
![](https://raw.githubusercontent.com/la0s/la0s.github.io/master/screenshots/20181207.3.png)

直接hook CCCrypt函数，将这个脚本保存为CC_hook.js
```javascript
// Intercept the CCCrypt call.
Interceptor.attach(Module.findExportByName('libcommonCrypto.dylib', 'CCCrypt'), {
    onEnter: function (args) {
        // Save the arguments
        this.operation   = args[0]
        this.CCAlgorithm = args[1]
        this.CCOptions   = args[2]
        this.keyBytes    = args[3]
        this.keyLength   = args[4]
        this.ivBuffer    = args[5]
        this.inBuffer    = args[6]
        this.inLength    = args[7]
        this.outBuffer   = args[8]
        this.outLength   = args[9]
        this.outCountPtr = args[10]

        console.log('CCCrypt(' + 
            'operation: '   + this.operation    +', ' +
            'CCAlgorithm: ' + this.CCAlgorithm  +', ' +
            'CCOptions: '   + this.CCOptions    +', ' +
            'keyBytes: '    + this.keyBytes     +', ' +
            'keyLength: '   + this.keyLength    +', ' +
            'ivBuffer: '    + this.ivBuffer     +', ' +
            'inBuffer: '    + this.inBuffer     +', ' +
            'inLength: '    + this.inLength     +', ' +
            'outBuffer: '   + this.outBuffer    +', ' +
            'outLength: '   + this.outLength    +', ' +
            'outCountPtr: ' + this.outCountPtr  +')')

        if (this.operation == 0) {
            // Show the buffers here if this an encryption operation
            console.log("In buffer:")
            console.log(hexdump(ptr(this.inBuffer), {
                length: this.inLength.toInt32(),
                header: true,
                ansi: true
            }))
            console.log("Key: ")
            console.log(hexdump(ptr(this.keyBytes), {
                length: this.keyLength.toInt32(),
                header: true,
                ansi: true
            }))
            console.log("IV: ")
            console.log(hexdump(ptr(this.ivBuffer), {
                length: this.keyLength.toInt32(),
                header: true,
                ansi: true
            }))
        }
    },
    onLeave: function (retVal) {
        if (this.operation == 1) {
            // Show the buffers here if this a decryption operation
            console.log("Out buffer:")
            console.log(hexdump(ptr(this.outBuffer), {
                length: Memory.readUInt(this.outCountPtr),
                header: true,
                ansi: true
            }))
            console.log("Key: ")
            console.log(hexdump(ptr(this.keyBytes), {
                length: this.keyLength.toInt32(),
                header: true,
                ansi: true
            }))
            console.log("IV: ")
            console.log(hexdump(ptr(this.ivBuffer), {
                length: this.keyLength.toInt32(),
                header: true,
                ansi: true
            }))
        }
    }
})
```
直接使用 frida -U -f com.Dongfangjinke.dongfanghui -l CC_hook.js --no-pause
![](https://raw.githubusercontent.com/la0s/la0s.github.io/master/screenshots/20181207.4.png)

operation: 0x0代表加密，0x1代表解密，CCAlgorithm: 0x0指加密方式是kCCAlgorithmAES128，CCOptions: 0x1指模式是cbc，具体参阅CommonCryptor.h各参数意义，key=DATA_KEY20150116和iv=20150116和Android客户端是一致的，而且由于Android是在so库生成的AES加密，避免了Android端java层hook不到的情况。

参考  
[appmon project提供的scripts](https://github.com/dpnishant/appmon/tree/master/scripts/iOS/Crypto)   
[hack.lu的scripts](https://github.com/theart42/hack.lu/blob/master/IOS/Notes/05-Crypt/00-crypto-hooks.md)  
[Frida-javascript-api-apiresolver](https://www.frida.re/docs/javascript-api/#apiresolver)
