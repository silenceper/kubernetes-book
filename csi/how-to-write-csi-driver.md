# 如何编写一个CSI插件

> 这里以[csi-driver-host-path](https://github.com/kubernetes-csi/csi-driver-host-path)作为例子，来看看是如何实现一个csi插件的？

**目标：**

- 支持PV动态创建，并且能够挂载在POD中
- volume来自本地目录，主要是模拟volume产生的过程，这样就不依赖于某个特定的存储服务

## 预备知识

在[上一篇文章](./README.)中，已经对CSI概念有个了解，并且提出了CSI组件需要实现的RPC接口，那我们为什么需要这些接口，这需要从volume要被使用经过了以下流程：

- volume创建
- volume `attach`到节点(比如像EBS硬盘，NFS可能就直接下一步mount了)
- volume 被`mount`到指定目录(这个目录其实就被映射到容器中，由kubelet 中的`VolumeManager` 调用)

而当卸载时正好是相反的：`unmount`,`detach`,`delete volume`

正好对应如下图：

```
   CreateVolume +------------+ DeleteVolume
 +------------->|  CREATED   +--------------+
 |              +---+----^---+              |
 |       Controller |    | Controller       v
+++         Publish |    | Unpublish       +++
|X|          Volume |    | Volume          | |
+-+             +---v----+---+             +-+
                | NODE_READY |
                +---+----^---+
               Node |    | Node
              Stage |    | Unstage
             Volume |    | Volume
                +---v----+---+
                |  VOL_READY |
                +---+----^---+
               Node |    | Node
            Publish |    | Unpublish
             Volume |    | Volume
                +---v----+---+
                | PUBLISHED  |
                +------------+
```

而为什么多个`NodeStageVolume`的过程是因为：

> 对于块存储来说，设备只能mount到一个目录上，所以`NodeStageVolume`就是先mount到一个globalmount目录(类似:`/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/globalmount`)，然后再`NodePublishVolume`这一步中通过`mount bind`到pod的目录（`/var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount/hello-world`），这样就可以实现一个pv挂载在多个pod中使用。



## 代码实现

我们并不一定要实现所有的接口，这个可以通过CSI中`Capabilities`能力标识出来，我们组件提供的能力，比如

- `IdentityServer`中的`GetPluginCapabilities`方法

- `ControllerServer`中的`ControllerGetCapabilities`方法
- `NodeServer`中的`NodeGetCapabilities` 

这些方法都是在告诉调用方，我们的组件实现了哪些能力，未实现的方法就不会调用了。

### IdentityServer

`IdentityServer`包含了三个接口，这里我们主要实现

```go
// IdentityServer is the server API for Identity service.
type IdentityServer interface {
	GetPluginInfo(context.Context, *GetPluginInfoRequest) (*GetPluginInfoResponse, error)
	GetPluginCapabilities(context.Context, *GetPluginCapabilitiesRequest) (*GetPluginCapabilitiesResponse, error)
	Probe(context.Context, *ProbeRequest) (*ProbeResponse, error)
}
```

主要看下`GetPluginCapabilities`这个方法：

[identityserver.go#L60](https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/pkg/hostpath/identityserver.go#L60):

```go
func (ids *identityServer) GetPluginCapabilities(ctx context.Context, req *csi.GetPluginCapabilitiesRequest) (*csi.GetPluginCapabilitiesResponse, error) {
	return &csi.GetPluginCapabilitiesResponse{
		Capabilities: []*csi.PluginCapability{
			{
				Type: &csi.PluginCapability_Service_{
					Service: &csi.PluginCapability_Service{
						Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
					},
				},
			},
			{
				Type: &csi.PluginCapability_Service_{
					Service: &csi.PluginCapability_Service{
						Type: csi.PluginCapability_Service_VOLUME_ACCESSIBILITY_CONSTRAINTS,
					},
				},
			},
		},
	}, nil
}
```

以上就告诉调用者我们提供了ControllerService的能力，以及volume访问限制的能力（CSI 处理时需要根据集群拓扑作调整）

> PS：其实在k8s还提供了一个包：github.com/kubernetes-csi/drivers/pkg/csi-common，里面提供了比如`DefaultIdentityServer`，`DefaultControllerServer`,`DefaultNodeServer`的struct，只要在我们自己的`XXXServer struct`中继承这些`struct`，我们的代码中就只要包含自己实现的方法就行了，可以参考[alibaba-cloud-csi-driver](https://github.com/kubernetes-sigs/alibaba-cloud-csi-driver/blob/master/pkg/disk/identityserver.go#L26)中的。

###ControllerServer

`ControllerServer`我们主要关注`CreateVolume`,`DeleteVolume`，因为是hostpath volume，所以就没有attach的这个过程了，我们放在NodeServer中实现：

#### CreateVolume

[controllerserver.go#L73](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/controllerserver.go#L73)

```go
func (cs *controllerServer) CreateVolume(ctx context.Context, req *csi.CreateVolumeRequest) (*csi.CreateVolumeResponse, error) {
  //校验参数是否有CreateVolume的能力
	if err := cs.validateControllerServiceRequest(csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME); err != nil {
		glog.V(3).Infof("invalid create volume req: %v", req)
		return nil, err
	}

  //.....这里省略的校验参数的过程


  //这里根据volume name判断是否已经存在了，存在了就返回就行了
	if exVol, err := getVolumeByName(req.GetName()); err == nil {
		// volume已经存在，但是大小不符合
		if exVol.VolSize < capacity {
			return nil, status.Errorf(codes.AlreadyExists, "Volume with the same name: %s but with different size already exist", req.GetName())
		}
    //这里判断是否设置了pvc.dataSource，就表示是一个restore过程
		if req.GetVolumeContentSource() != nil {
			volumeSource := req.VolumeContentSource
			switch volumeSource.Type.(type) {
        //校验：从快照中恢复
			case *csi.VolumeContentSource_Snapshot:
				if volumeSource.GetSnapshot() != nil && exVol.ParentSnapID != "" && exVol.ParentSnapID != volumeSource.GetSnapshot().GetSnapshotId() {
					return nil, status.Error(codes.AlreadyExists, "existing volume source snapshot id not matching")
				}
        //校验：clone过程
			case *csi.VolumeContentSource_Volume:
				if volumeSource.GetVolume() != nil && exVol.ParentVolID != volumeSource.GetVolume().GetVolumeId() {
					return nil, status.Error(codes.AlreadyExists, "existing volume source volume id not matching")
				}
			default:
				return nil, status.Errorf(codes.InvalidArgument, "%v not a proper volume source", volumeSource)
			}
		}
		// TODO (sbezverk) Do I need to make sure that volume still exists?
		return &csi.CreateVolumeResponse{
			Volume: &csi.Volume{
				VolumeId:      exVol.VolID,
				CapacityBytes: int64(exVol.VolSize),
				VolumeContext: req.GetParameters(),
				ContentSource: req.GetVolumeContentSource(),
			},
		}, nil
	}

  //创建volume
	volumeID := uuid.NewUUID().String()
  //创建hostpath的volume
	vol, err := createHostpathVolume(volumeID, req.GetName(), capacity, requestedAccessType, false /* ephemeral */)
	if err != nil {
		return nil, status.Errorf(codes.Internal, "failed to create volume %v: %v", volumeID, err)
	}
	glog.V(4).Infof("created volume %s at path %s", vol.VolID, vol.VolPath)
  
  //判断是从快照恢复，还是clone
	if req.GetVolumeContentSource() != nil {
		path := getVolumePath(volumeID)
		volumeSource := req.VolumeContentSource
		switch volumeSource.Type.(type) {
      //从快照恢复
		case *csi.VolumeContentSource_Snapshot:
			if snapshot := volumeSource.GetSnapshot(); snapshot != nil {
				err = loadFromSnapshot(capacity, snapshot.GetSnapshotId(), path, requestedAccessType)
				vol.ParentSnapID = snapshot.GetSnapshotId()
			}
      //clone
		case *csi.VolumeContentSource_Volume:
			if srcVolume := volumeSource.GetVolume(); srcVolume != nil {
				err = loadFromVolume(capacity, srcVolume.GetVolumeId(), path, requestedAccessType)
				vol.ParentVolID = srcVolume.GetVolumeId()
			}
		default:
			err = status.Errorf(codes.InvalidArgument, "%v not a proper volume source", volumeSource)
		}
		if err != nil {
			if delErr := deleteHostpathVolume(volumeID); delErr != nil {
				glog.V(2).Infof("deleting hostpath volume %v failed: %v", volumeID, delErr)
			}
			return nil, err
		}
		glog.V(4).Infof("successfully populated volume %s", vol.VolID)
	}
  
  //Topology表示volume能够部署在哪些节点（生产情况可能就对应可用区）
	topologies := []*csi.Topology{&csi.Topology{
		Segments: map[string]string{TopologyKeyNode: cs.nodeID},
	}}

	return &csi.CreateVolumeResponse{
		Volume: &csi.Volume{
			VolumeId:           volumeID,
			CapacityBytes:      req.GetCapacityRange().GetRequiredBytes(),
			VolumeContext:      req.GetParameters(),
			ContentSource:      req.GetVolumeContentSource(),
			AccessibleTopology: topologies,
		},
	}, nil
}
```

**`createHostpathVolume`**

再来看下`createHostpathVolume`方法，这里`accessType`有两个选项，是创建文件系统，还是创建块，其实就是对应pvc中`volumeMode`字段：

[pkg/hostpath/hostpath.go#L208](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/hostpath.go#L208)

```go

// createVolume create the directory for the hostpath volume.
// It returns the volume path or err if one occurs.
func createHostpathVolume(volID, name string, cap int64, volAccessType accessType, ephemeral bool) (*hostPathVolume, error) {
	path := getVolumePath(volID)

	switch volAccessType {
	case mountAccess:
    //创建文件
		err := os.MkdirAll(path, 0777)
		if err != nil {
			return nil, err
		}
	case blockAccess:
    //创建块
		executor := utilexec.New()
		size := fmt.Sprintf("%dM", cap/mib)
		// Create a block file.
		_, err := os.Stat(path)
		if err != nil {
			if os.IsNotExist(err) {
				out, err := executor.Command("fallocate", "-l", size, path).CombinedOutput()
				if err != nil {
					return nil, fmt.Errorf("failed to create block device: %v, %v", err, string(out))
				}
			} else {
				return nil, fmt.Errorf("failed to stat block device: %v, %v", path, err)
			}
		}

    // 通过losetup将文件虚拟成块设备
		// Associate block file with the loop device.
		volPathHandler := volumepathhandler.VolumePathHandler{}
		_, err = volPathHandler.AttachFileDevice(path)
		if err != nil {
			// Remove the block file because it'll no longer be used again.
			if err2 := os.Remove(path); err2 != nil {
				glog.Errorf("failed to cleanup block file %s: %v", path, err2)
			}
			return nil, fmt.Errorf("failed to attach device %v: %v", path, err)
		}
	default:
		return nil, fmt.Errorf("unsupported access type %v", volAccessType)
	}

	hostpathVol := hostPathVolume{
		VolID:         volID,
		VolName:       name,
		VolSize:       cap,
		VolPath:       path,
		VolAccessType: volAccessType,
		Ephemeral:     ephemeral,
	}
	hostPathVolumes[volID] = hostpathVol
	return &hostpathVol, nil
}
```

#### DeleteVolume

在DeleteVolume这里主要是删除volume：

[pkg/hostpath/controllerserver.go#L2](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/controllerserver.go#L208)

```go
func (cs *controllerServer) DeleteVolume(ctx context.Context, req *csi.DeleteVolumeRequest) (*csi.DeleteVolumeResponse, error) {
	// Check arguments
	if len(req.GetVolumeId()) == 0 {
		return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
	}

	if err := cs.validateControllerServiceRequest(csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME); err != nil {
		glog.V(3).Infof("invalid delete volume req: %v", req)
		return nil, err
	}

	volId := req.GetVolumeId()
	if err := deleteHostpathVolume(volId); err != nil {
		return nil, status.Errorf(codes.Internal, "failed to delete volume %v: %v", volId, err)
	}

	glog.V(4).Infof("volume %v successfully deleted", volId)

	return &csi.DeleteVolumeResponse{}, nil
}
```





在`ControllerService`中还有一些其他接口，比如`CreateSnapshot`创建快照，`DeleteSnapshot`删除快照，`扩容`等，其实都会依赖于我们存储服务端的提供的能力，调用相应的接口就行了。

### NodeServer

在`nodeServer`中就是实现我们的`mount`,`unmount`过程了，分别对应`NodePublishVolume`和`NodeUnpublishVolume`



#### NodePublishVolume

[pkg/hostpath/nodeserver.go#L5](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/nodeserver.go#L50)

```go
func (ns *nodeServer) NodePublishVolume(ctx context.Context, req *csi.NodePublishVolumeRequest) (*csi.NodePublishVolumeResponse, error) {
	//......这里省略校验参数代码

	

	vol, err := getVolumeByID(req.GetVolumeId())
	if err != nil {
		return nil, status.Error(codes.NotFound, err.Error())
	}
  //对应pvc.volumeBind字段是block的情况
	if req.GetVolumeCapability().GetBlock() != nil {
		if vol.VolAccessType != blockAccess {
			return nil, status.Error(codes.InvalidArgument, "cannot publish a non-block volume as block volume")
		}

		volPathHandler := volumepathhandler.VolumePathHandler{}

    //获取device地址（通过loopset -l命令，因为是通过文件虚拟出来的块设备）
		// Get loop device from the volume path.
		loopDevice, err := volPathHandler.GetLoopDevice(vol.VolPath)
		if err != nil {
			return nil, status.Error(codes.Internal, fmt.Sprintf("failed to get the loop device: %v", err))
		}

		mounter := mount.New("")

		// Check if the target path exists. Create if not present.
		_, err = os.Lstat(targetPath)
		if os.IsNotExist(err) {
			if err = mounter.MakeFile(targetPath); err != nil {
				return nil, status.Error(codes.Internal, fmt.Sprintf("failed to create target path: %s: %v", targetPath, err))
			}
		}
		if err != nil {
			return nil, status.Errorf(codes.Internal, "failed to check if the target block file exists: %v", err)
		}

		// Check if the target path is already mounted. Prevent remounting.
		notMount, err := mounter.IsNotMountPoint(targetPath)
		if err != nil {
			if !os.IsNotExist(err) {
				return nil, status.Errorf(codes.Internal, "error checking path %s for mount: %s", targetPath, err)
			}
			notMount = true
		}
		if !notMount {
			// It's already mounted.
			glog.V(5).Infof("Skipping bind-mounting subpath %s: already mounted", targetPath)
			return &csi.NodePublishVolumeResponse{}, nil
		}

    //进行绑定挂载（mount bind），将块设备绑定到容器目录(targetpath类似这种：/var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount)
		options := []string{"bind"}
		if err := mount.New("").Mount(loopDevice, targetPath, "", options); err != nil {
			return nil, status.Error(codes.Internal, fmt.Sprintf("failed to mount block device: %s at %s: %v", loopDevice, targetPath, err))
		}
    //对应pvc.volumeBind字段是filesystem的情况
	} else if req.GetVolumeCapability().GetMount() != nil {
		//....这里省略，因为跟上面类似也是mount bind过程
	}

	return &csi.NodePublishVolumeResponse{}, nil
}
```



####NodeUnpublishVolume

`NodeUnpublishVolume`过程就是unmount过程，如下：

[pkg/hostpath/nodeserver.go#L191](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/nodeserver.go#L191)

```go
func (ns *nodeServer) NodeUnpublishVolume(ctx context.Context, req *csi.NodeUnpublishVolumeRequest) (*csi.NodeUnpublishVolumeResponse, error) {

	// Check arguments
	if len(req.GetVolumeId()) == 0 {
		return nil, status.Error(codes.InvalidArgument, "Volume ID missing in request")
	}
	if len(req.GetTargetPath()) == 0 {
		return nil, status.Error(codes.InvalidArgument, "Target path missing in request")
	}
	targetPath := req.GetTargetPath()
	volumeID := req.GetVolumeId()

	vol, err := getVolumeByID(volumeID)
	if err != nil {
		return nil, status.Error(codes.NotFound, err.Error())
	}

	// Unmount only if the target path is really a mount point.
	if notMnt, err := mount.IsNotMountPoint(mount.New(""), targetPath); err != nil {
		if !os.IsNotExist(err) {
			return nil, status.Error(codes.Internal, err.Error())
		}
	} else if !notMnt {
		// Unmounting the image or filesystem.
		err = mount.New("").Unmount(targetPath)
		if err != nil {
			return nil, status.Error(codes.Internal, err.Error())
		}
	}
	// Delete the mount point.
	// Does not return error for non-existent path, repeated calls OK for idempotency.
	if err = os.RemoveAll(targetPath); err != nil {
		return nil, status.Error(codes.Internal, err.Error())
	}
	glog.V(4).Infof("hostpath: volume %s has been unpublished.", targetPath)

	if vol.Ephemeral {
		glog.V(4).Infof("deleting volume %s", volumeID)
		if err := deleteHostpathVolume(volumeID); err != nil && !os.IsNotExist(err) {
			return nil, status.Error(codes.Internal, fmt.Sprintf("failed to delete volume: %s", err))
		}
	}

	return &csi.NodeUnpublishVolumeResponse{}, nil
}
```

### 启动grpc server

[pkg/hostpath/hostpath.go#L164](https://github.com/kubernetes-csi/csi-driver-host-path/blob/b1cfe85dd7bfffce2bbe5b1228c994a9bc3649fb/pkg/hostpath/hostpath.go#L164)

```go
func (hp *hostPath) Run() {
	// Create GRPC servers
	hp.ids = NewIdentityServer(hp.name, hp.version)
	hp.ns = NewNodeServer(hp.nodeID, hp.ephemeral, hp.maxVolumesPerNode)
	hp.cs = NewControllerServer(hp.ephemeral, hp.nodeID)

  
	s := NewNonBlockingGRPCServer()
	s.Start(hp.endpoint, hp.ids, hp.cs, hp.ns)
	s.Wait()
}
```



##测试

我们可以通过[csc](https://github.com/rexray/gocsi/tree/master/csc)工具来进行grpc接口的测试：

```sh
$ GO111MODULE=off go get -u github.com/rexray/gocsi/csc
```

**Get plugin info**

```
$ csc identity plugin-info --endpoint tcp://127.0.0.1:10000
"csi-hostpath"  "0.1.0"
```

**Create a volume**

```
$ csc controller new --endpoint tcp://127.0.0.1:10000 --cap 1,block CSIVolumeName
CSIVolumeID
```

**Delete a volume**

```
$ csc controller del --endpoint tcp://127.0.0.1:10000 CSIVolumeID
CSIVolumeID
```

**Validate volume capabilities**

```
$ csc controller validate-volume-capabilities --endpoint tcp://127.0.0.1:10000 --cap 1,block CSIVolumeID
CSIVolumeID  true
```

**NodePublish a volume**

```
$ csc node publish --endpoint tcp://127.0.0.1:10000 --cap 1,block --target-path /mnt/hostpath CSIVolumeID
CSIVolumeID
```

**NodeUnpublish a volume**

```
$ csc node unpublish --endpoint tcp://127.0.0.1:10000 --target-path /mnt/hostpath CSIVolumeID
CSIVolumeID
```

**Get Nodeinfo**

```
$ csc node get-info --endpoint tcp://127.0.0.1:10000
CSINode
```



## 部署

从上一篇文章中我们可以看到，CSI真正运行起来，其实还需要一些官方提供的组件进行配合，比如`node-driver-registrar`，`csi-provision`，`csi-attacher`，我们将这些container作为我们的sidecar容器，通过volume共享socket连接，方便调用，部署在一起。

我们把服务分为两个部分：

- controller ：以Deployment或者Statefulset方式部署，通过leader selector，控制只有一个在工作。 
- node：以DaemonSet方式部署，在每个节点上都调度

> hostpath因为只有在单个节点上测试用，所以它的都使用了Statefulset，因为只是测试。

在生产部署的话可以参考[csi-driver-nfs]( https://github.com/kubernetes-csi/csi-driver-nfs/tree/master/deploy) 服务的部署，这个服务比较完整。

- https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/csi-nfs-node.yaml
- https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/deploy/csi-nfs-controller.yaml

当然还有一些rbac，CSIDriver的创建，这里就不贴出来了。



## 总结

回顾下整个组件是怎么协调工作的：

- `csi-provisioner`组件监听pvc的创建，从而通过 CSI socket 创建 [CreateVolumeRequest](https://github.com/container-storage-interface/spec/blob/master/spec.md#createvolume) 请求至`CreateVolume`方法
- `csi-provisioner`创建 PV 以及更新 PVC状态至  bound ，从而由 controller-manager创建`VolumeAttachment`对象
- `csi-attacher` 监听`VolumeAttachments` 对象创建，从而调用[ ControllerPublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#controllerpublishvolume) 方法。
- `kubelet`一直都在等待volume attach， 从而调用 [NodeStageVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodestagevolume) (主要做格式化以及mount到节点上一个全局目录) 方法 - *这一步可选*
- CSI Driver在 在 [NodeStageVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodestagevolume) 方法中将volumemount到 `/var/lib/kubelet/plugins/kubernetes.io/csi/pv/<pv-name>/globalmount`这个目录并返回给kubelet - *这一步可选*
- `kubelet`调用[NodePublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume) (挂载到pod目录通过`mount bind`)
- CSI Driver相应[ NodePublishVolume](https://github.com/container-storage-interface/spec/blob/master/spec.md#nodepublishvolume) 请求，将volume挂载到pod目录 `/var/lib/kubelet/pods/<pod-uuid>/volumes/[kubernetes.io](http://kubernetes.io/)~csi/<pvc-name>/mount`
- 最后，kubelet启动容器



## 参考

- https://medium.com/velotio-perspectives/kubernetes-csi-in-action-explained-with-features-and-use-cases-4f966b910774

- https://kubernetes-csi.github.io/docs/developing.html

  