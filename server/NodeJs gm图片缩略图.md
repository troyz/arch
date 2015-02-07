> 我的另一篇文章: [Nginx/Apache图片缩略图技术](http://www.cnblogs.com/java-koma/p/4179858.html)
> 
> [gm官网](http://aheckmann.github.io/gm)
###1, 软件环境
> nodejs
> npm
> GraphicsMagick or ImageMagick

貌似ImageMagick在处理大图片时比GraphicsMagick要快很多。

###2, gm的一些关键函数
```
//1, 使用方式
var gm = require('gm');
gm("图片源路径")
    .resize(200,0)     //设置压缩后的w/h
    .setFormat('JPEG')
    .quality(70)       //设置压缩质量: 0-100
    .strip()
    .autoOrient()
    .write("压缩后保存路径" , 
    function(err){console.log("err: " + err);})
//2, 获取图片尺寸
gm("图片路径").size(function(err,value){});
//3, 获取图片大小
gm("图片路径").filesize(function(err,value){});```
```

> **resize函数**, [详细参数](http://www.graphicsmagick.org/GraphicsMagick.html#details-resize)
> resize {w}x{h} {%} {@} {!} {<} {>}
> 这里需要注意第3个参数
> 
> **%** 
> 表示按照width/height的百分比, resize(70, 0, '%')表示宽度为原先的70%
> 
> **@**
> (×_×)表示不明白，貌似可以限制压缩文件的filesize, gm文档上是这样描述的：
> Use @ to specify the maximum area in pixels of an image.
> 
> **! **
> 表示强制width/height, resize(70, 70, '%')表示输出图片尺寸70x70，图片可能变形
> 
> **^**
> 表示最小width/height, resize(70,70,'^')表示width/height最小不能小于70px
> 
> **\>**
> 表示只有源图片的width or height超过指定的width/height时，图片尺寸才会变。
> 如：源图片大小：640x640, resize(1000, 1000)最终图片尺寸不变。
> 
> **<**
> 跟**\>**正好相反，表示只有源图片的width or height小于指定的width/height时，图片尺寸才会变

gm默认使用GraphicsMagick处理图片，如果你想使用ImageMagick，则:

```
var gm = require('gm');
var imageMagick = gm.subClass({ imageMagick: true })
//使用方式同上，把上面的gm(..)函数替换成imageMagick(..)函数即可
```