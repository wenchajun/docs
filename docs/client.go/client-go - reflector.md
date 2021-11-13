# client-go

在说之前先引入上次讲的图：

![编写自定义控制器所依赖的组件](https://github.com/wenchajun/docs/blob/master/docs/images/%E7%BC%96%E5%86%99%E8%87%AA%E5%AE%9A%E4%B9%89%E6%8E%A7%E5%88%B6%E5%99%A8%E6%89%80%E4%BE%9D%E8%B5%96%E7%9A%84%E7%BB%84%E4%BB%B6.png)

在client-gp中我们提到过 **Reflector** 的任务就是向 apiserver watch 特定类型的资源，拿到变更通知后将其丢到 DeltaFIFO 队列中。另外前面已经在 client-go中分析过 **ListWatcher** 是如何从 apiserver 中 list-watch 资源的，今天我们继续来看 **Reflector** 的实现。



参考 [https://www.danielhu.cn](https://www.danielhu.cn/)
