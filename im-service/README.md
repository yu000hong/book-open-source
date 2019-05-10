# im-service 简介

im-service是GoBelieve团队提供的一款开源IM服务，源代码在Github [GoBelieveIO/im_service](https://github.com/GoBelieveIO/im_service)。

GoBelieve官方网站：https://developer.gobelieve.io/

> GoBelieve IM 主要特点：

> 
- 一小时接入：专注IM，无冗余功能，几行代码一小时接入，省时省力。
- 自由定制：提供最新源码，自行二次开发，业务协议交互视觉均可根据业务需求自由定制。
- 完全开源：国内唯一开源IM服务，所有源码在Github开放，与线上版本一致。
- 私有云部署：彻底排除窃取隐私可能，支持自有域名部署可搭建私有云，无需更新客户端，即可热切换到私有服务，来去自由。
- IM云平台支持功能：
    1).语音，图片，文字。
    2).群组
    3).聊天室
- 提供多平台支持：
   1).IOS
   2).Android
   3).Web
- 私有服务器性能指标：单台50w并发，3000条/s消息发送量(32g,16核)，支持服务器的超大集群。

im-service是IM服务器端，另外还开源了：

- Android SDK: [im_android](https://github.com/GoBelieveIO/im_android)
- iOS SDK：[im_ios](https://github.com/GoBelieveIO/im_ios)
- JS SDK：[im_js](https://github.com/GoBelieveIO/im_js)
- React Native SDK：[im_rn](https://github.com/GoBelieveIO/im_rn)


开源出来的代码有一些问题，如：有很多重复代码、有一些bug、功能不完备(没有Token鉴权，没有用户体系)等。

im-service由三大部分组成，我把原始代码进行了拆分：

- im：https://github.com/yu000hong/im_research/tree/master/im_debug
- ims：https://github.com/yu000hong/im_research/tree/master/ims_debug
- imr：https://github.com/yu000hong/im_research/tree/master/imr_debug