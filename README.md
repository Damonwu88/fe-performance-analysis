# 加载分析与性能优化
## 前言
积分商城模块相对比较独立，而且与运营活动、会员日密切相关。在推活动的期间，流量非常庞大，甚至活动推出的前后几分钟访问量以数十万计。所以这次选择积分商城模块进行优化升级

## 影响性能因素
（1）请求队列、网络质量、web应用程序、页面加载、资源下载
（2）白屏时间、首屏时间、页面加载完成时间、资源下载时间、整页完成时间
其中，加载过程会产生以下耗时，如：网络耗时、web服务器接口处理耗时、web服务器返回数据耗时、下载js、css、image、video等资源耗时。

## DOM文档加载过程
   解析html结构->加载外部脚本和样式表文件->解析并执行脚本代码->构造HTML DOM模型(DOMContentLoaded事件)->加载图片等外部文件->页面加载完毕(onload事件)

## 数据收集与分析
使用Chrome 浏览器的network 与 performance来收集数据
### （1）network 模块是用来收集网页中资源请求的数据，为了更好模拟移动端，将网速调整为slow 3G
 ![](https://github.com/Damonwu88/fe-performance-analysis/blob/master/1.png)
 ![](https://github.com/Damonwu88/fe-performance-analysis/blob/master/2.png)
说明：Wating(TTFB) 可以理解是指从客户端开始和服务端交互到服务端开始向客户端浏览器传输数据的时间（包括DNS、socket连接和请求响应时间），是能够反映服务端响应速度的重要指标，获取在接收到响应的首字节前花费的毫秒数。

由于是slow3G网速了，各个资源加载时间都相对加长了。可以分析出页面资源加载顺序如下：
资源名称	说明
Index.html	Html文件，首先加载的资源文件
Angular相关模块资源文件	Angular 所有模块资源文件
index-eb0265c4eb.css	积分商城首页css文件
index_controller-34239bfb19.js	积分商城首页js文件
Mama.js	Mama100 公共方法文件
Common.js	Mama100 公共配置文件
Lazyload.js	懒加载核心文件
Md5.min.js/pointStat_service.js	埋点模块文件
Weixin.js/js_sdk.js	微信sdk 相关文件

根据这个加载瀑布流，可以发现资源下载非常多，浏览器的单线程加载，导致许多资源被挂起等待，而且与服务器交互的ttfb时间比较久。
### （2）performance 模块是记录了页面加载的全过程，包括一些函数的调用，耗时等。为了达到移动端的要求，我将cpu调低了2倍速度，然后网速选择slow3G。下面来详细说说加载过程。
 ![](https://github.com/Damonwu88/fe-performance-analysis/blob/master/3.png)

a、	资源加载顺序与network一样，由图可以看出白屏时间大约接近1000ms。也就是说需要花1秒时间页面才可以看到有内容展示。

b、	在ScriptStreamerThread 选项中，可以通过配置script 的async/defer 选择使用script streaming模式，可以使得解析时间减少10-20%。原理是允许html解析器能够先检测到资源，将解析工作分配给script streaming线程，从而不阻塞文档解析。
 ![](https://github.com/Damonwu88/fe-performance-analysis/blob/master/4.png)
 
c、	页面加载时间段是页面加载完成时间-白屏时间，即是10.29s – 1s=9.29s。页面加载完成时间是和业务关系最为密切的时间点，大量JS业务逻辑都在这个时间点触发。在这个时间段期间，页面由杂乱无章到页面完整渲染，而且用户是无法操作的。

## 解决方案
 （1）提高基础网络质量； http请求头设置与服务器配置，使用expires、cache-control/max-age积极缓存；使用last-modified/Etag 更新缓存。

 （2）使用构建工具，压缩html/js/css资源，使用gzip压缩传输。

 （3）优化资源加载顺序，启用多域名并行下载。一般都会把一个域名下载资源分散到3个左右域名下，同时下载。启用cdn 资源库域名 https://cdn.xxx.com

 （4）公共的js文件放在head里面，业务的js建议通过动态加载，根据业务动态创建script标签引入js

 （5）切分首屏可视与不可视模块，根据屏幕滚动进行懒加载与预加载处理

 （6）使用阿里云服务，提升服务器响应速度，减少ttfb延迟等待

 （7）使用async 与 defer 进行文件异步加载，启用浏览器script streaming 解析线程

 （8）CSS位于顶部，JS位于底部，减少白屏时间

性能优化示例：http://stevesouders.com/hpws/rule-min-http.php
