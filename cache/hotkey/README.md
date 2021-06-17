# 京东毫秒级热key探测框架 - hotkey

[京东零售 / hotkey](https://gitee.com/jd-platform-opensource/hotkey)

代码水平是真的烂，各种全局变量，各种静态Holder！没有遵循面向对象的一些基本原则，如开闭原则，没有将功能实现封装到一个地方，而是共享公共变量，功能松散到各个地方。很多功能的实现也是直接使用的静态方法，因为变量也是静态公共变量！一些需要并发访问的变量也是公共变量，还能保证线程安全，666！

在阅读源码之前，先看看这几篇文章：

[京东毫秒级热key探测框架设计与实践，已实战于618大促](https://mp.weixin.qq.com/s/xOzEj5HtCeh_ezHDPHw6Jw)

[京东开源热key探测（JD-hotkey）中间件单机qps 提升17倍实战](https://tianyalei.blog.csdn.net/article/details/108347991)




