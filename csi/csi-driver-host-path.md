# csi-driver-host-path安装

> 集群 v1.19.0

## 安装

https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md

### VolumeSnapshot CRDs and snapshot controller installation

```shell

# Apply VolumeSnapshot CRDs  version:v2.0.1
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v2.0.1/config/crd/snapshot.storage.k8s.io_volumesnapshotclasses.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v2.0.1/config/crd/snapshot.storage.k8s.io_volumesnapshotcontents.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v2.0.1/config/crd/snapshot.storage.k8s.io_volumesnapshots.yaml
# Create snapshot controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v2.0.1/deploy/kubernetes/snapshot-controller/rbac-snapshot-controller.yaml
kubectl apply -f https://raw.githubusercontent.com/kubernetes-csi/external-snapshotter/v2.0.1/deploy/kubernetes/snapshot-controller/setup-snapshot-controller.yaml
```

### Deployment

代码地址：https://github.com/kubernetes-csi/csi-driver-host-path

```shell
sh deploy/kubernetes-latest/deploy.sh
```

安装成功：

```shell
➜  ~ kubectl get pod|grep csi
csi-hostpath-attacher-0            1/1     Running   0          76m
csi-hostpath-provisioner-0         1/1     Running   0          76m
csi-hostpath-resizer-0             1/1     Running   1          76m
csi-hostpath-snapshotter-0         1/1     Running   0          76m
csi-hostpath-socat-0               1/1     Running   0          76m
csi-hostpathplugin-0               3/3     Running   1          76m
my-csi-app                         1/1     Running   0          46m
```

因为验证而已，这里`csi-hostpathplugin`组件只部署了一个，provisioner和attacher组件都会调用这个组件上来

### 验证

部署测试应用，storageClass,pvc,app:

```shell
$ for i in ./examples/csi-storageclass.yaml ./examples/csi-pvc.yaml ./examples/csi-app.yaml; do kubectl apply -f $i; done
storageclass.storage.k8s.io/csi-hostpath-sc created
persistentvolumeclaim/csi-pvc created
pod/my-csi-app created

```

查看pv已自动创建出来了

```shell
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS      REASON   AGE
pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e   1Gi        RWO            Delete           Bound    default/csi-pvc   csi-hostpath-sc            50m

$ kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS      AGE
csi-pvc   Bound    pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e   1Gi        RWO            csi-hostpath-sc   50m
```

> 在节点上的/var/lib/csi-hostpath-data/目录下已经有目录创建出来了



可以通过查看容器`csi-hostpathplugin-0 `的log来看整个pv的创建以及attach过程：

