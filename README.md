# Qiniu SDK 接口标准

版本：__v8.2__

本文档基于 Qiniu 后端 SDK 定义 Qiniu SDK v8 接口标准，并提供伪代码可供参考实现。

理论上适合于所有语言及平台（前端 Javascript SDK 除外）。

对于移动端来说，后端 SDK 的伪代码由于会在主线程中使用到阻塞 IO，以及主要为高速网络优化性能，可能无法直接照搬其实现，但其设计思想依然具有参考价值。

此外，后端 SDK 需要额外配备线程池库，而移动端则一般自带线程池库，无需手动引入或配置。
