[TOC]

# Fresco

[官网](https://www.fresco-cn.org/)

Fresco 的最大亮点在于它的内存管理。Fresco 会将图片放到一个特别的内存区域，当图片不再显示的时候，占用的内存会自动被释放，这会使得 APP 更流畅，减少因图片内存占用而引发的 OOM。当 APP 包含的图片较多时，这个效果尤其明显。

支持加载Gif图和WebP动图.

Fresco 中设计有一个叫做 **Image Pipeline** 的模块。它负责从网络，从本地文件系统，本地资源加载图片。为了最大限度节省空间和CPU时间，它含有3级缓存设计（2级内存，1级磁盘）。**Image Pipeline**允许你用很多种方式来自定义图片加载过程

Fresco 中设计有一个叫做 **Drawees** 模块，它会在图片加载完成前显示占位图，加载成功后自动替换为目标图片。当图片不再显示在屏幕上时，它会及时地释放内存和空间占用。

Fresco 支持图像的渐进式呈现，渐进式的图片格式先呈现大致的图片轮廓，然后随着图片下载的继续，逐渐呈现清晰的图片，这在低网速情况下浏览图片十分有帮助，可以带来更好地用户体验。另外，Fresco 支持加载 gif 图，支持 WebP 格式。



# Glide

他支持gif / webP / 缩略图 / Video .

Glide对每一个Context都保持一个RequestManager , 所以他可以与Activity/Fragment生命周期保持一致 , 并且他支持trimMemory.

Glide默认为UrlConnection , 可以配合okhttp和volley使用.

Glide在存储缓存图片至本地的时候时按照View的大小来存储的 , 这样在加载的时候可以节省内存.