```shell
I1116 06:54:50.518286       1 server.go:117] GRPC call: /csi.v1.Controller/CreateVolume
I1116 06:54:50.518384       1 server.go:118] GRPC request: {"accessibility_requirements":{"preferred":[{"segments":{"topology.hostpath.csi/node":"minikube"}}],"requisite":[{"segments":{"topology.hostpath.csi/node":"minikube"}}]},"capacity_range":{"required_bytes":1073741824},"name":"pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e","volume_capabilities":[{"AccessType":{"Mount":{}},"access_mode":{"mode":1}}]}
I1116 06:54:50.581056       1 controllerserver.go:165] created volume 9f684dbc-27d8-11eb-884a-0242ac120016 at path /csi-data-dir/9f684dbc-27d8-11eb-884a-0242ac120016
I1116 06:54:50.581129       1 server.go:123] GRPC response: {"volume":{"accessible_topology":[{"segments":{"topology.hostpath.csi/node":"minikube"}}],"capacity_bytes":1073741824,"volume_id":"9f684dbc-27d8-11eb-884a-0242ac120016"}}
I1116 06:54:52.020457       1 server.go:117] GRPC call: /csi.v1.Identity/Probe
I1116 06:54:56.879104       1 server.go:117] GRPC call: /csi.v1.Node/NodeGetCapabilities
I1116 06:54:56.879169       1 server.go:118] GRPC request: {}
I1116 06:54:56.879866       1 server.go:123] GRPC response: {"capabilities":[{"Type":{"Rpc":{"type":1}}},{"Type":{"Rpc":{"type":3}}}]}
I1116 06:54:57.020014       1 server.go:117] GRPC call: /csi.v1.Node/NodeStageVolume
I1116 06:54:57.020087       1 server.go:118] GRPC request: {"staging_target_path":"/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/globalmount","volume_capability":{"AccessType":{"Mount":{}},"access_mode":{"mode":1}},"volume_context":{"storage.kubernetes.io/csiProvisionerIdentity":"1605508257344-8081-hostpath.csi.k8s.io"},"volume_id":"9f684dbc-27d8-11eb-884a-0242ac120016"}
I1116 06:54:57.022399       1 server.go:123] GRPC response: {}
I1116 06:54:57.041971       1 server.go:117] GRPC call: /csi.v1.Node/NodeGetCapabilities
I1116 06:54:57.042036       1 server.go:118] GRPC request: {}
I1116 06:54:57.043319       1 server.go:123] GRPC response: {"capabilities":[{"Type":{"Rpc":{"type":1}}},{"Type":{"Rpc":{"type":3}}}]}
I1116 06:54:57.068101       1 server.go:117] GRPC call: /csi.v1.Node/NodePublishVolume
I1116 06:54:57.068159       1 server.go:118] GRPC request: {"staging_target_path":"/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/globalmount","target_path":"/var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount","volume_capability":{"AccessType":{"Mount":{}},"access_mode":{"mode":1}},"volume_context":{"csi.storage.k8s.io/ephemeral":"false","csi.storage.k8s.io/pod.name":"my-csi-app","csi.storage.k8s.io/pod.namespace":"default","csi.storage.k8s.io/pod.uid":"9c5aa371-e5a7-4b67-8795-ec7013811363","csi.storage.k8s.io/serviceAccount.name":"default","storage.kubernetes.io/csiProvisionerIdentity":"1605508257344-8081-hostpath.csi.k8s.io"},"volume_id":"9f684dbc-27d8-11eb-884a-0242ac120016"}
I1116 06:54:57.104975       1 mount_linux.go:164] Detected OS without systemd
I1116 06:54:57.146159       1 nodeserver.go:166] target /var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount
fstype
device
readonly false
volumeId 9f684dbc-27d8-11eb-884a-0242ac120016
attributes map[csi.storage.k8s.io/ephemeral:false csi.storage.k8s.io/pod.name:my-csi-app csi.storage.k8s.io/pod.namespace:default csi.storage.k8s.io/pod.uid:9c5aa371-e5a7-4b67-8795-ec7013811363 csi.storage.k8s.io/serviceAccount.name:default storage.kubernetes.io/csiProvisionerIdentity:1605508257344-8081-hostpath.csi.k8s.io]
mountflags []
I1116 06:54:57.146375       1 mount_linux.go:164] Detected OS without systemd
I1116 06:54:57.146449       1 mount_linux.go:146] Mounting cmd (mount) with arguments ([-o bind /csi-data-dir/9f684dbc-27d8-11eb-884a-0242ac120016 /var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount])
I1116 06:54:57.448129       1 mount_linux.go:146] Mounting cmd (mount) with arguments ([-o bind,remount /csi-data-dir/9f684dbc-27d8-11eb-884a-0242ac120016 /var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount])
```

先进行create volume，在进行NodePublishVolume

#### NodeStageVolume的作用

对于块存储来说，设备只能mount到一个目录上，所以NodeStageVolume就是先mount到一个globalmount目录(类似:/var/lib/kubelet/plugins/kubernetes.io/csi/pv/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/globalmount)，然后再bind到pod的目录（/var/lib/kubelet/pods/9c5aa371-e5a7-4b67-8795-ec7013811363/volumes/kubernetes.io~csi/pvc-bcfe33ed-e822-4b0e-954a-0f5c0468525e/mount/hello-world），这样就可以pv挂载在多个pod中(NodePublishVolume)。在这一步可以进行格式化



## 参考

- https://github.com/kubernetes-csi/csi-driver-host-path/blob/master/docs/deploy-1.17-and-later.md
- https://arslan.io/2018/06/21/how-to-write-a-container-storage-interface-csi-plugin/