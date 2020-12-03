# Containerd

Containerd是一个容器运行时，可以在宿主机中管理完整的容器生命周期：容器镜像的传输和存储、容器的执行和管理、存储和网络等。

containerd是从docker中抽离出来的，并提供了实现了cri接口（cri-containerd组件）使得containerd可以作为kubelet的运行时。这样相比docker的调用少了一层dockershim(在kubelet内置的实现的cri接口)：

![cri-containerd architecture](https://silenceper.oss-cn-beijing.aliyuncs.com/images/cri-containerd.png)



这是在containerd1.0上的调用关系，在这里cri-containerd还是作为独立的进程，相比dockershim的调度少了一层。在1.1之后这个将cri-containerd嵌入到containerd，调用层级又少了一层。

![containerd architecture](https://silenceper.oss-cn-beijing.aliyuncs.com/images/containerd.png)

这样相当于调用关系少了两层，所以在性能上肯定会优于默认的dockershim调用docker的方式。



Containerd 内置的 CRI 插件实现了 Kubelet CRI 接口中的 Image Service 和 Runtime Service，通过内部接口管理容器和镜像，并通过 CNI 插件给 Pod 配置网络。

![img](https://gblobscdn.gitbook.com/assets%2F-LDAOok5ngY4pc1lEDes%2F-LpOIkR-zouVcB8QsFj_%2F-LpOIs1LZxBhsxPgKp4w%2Fcontainerd.png?alt=media)

## 安装

安装参考官方文档：https://github.com/containerd/containerd/blob/master/docs/ops.md

注意：可以通过`containerd default> /etc/containerd/config.toml`的方式生成默认的配置文件。



**kubelet配置containerd**

```
kubelet --container-runtime=remote --container-runtime-endpoint=unix:///run/containerd/containerd.sock --image-service-endpoint=unix:///run/containerd/containerd.sock
```





