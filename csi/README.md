

# CSI - 容器存储接口

[容器存储接口](https://github.com/container-storage-interface/spec/blob/master/spec.md)（CSI）是用于将任意块和文件存储系统暴露给诸如Kubernetes之类的容器编排系统（CO）上的容器化工作负载的标准。 使用CSI的第三方存储提供商可以编写和部署在Kubernetes中公开新存储系统的插件，而无需接触核心的Kubernetes代码。



具体来说，Kubernetes针对CSI规定了以下内容：

- Kubelet到CSI驱动程序的通信
  - Kubelet通过Unix域套接字直接向CSI驱动程序发起CSI调用（例如`NodeStageVolume`，`NodePublishVolume`等），以挂载和卸载卷。
  - Kubelet通过kubelet插件注册机制发现CSI驱动程序（以及用于与CSI驱动程序进行交互的Unix域套接字）。
  - 因此，部署在Kubernetes上的所有CSI驱动程序**必须**在每个受支持的节点上使用kubelet插件注册机制进行注册。
- Master到CSI驱动程序的通信
  - Kubernetes master组件不会直接（通过Unix域套接字或其他方式）与CSI驱动程序通信。
  - Kubernetes master组件仅与Kubernetes API交互。
  - 因此，需要依赖于Kubernetes API的操作的CSI驱动程序（例如卷创建，卷attach，卷快照等）必须监听Kubernetes API并针对它触发适当的CSI操作（例如下面的一系列的external组件）。

## 组件

![CSI调用说明](https://silenceper.oss-cn-beijing.aliyuncs.com/images/1*bkMOGMuyCYXH8ZyR5Tngig.png)



CSI实现中的组件分为两部分：

- 由k8s官方维护的一系列external组件负责注册CSI driver 或监听k8s对象资源，从而发起csi driver调用，比如（node-driver-registrar，external-attacher，external-provisioner，external-resizer，external-snapshotter，livenessprobe）
- 各云厂商or开发者自行开发的组件（需要实现CSI Identity，CSI Controller，CSI Node 接口）



### RPC接口(开发商实现)

**Identity Service**

```go
service Identity {
  //返回driver的信息，比如名字，版本
  rpc GetPluginInfo(GetPluginInfoRequest)
    returns (GetPluginInfoResponse) {}
  //返回driver提供的能力，比如是否提供Controller Service,volume 访问能能力
  rpc GetPluginCapabilities(GetPluginCapabilitiesRequest)
    returns (GetPluginCapabilitiesResponse) {}
  //探针
  rpc Probe (ProbeRequest)
    returns (ProbeResponse) {}
}
```

**Controller service**

```go
service Controller {
  //创建卷
  rpc CreateVolume (CreateVolumeRequest)
    returns (CreateVolumeResponse) {}
  //删除卷
  rpc DeleteVolume (DeleteVolumeRequest)
    returns (DeleteVolumeResponse) {}
  //attach 卷
  rpc ControllerPublishVolume (ControllerPublishVolumeRequest)
    returns (ControllerPublishVolumeResponse) {}
  //unattach卷
  rpc ControllerUnpublishVolume (ControllerUnpublishVolumeRequest)
    returns (ControllerUnpublishVolumeResponse) {}
  //返回存储卷的功能点，如是否支持挂载到多个节点上，是否支持多个节点同时读写
  rpc ValidateVolumeCapabilities (ValidateVolumeCapabilitiesRequest)
    returns (ValidateVolumeCapabilitiesResponse) {}
  //列出所有卷
  rpc ListVolumes (ListVolumesRequest)
    returns (ListVolumesResponse) {}
  //返回存储资源池的可用空间大小
  rpc GetCapacity (GetCapacityRequest)
    returns (GetCapacityResponse) {}
  //返回controller插件的功能点，如是否支持GetCapacity接口，是否支持snapshot功能等
  rpc ControllerGetCapabilities (ControllerGetCapabilitiesRequest)
    returns (ControllerGetCapabilitiesResponse) {}
  //创建快照
  rpc CreateSnapshot (CreateSnapshotRequest)
    returns (CreateSnapshotResponse) {}
  //删除快照
  rpc DeleteSnapshot (DeleteSnapshotRequest)
    returns (DeleteSnapshotResponse) {}
  //列出快照
  rpc ListSnapshots (ListSnapshotsRequest)
    returns (ListSnapshotsResponse) {}
  //扩容
  rpc ControllerExpandVolume (ControllerExpandVolumeRequest)
    returns (ControllerExpandVolumeResponse) {}
  //获得卷
  rpc ControllerGetVolume (ControllerGetVolumeRequest)
    returns (ControllerGetVolumeResponse) {
        option (alpha_method) = true;
    }
}
```

**Node Service**

```go
service Node {
  //如果存储卷没有格式化，首先要格式化。然后把存储卷mount到一个临时的目录（这个目录通常是节点上的一个全局目录）。再通过NodePublishVolume将存储卷mount到pod的目录中。mount过程分为2步，原因是为了支持多个pod共享同一个volume（如NFS）。
  rpc NodeStageVolume (NodeStageVolumeRequest)
    returns (NodeStageVolumeResponse) {}
  //NodeStageVolume的逆操作，将一个存储卷从临时目录umount掉
  rpc NodeUnstageVolume (NodeUnstageVolumeRequest)
    returns (NodeUnstageVolumeResponse) {}
  //将存储卷从临时目录mount到目标目录（pod目录）
  rpc NodePublishVolume (NodePublishVolumeRequest)
    returns (NodePublishVolumeResponse) {}
  //将存储卷从pod目录umount掉
  rpc NodeUnpublishVolume (NodeUnpublishVolumeRequest)
    returns (NodeUnpublishVolumeResponse) {}
  //返回可用于该卷的卷容量统计信息。
  rpc NodeGetVolumeStats (NodeGetVolumeStatsRequest)
    returns (NodeGetVolumeStatsResponse) {}

  //noe上执行卷扩容
  rpc NodeExpandVolume(NodeExpandVolumeRequest)
    returns (NodeExpandVolumeResponse) {}

  //返回Node插件的功能点，如是否支持stage/unstage功能
  rpc NodeGetCapabilities (NodeGetCapabilitiesRequest)
    returns (NodeGetCapabilitiesResponse) {}
  //返回节点信息
  rpc NodeGetInfo (NodeGetInfoRequest)
    returns (NodeGetInfoResponse) {}
}
```



###External 组件（k8s Team）

这部分组件是由k8s官方提供的，作为k8s api跟csi driver的桥梁：

- **node-driver-registrar**

  CSI node-driver-registrar是一个sidecar容器，可从CSI driver获取驱动程序信息（使用NodeGetInfo），并使用kubelet插件注册机制在该节点上的kubelet中对其进行注册。

- **external-attacher**

  它是一个sidecar容器，用于监视Kubernetes VolumeAttachment对象并针对驱动程序端点触发CSI ControllerPublish和ControllerUnpublish操作

- **external-provisioner**

  它是一个sidecar容器，用于监视Kubernetes PersistentVolumeClaim对象并针对驱动程序端点触发CSI CreateVolume和DeleteVolume操作。
  external-attacher还支持快照数据源。 如果将快照CRD资源指定为PVC对象上的数据源，则此sidecar容器通过获取SnapshotContent对象获取有关快照的信息，并填充数据源字段，该字段向存储系统指示应使用指定的快照填充新卷 。

- **external-resizer**

  它是一个sidecar容器，用于监视Kubernetes API服务器上的PersistentVolumeClaim对象的改动，如果用户请求在PersistentVolumeClaim对象上请求更多存储，则会针对CSI端点触发ControllerExpandVolume操作。

- **external-snapshotter**

  它是一个sidecar容器，用于监视Kubernetes API服务器上的VolumeSnapshot和VolumeSnapshotContent CRD对象。创建新的VolumeSnapshot对象（引用与此驱动程序对应的SnapshotClass CRD对象）将导致sidecar容器提供新的快照。
  该Sidecar侦听指示成功创建VolumeSnapshot的服务，并立即创建VolumeSnapshotContent资源。

- **livenessprobe**

  它是一个sidecar容器，用于监视CSI驱动程序的运行状况，并通过[Liveness Probe机制](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)将其报告给Kubernetes。 这使Kubernetes能够自动检测驱动程序问题并重新启动Pod以尝试解决问题。



## 参考

- [kubernetes-csi-introduction](https://kubernetes-csi.github.io/docs/introduction.html)
- [详解 Kubernetes Volume 的实现原理](https://draveness.me/kubernetes-volume/)

- [CSI存储接口解释](https://www.dazhuanlan.com/2020/01/31/5e33a33ba05d1/)