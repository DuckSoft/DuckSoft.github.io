---
layout: post
title: 破解 91porn 的视频网址加密机制
author: DuckSoft
categories: [安全]
tags: [JavaScript, 混淆, 91porn]
image: cutting.jpg
---

## 0x0: 写在前面
自从之前写的一个油猴脚本 [`91porn-utility`](https://github.com/DuckSoft/91porn-utility) 所利用的白嫖 VIP 会员的方法被官方封禁之后，这个脚本的卖点也就从之前的“破解 VIP 高清视频”变成了“清爽去广告”。

随着脚本在 GitHub 上的点击量不断激增，截至发文之日，脚本的下载量已经突破 13k 人次，各式各样的人也在 Issue 列表里留下了各种言论。有的 Issue 发了一个 Bug Report，内容是“脚本太牛逼的bug”；有的 Issue 劝我不要公开这个脚本，以免利用方式被官方封禁（事实上他一口奶中了）。不过，其中一个人的 Issue 格外扎眼——他想付费让我做一个 91porn 的爬虫。

说实话，刚看到这个 Issue 的时候，我的眼里满是轻蔑——91porn 这种网站的爬虫难道还值得掏钱去请人做吗？尽管如此，我还是出于对技术的好奇，掏出 GoLand 对着窗口就是一顿乱敲，搭建了一个简单的并发爬虫框架。

Boilerplate 代码写完了，接下来应该是去网站上实地考察了。爬取视频页码、爬取视频页面到单个视频的链接列表、批量获取所有的视频链接，这些都轻车熟路地完成了。终于只需要解决一个问题，就能让这个爬虫变得有价值了——把单个视频网页的网址转换成实际播放的视频的网址。

当我再次抱着轻蔑的心态点开某个 NSFW 的视频页面的源代码，定位到一段特别扎眼的代码时，我愣住了：

```html
<script>
<!--
document.write(strencode("Ynl6Fy4tKR1pC3MtIQMpWBt+CntRZyN1IhBwIX8tCgcmAGVgJV0JQX8oAAsFFF0fEXMfFgEzBRJ9Pg90HytYHxISAhpVXwFYFWFfdjR2IGdmJGktYg9jczRqASIEJ2UhDwFyJQsfGSIYPhwFGDcMGn4sI04PFCBcKiscdCk+BUAGWxMWLhEBNQYKY1x8cldK","214aJucw3X1WBndaQLbK5/bCniHY0ymrkj0Qi7n83BksImdkr7NlMIHh3DBFPhmkqVS56lPaGP0Dy3vU+H+X/Z1Ff/mkfc3yVTCpZLCNHjY4VMMb3xv6BPG2ccNAJyPyLhIftVWCJ8R+","Ynl6Fy4tKR1pC3MtIQMpWBt+CntRZyN1IhBwIX8tCgcmAGVgJV0JQX8oAAsFFF0fEXMfFgEzBRJ9Pg90HytYHxISAhpVXwFYFWFfdjR2IGdmJGktYg9jczRqASIEJ2UhDwFyJQsfGSIYPhwFGDcMGn4sI04PFCBcKiscdCk+BUAGWxMWLhEBNQYKY1x8cldK"));
//-->
</script>
```

刷新一下页面，发现这段代码又变成了另一个：

```html
<script>
<!--
document.write(strencode("NC19FwABclwtGQ5VEEYOVBMFPFEVD3ZYdTh7TSIhBSEYXhZ+DlpRGCAEOEsOFGE9AS4eGQZMAAIrAmYBFjN8FlAAHngOBCkBdwpKRTFXEzsqNRtTHycccD06BEAHPwwmHTgYEDkLbSFoe3UIHhF5LTQGKT0iGWU3MwEAU1AQGUASejBJG3U4PRtEVScqJlBK","de3adY86wJL/s+CmY7TaqG7n9AC5mubTU4COB06alnS3BmXIbjOcJ6Mxex+3YpIb3DOWm7x88Ls/UdEaxXJ+PpVAnoI4HhBSSBpGlX7M8/09Pk8UyRpJls0YzIRf3WLyXIj9A2nKWvdP","NC19FwABclwtGQ5VEEYOVBMFPFEVD3ZYdTh7TSIhBSEYXhZ+DlpRGCAEOEsOFGE9AS4eGQZMAAIrAmYBFjN8FlAAHngOBCkBdwpKRTFXEzsqNRtTHycccD06BEAHPwwmHTgYEDkLbSFoe3UIHhF5LTQGKT0iGWU3MwEAU1AQGUASejBJG3U4PRtEVScqJlBK"));
//-->
</script>
```

按理来讲，一个视频的地址是固定的，这从网页加载完毕之后的开发者工具上能明显地看出来。但是这个给予网页视频实际地址的代码，每次刷新完毕之后都会发生变化，显然这背后隐藏着什么不为人知的秘密。

于是乎，我们的探索从这里开始。

## 0x1: 寻根溯源

从上面的那段神秘代码中，我们不难发现这个名为 `strencode` 的蜜汁函数才是我们要关心的重点。在 Firefox 的开发者工具中打开控制台，输入 `strencode`，得到下面的结果：

```
>>> strencode
+ strencode() [jump to definition]
|- arguments: null
|- caller: null
|- length: 2
|- name: "strencode"
+- prototype: Object { … }
+- <prototype>: function ()
```

点击跳转到定义后，我们来到了一个叫 `md5.js` 的文件里，但是文件乱的像屎一样：

```javascript
;var encode_version = 'sojson.v5', lbbpm = '__0x33ad7',  __0x33ad7=['QMOTw6XDtVE=','w5XDgsORw5LCuQ==','wojDrWTChFU=','dkdJACw=','w6zDpXDDvsKVwqA=','ZifCsh85fsKaXsOOWg==','RcOvw47DghzDuA==','w7siYTLCnw=='];(function(_0x94dee0,_0x4a3b74){var _0x588ae7=function(_0x32b32e){while(--_0x32b32e){_0x94dee0['push'](_0x94dee0['shift']());}};_0x588ae7(++_0x4a3b74);}(__0x33ad7,0x8f));var _0x5b60=function(_0x4d4456,_0x5a24e3){_0x4d4456=_0x4d4456-0x0;var _0xa82079=__0x33ad7[_0x4d4456];if(_0x5b60['initialized']===undefined){(function(){var _0xef6e0=typeof window!=='undefined'?window:typeof process==='object'&&typeof require==='function'&&typeof global==='object'?global:this;var _0x221728='ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=';_0xef6e0['atob']||(_0xef6e0['atob']=function(_0x4bb81e){var _0x1c1b59=String(_0x4bb81e)['replace'](/=+$/,'');for(var _0x5e3437=0x0,_0x2da204,_0x1f23f4,_0x3f19c1=0x0,_0x3fb8a7='';_0x1f23f4=_0x1c1b59['charAt'](_0x3f19c1++);~_0x1f23f4&&(_0x2da204=_0x5e3437%0x4?_0x2da204*0x40+_0x1f23f4:_0x1f23f4,_0x5e3437++%0x4)?_0x3fb8a7+=String['fromCharCode'](0xff&_0x2da204>>(-0x2*_0x5e3437&0x6)):0x0){_0x1f23f4=_0x221728['indexOf'](_0x1f23f4);}return _0x3fb8a7;});}());var _0x43712e=function(_0x2e9442,_0x305a3a){var _0x3702d8=[],_0x234ad1=0x0,_0xd45a92,_0x5a1bee='',_0x4a894e='';_0x2e9442=atob(_0x2e9442);for(var _0x67ab0e=0x0,_0x1753b1=_0x2e9442['length'];_0x67ab0e<_0x1753b1;_0x67ab0e++){_0x4a894e+='%'+('00'+_0x2e9442['charCodeAt'](_0x67ab0e)['toString'](0x10))['slice'](-0x2);}_0x2e9442=decodeURIComponent(_0x4a894e);for(var _0x246dd5=0x0;_0x246dd5<0x100;_0x246dd5++){_0x3702d8[_0x246dd5]=_0x246dd5;}for(_0x246dd5=0x0;_0x246dd5<0x100;_0x246dd5++){_0x234ad1=(_0x234ad1+_0x3702d8[_0x246dd5]+_0x305a3a['charCodeAt'](_0x246dd5%_0x305a3a['length']))%0x100;_0xd45a92=_0x3702d8[_0x246dd5];_0x3702d8[_0x246dd5]=_0x3702d8[_0x234ad1];_0x3702d8[_0x234ad1]=_0xd45a92;}_0x246dd5=0x0;_0x234ad1=0x0;for(var _0x39e824=0x0;_0x39e824<_0x2e9442['length'];_0x39e824++){_0x246dd5=(_0x246dd5+0x1)%0x100;_0x234ad1=(_0x234ad1+_0x3702d8[_0x246dd5])%0x100;_0xd45a92=_0x3702d8[_0x246dd5];_0x3702d8[_0x246dd5]=_0x3702d8[_0x234ad1];_0x3702d8[_0x234ad1]=_0xd45a92;_0x5a1bee+=String['fromCharCode'](_0x2e9442['charCodeAt'](_0x39e824)^_0x3702d8[(_0x3702d8[_0x246dd5]+_0x3702d8[_0x234ad1])%0x100]);}return _0x5a1bee;};_0x5b60['rc4']=_0x43712e;_0x5b60['data']={};_0x5b60['initialized']=!![];}var _0x4be5de=_0x5b60['data'][_0x4d4456];if(_0x4be5de===undefined){if(_0x5b60['once']===undefined){_0x5b60['once']=!![];}_0xa82079=_0x5b60['rc4'](_0xa82079,_0x5a24e3);_0x5b60['data'][_0x4d4456]=_0xa82079;}else{_0xa82079=_0x4be5de;}return _0xa82079;};if(typeof encode_version!=='undefined'&&encode_version==='sojson.v5'){function strencode(_0x50cb35,_0x1e821d){var _0x59f053={'MDWYS':'0|4|1|3|2','uyGXL':function _0x3726b1(_0x2b01e8,_0x53b357){return _0x2b01e8(_0x53b357);},'otDTt':function _0x4f6396(_0x33a2eb,_0x5aa7c9){return _0x33a2eb<_0x5aa7c9;},'tPPtN':function _0x3a63ea(_0x1546a9,_0x3fa992){return _0x1546a9%_0x3fa992;}};var _0xd6483c=_0x59f053[_0x5b60('0x0','cEiQ')][_0x5b60('0x1','&]Gi')]('|'),_0x1a3127=0x0;while(!![]){switch(_0xd6483c[_0x1a3127++]){case'0':_0x50cb35=_0x59f053[_0x5b60('0x2','ofbL')](atob,_0x50cb35);continue;case'1':code='';continue;case'2':return _0x59f053[_0x5b60('0x3','mLzQ')](atob,code);case'3':for(i=0x0;_0x59f053[_0x5b60('0x4','J2rX')](i,_0x50cb35[_0x5b60('0x5','Z(CX')]);i++){k=_0x59f053['tPPtN'](i,len);code+=String['fromCharCode'](_0x50cb35[_0x5b60('0x6','s4(u')](i)^_0x1e821d['charCodeAt'](k));}continue;case'4':len=_0x1e821d[_0x5b60('0x7','!Mys')];continue;}break;}}}else{alert('');};
```

使用 Firefox 自带的文件格式化功能，得到一段长达 129 行的代码。代码非常长，建议读者快速滚动跳过：

```javascript
;
var encode_version = 'sojson.v5',
lbbpm = '__0x33ad7',
__0x33ad7 = [
  'QMOTw6XDtVE=',
  'w5XDgsORw5LCuQ==',
  'wojDrWTChFU=',
  'dkdJACw=',
  'w6zDpXDDvsKVwqA=',
  'ZifCsh85fsKaXsOOWg==',
  'RcOvw47DghzDuA==',
  'w7siYTLCnw=='
];
(function (_0x94dee0, _0x4a3b74) {
  var _0x588ae7 = function (_0x32b32e) {
    while (--_0x32b32e) {
      _0x94dee0['push'](_0x94dee0['shift']());
    }
  };
  _0x588ae7(++_0x4a3b74);
}(__0x33ad7, 143));
var _0x5b60 = function (_0x4d4456, _0x5a24e3) {
  _0x4d4456 = _0x4d4456 - 0;
  var _0xa82079 = __0x33ad7[_0x4d4456];
  if (_0x5b60['initialized'] === undefined) {
    (function () {
      var _0xef6e0 = typeof window !== 'undefined' ? window : typeof process === 'object' && typeof require === 'function' && typeof global === 'object' ? global : this;
      var _0x221728 = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/=';
      _0xef6e0['atob'] || (_0xef6e0['atob'] = function (_0x4bb81e) {
        var _0x1c1b59 = String(_0x4bb81e) ['replace'](/=+$/, '');
        for (var _0x5e3437 = 0, _0x2da204, _0x1f23f4, _0x3f19c1 = 0, _0x3fb8a7 = ''; _0x1f23f4 = _0x1c1b59['charAt'](_0x3f19c1++); ~_0x1f23f4 && (_0x2da204 = _0x5e3437 % 4 ? _0x2da204 * 64 + _0x1f23f4 : _0x1f23f4, _0x5e3437++ % 4) ? _0x3fb8a7 += String['fromCharCode'](255 & _0x2da204 >> ( - 2 * _0x5e3437 & 6))  : 0) {
          _0x1f23f4 = _0x221728['indexOf'](_0x1f23f4);
        }
        return _0x3fb8a7;
      });
    }());
    var _0x43712e = function (_0x2e9442, _0x305a3a) {
      var _0x3702d8 = [
      ],
      _0x234ad1 = 0,
      _0xd45a92,
      _0x5a1bee = '',
      _0x4a894e = '';
      _0x2e9442 = atob(_0x2e9442);
      for (var _0x67ab0e = 0, _0x1753b1 = _0x2e9442['length']; _0x67ab0e < _0x1753b1; _0x67ab0e++) {
        _0x4a894e += '%' + ('00' + _0x2e9442['charCodeAt'](_0x67ab0e) ['toString'](16)) ['slice']( - 2);
      }
      _0x2e9442 = decodeURIComponent(_0x4a894e);
      for (var _0x246dd5 = 0; _0x246dd5 < 256; _0x246dd5++) {
        _0x3702d8[_0x246dd5] = _0x246dd5;
      }
      for (_0x246dd5 = 0; _0x246dd5 < 256; _0x246dd5++) {
        _0x234ad1 = (_0x234ad1 + _0x3702d8[_0x246dd5] + _0x305a3a['charCodeAt'](_0x246dd5 % _0x305a3a['length'])) % 256;
        _0xd45a92 = _0x3702d8[_0x246dd5];
        _0x3702d8[_0x246dd5] = _0x3702d8[_0x234ad1];
        _0x3702d8[_0x234ad1] = _0xd45a92;
      }
      _0x246dd5 = 0;
      _0x234ad1 = 0;
      for (var _0x39e824 = 0; _0x39e824 < _0x2e9442['length']; _0x39e824++) {
        _0x246dd5 = (_0x246dd5 + 1) % 256;
        _0x234ad1 = (_0x234ad1 + _0x3702d8[_0x246dd5]) % 256;
        _0xd45a92 = _0x3702d8[_0x246dd5];
        _0x3702d8[_0x246dd5] = _0x3702d8[_0x234ad1];
        _0x3702d8[_0x234ad1] = _0xd45a92;
        _0x5a1bee += String['fromCharCode'](_0x2e9442['charCodeAt'](_0x39e824) ^ _0x3702d8[(_0x3702d8[_0x246dd5] + _0x3702d8[_0x234ad1]) % 256]);
      }
      return _0x5a1bee;
    };
    _0x5b60['rc4'] = _0x43712e;
    _0x5b60['data'] = {
    };
    _0x5b60['initialized'] = !![];
  }
  var _0x4be5de = _0x5b60['data'][_0x4d4456];
  if (_0x4be5de === undefined) {
    if (_0x5b60['once'] === undefined) {
      _0x5b60['once'] = !![];
    }
    _0xa82079 = _0x5b60['rc4'](_0xa82079, _0x5a24e3);
    _0x5b60['data'][_0x4d4456] = _0xa82079;
  } else {
    _0xa82079 = _0x4be5de;
  }
  return _0xa82079;
};
if (typeof encode_version !== 'undefined' && encode_version === 'sojson.v5') {
  function strencode(_0x50cb35, _0x1e821d) {
    var _0x59f053 = {
      'MDWYS': '0|4|1|3|2',
      'uyGXL': function _0x3726b1(_0x2b01e8, _0x53b357) {
        return _0x2b01e8(_0x53b357);
      },
      'otDTt': function _0x4f6396(_0x33a2eb, _0x5aa7c9) {
        return _0x33a2eb < _0x5aa7c9;
      },
      'tPPtN': function _0x3a63ea(_0x1546a9, _0x3fa992) {
        return _0x1546a9 % _0x3fa992;
      }
    };
    var _0xd6483c = _0x59f053[_0x5b60('0x0', 'cEiQ')][_0x5b60('0x1', '&]Gi')]('|'),
    _0x1a3127 = 0;
    while (!![]) {
      switch (_0xd6483c[_0x1a3127++]) {
        case '0':
          _0x50cb35 = _0x59f053[_0x5b60('0x2', 'ofbL')](atob, _0x50cb35);
          continue;
        case '1':
          code = '';
          continue;
        case '2':
          return _0x59f053[_0x5b60('0x3', 'mLzQ')](atob, code);
        case '3':
          for (i = 0; _0x59f053[_0x5b60('0x4', 'J2rX')](i, _0x50cb35[_0x5b60('0x5', 'Z(CX')]); i++) {
            k = _0x59f053['tPPtN'](i, len);
            code += String['fromCharCode'](_0x50cb35[_0x5b60('0x6', 's4(u')](i) ^ _0x1e821d['charCodeAt'](k));
          }
          continue;
        case '4':
          len = _0x1e821d[_0x5b60('0x7', '!Mys')];
          continue;
      }
      break;
    }
  }
} else {
  alert('');
};
```

看来这段代码就是其罪魁祸首了。

## 0x2: 代码剖析
### 0x20: 直入主题：对 `strencode` 函数体的分析
从原代码的 88 行开始，我们可以看到这个函数的定义：
```javascript
function strencode(_0x50cb35, _0x1e821d)
```

显然这个函数只接收两个参数，而我们在前面看到的对这个函数的调用却传递了三个参数。根据 JavaScript 的特性，最后一个参数在这里是没有作用的。事实上，对函数调用的仔细观察可以发现，每次调用这个函数，所传递的第一个参数和第三个参数是相同的，而第三个参数恰好是没有任何作用的，这可能是代码的作者为了混淆视听故意而为。


进入函数体，在原代码的 89 到 100 行，我们首先看到了一个 JavaScript 对象的定义：
```javascript
var _0x59f053 = {
    'MDWYS': '0|4|1|3|2',
    'uyGXL': function _0x3726b1(_0x2b01e8, _0x53b357) {
        return _0x2b01e8(_0x53b357);
    },
    'otDTt': function _0x4f6396(_0x33a2eb, _0x5aa7c9) {
        return _0x33a2eb < _0x5aa7c9;
    },
    'tPPtN': function _0x3a63ea(_0x1546a9, _0x3fa992) {
        return _0x1546a9 % _0x3fa992;
    }
};
```

其中：
* `uyGXL` 函数：接受两个参数，返回的是将第一个参数作为函数，第二个参数作为单一参数的函数调用的返回值；
* `otDTt` 函数：接受两个参数，将第一个参数与第二个参数比较大小，然后返回结果；
* `tPPtN` 函数：接受两个参数，对第一个参数用第二个参数取模，然后返回结果。


接下来在 101 到 102 行是几个变量的声明：
```javascript
var _0xd6483c = _0x59f053[_0x5b60('0x0', 'cEiQ')][_0x5b60('0x1', '&]Gi')]('|'), _0x1a3127 = 0;
```

可以看到其中用到了原代码之前定义好的一个函数 `_0x5b60`，位于原代码的 22 到 86 行，占据了很大的一段空间。但幸运的是我们似乎不必对这个函数进行深入剖析，在这里我们可以把他当作一个**黑箱**去对待。打开控制台，我们直接输入两处函数调用，就可以得到下面的结果：

```
>>> _0x5b60('0x0', 'cEiQ')
"MDWYS"
>>> _0x5b60('0x1', '&]Gi')
"split"
```

将从控制台所得的结果替换掉原调用表达式，上面的代码就变成：
```javascript
var _0xd6483c = _0x59f053["MDWYS"]["split"]('|'), _0x1a3127 = 0;
```

而变量 `_0x59f053` 正是我们刚才分析过的那个 JavaScript 对象，直接查阅可知 `_0x59f053["MDWYS"] -> '0|4|1|3|2'`。这样，变量 `_0xd6483c` 的定义就变成了：
```javascript
var _0xd6483c = '0|4|1|3|2'["split"]('|');
```

在这里，如果对 JavaScript 的 *property access literals* 之间的转换比较熟悉的话，容易知道上面的写法等价于下面的形式，同时也不难得到变量 `_0xd6483c` 的实际值为数组 `[0,4,1,3,2]`。
```javascript
var _0xd6483c = '0|4|1|3|2'."split"('|');
```

回到原代码，接下来在 103 到 124 行是一个巨大的 `while` 循环，其模式酷似一个状态机（实际就是），代码如下：
```javascript
while (!![]) {
    switch (_0xd6483c[_0x1a3127++]) {
        case '0':
            _0x50cb35 = _0x59f053[_0x5b60('0x2', 'ofbL')](atob, _0x50cb35);
            continue;
        case '1':
            code = '';
            continue;
        case '2':
            return _0x59f053[_0x5b60('0x3', 'mLzQ')](atob, code);
        case '3':
            for (i = 0; _0x59f053[_0x5b60('0x4', 'J2rX')](i, _0x50cb35[_0x5b60('0x5', 'Z(CX')]); i++) {
                k = _0x59f053['tPPtN'](i, len);
                code += String['fromCharCode'](_0x50cb35[_0x5b60('0x6', 's4(u')](i) ^ _0x1e821d['charCodeAt'](k));
            }
            continue;
        case '4':
            len = _0x1e821d[_0x5b60('0x7', '!Mys')];
            continue;
    }
    break;
}
```

对于 `while` 中的循环条件 `!![]`，我们即使猜也要猜它是 `true`，因为如果这个条件是 `false`，那么这个循环根本不会执行，也就没有之后的什么事情了。出于本能，我把这个表达式丢进了 console，返回的结果果然是 `true`，那么就没有什么问题了。

在这个死循环的内部，是一个简单的 `switch` 语句块，其分歧条件是：
```javascript
_0xd6483c[_0x1a3127++]
```

根据之前所得，我们知道：
* 其中的变量 `_0xd6483c` 就是数组 `[0,4,1,3,2]`；
* 其中的变量 `_0x1a3127 = 0`。

通过分析不难得出，这个状态机的状态执行-转移顺序和数组中的顺序一样，是 `0 -> 4 -> 1 -> 3 -> 2` 这样的顺序。我们将这些代码从 `switch` 的 `case` 中抽离出来，然后按照实际执行顺序将它们重新拼凑在一起，就可以得到进一步简化的代码：
```javascript
// status 0
_0x50cb35 = _0x59f053[_0x5b60('0x2', 'ofbL')](atob, _0x50cb35);
// status 4
len = _0x1e821d[_0x5b60('0x7', '!Mys')];
// status 1
code = '';
// status 3
for (i = 0; _0x59f053[_0x5b60('0x4', 'J2rX')](i, _0x50cb35[_0x5b60('0x5', 'Z(CX')]); i++) {
    k = _0x59f053['tPPtN'](i, len);
    code += String['fromCharCode'](_0x50cb35[_0x5b60('0x6', 's4(u')](i) ^ _0x1e821d['charCodeAt'](k));
}
// status 2
return _0x59f053[_0x5b60('0x3', 'mLzQ')](atob, code);
```

这里出现了一个陌生的函数 `_0x5b60`，它在原代码的 22 到 86 行被定义。同样的篇幅巨大，同样的可以当作黑盒子去用。我们使用控制台对这些函数调用一一求值，可以得到：
```
>>> _0x5b60('0x2', 'ofbL')
"uyGXL"
>>> _0x5b60('0x7', '!Mys')
"length"
>>> _0x5b60('0x4', 'J2rX')
"otDTt"
>>> _0x5b60('0x5', 'Z(CX')
"length"
>>> _0x5b60('0x6', 's4(u')
"charCodeAt"
>>> _0x5b60('0x3', 'mLzQ')
"uyGXL"
```

将这些结果一一替换掉函数调用，我们得到：
```javascript
_0x50cb35 = _0x59f053["uyGXL"](atob, _0x50cb35);
len = _0x1e821d["length"];
code = '';
for (i = 0; _0x59f053["otDTt"](i, _0x50cb35["length"]); i++) {
    k = _0x59f053['tPPtN'](i, len);
    code += String['fromCharCode'](_0x50cb35["length"](i) ^ _0x1e821d['charCodeAt'](k));
}
return _0x59f053["uyGXL"](atob, code);
```

再根据我们已知的 `_0x59f053` 变量，对相应位置进行进一步替换，我们得到：
```javascript
_0x50cb35 = atob(_0x50cb35);
len = _0x1e821d["length"];
code = '';
for (i = 0; i < _0x50cb35["length"]; i++) {
    k = i % len;
    code += String['fromCharCode'](_0x50cb35["length"](i) ^ _0x1e821d['charCodeAt'](k));
}
return atob(code);
```

结合这个函数的函数体，我们得到了这个函数比较接近实际的代码：
```javascript
function strencode(_0x50cb35, _0x1e821d) {
    _0x50cb35 = atob(_0x50cb35);
    len = _0x1e821d["length"];
    code = '';
    for (i = 0; i < _0x50cb35["length"]; i++) {
        k = i % len;
        code += String['fromCharCode'](_0x50cb35["length"](i) ^ _0x1e821d['charCodeAt'](k));
    }
    return atob(code);
}
```

对相应变量进行语义化、重命名，我们得到：
```javascript
function strencode(input, key) {
    input = atob(input);
    len = key["length"];
    code = '';
    for (i = 0; i < input["length"]; i++) {
        k = i % len;
        code += String['fromCharCode'](input["length"](i) ^ key['charCodeAt'](k));
    }
    return atob(code);
}
```

转换 `property access literal`，最终我们得到下面的代码：
```javascript
function strencode(input, key) {
    input = atob(input);
    len = key.length;
    code = '';
    for (i = 0; i < input.length; i++) {
        k = i % len;
        code += String.fromCharCode(input.length(i) ^ key.charCodeAt(k));
    }
    return atob(code);
}
```

这就是我们实际解出来的 `strencode` 函数的原貌。
## 0x3: 总结与收获

## 0x4: 番外
### 0x40: `!![]` 与 JSFuck 加密
不过话说回来，如果我们认真审视这个 `!![]` 表达式，不难联想到有一种 JavaScript 脚本的混淆叫做 [JSFuck](http://www.jsfuck.com/)。这种混淆方式使用 JavaScript 的一些特性，对代码进行等价替换，例如：
* `false     => ![]`
* `true      => !![]`
* `undefined => [][[]]`
* `NaN       => +[![]]`
* `0         => +[]`
* ...

