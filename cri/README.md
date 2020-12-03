# CRI 容器运行时接口

容器运行时插件（Container Runtime Interface，简称 CRI）是 Kubernetes v1.5 引入的容器运行时接口，它将 Kubelet 与容器运行时解耦，将原来完全面向 Pod 级别的内部接口拆分成面向 Sandbox 和 Container 的 gRPC 接口，并将镜像管理和容器管理分离到不同的服务。

![cri](https://silenceper.oss-cn-beijing.aliyuncs.com/images/cri.png)



> 在kubernetes 1.20的changelog中，表示未来会移除对docker的依赖，觉得dockershim放在k8s中内部维护是不合适的，而更加专注CRI的实现，提供CRI接口，又第三方组件来维护

## 容器运行时

| **CRI** **容器运行时** | **维护者** | **主要特性**                 | **容器引擎**               |
| ---------------------- | ---------- | ---------------------------- | -------------------------- |
| **Dockershim**         | Kubernetes | 内置实现、特性最新           | docker                     |
| **cri-o**              | Kubernetes | OCI标准不需要Docker          | OCI（runc、kata、gVisor…） |
| **cri-containerd**     | Containerd | 基于 containerd 不需要Docker | OCI（runc、kata、gVisor…） |
| **Frakti**             | Kubernetes | 虚拟化容器                   | hyperd、docker             |
| **rktlet**             | Kubernetes | 支持rkt                      | rkt                        |
| **PouchContainer**     | Alibaba    | 富容器                       | OCI（runc、kata…）         |
| **Virtlet**            | Mirantis   | 虚拟机和QCOW2镜像            | Libvirt（KVM）             |

目前基于 CRI 容器引擎已经比较丰富了，包括

- Docker: 核心代码依然保留在 kubelet 内部（[pkg/kubelet/dockershim](https://github.com/kubernetes/kubernetes/tree/master/pkg/kubelet/dockershim)），是最稳定和特性支持最好的运行时

- OCI 容器运行时：
  - 社区有两个实现
    - [Containerd](https://github.com/containerd/cri)，支持 kubernetes v1.7+
    - [CRI-O](https://github.com/kubernetes-incubator/cri-o)，支持 Kubernetes v1.6+
  - 支持的 OCI 容器引擎包括
    - [runc](https://github.com/opencontainers/runc)：OCI 标准容器引擎
    - [gVisor](https://github.com/google/gvisor)：谷歌开源的基于用户空间内核的沙箱容器引擎
    - [Clear Containers](https://github.com/clearcontainers/runtime)：Intel 开源的基于虚拟化的容器引擎
    - [Kata Containers](https://github.com/kata-containers/runtime)：基于虚拟化的容器引擎，由 Clear Containers 和 runV 合并而来
  
- [PouchContainer](https://github.com/alibaba/pouch)：阿里巴巴开源的胖容器引擎

- [Frakti](https://github.com/kubernetes/frakti)：支持 Kubernetes v1.6+，提供基于 hypervisor 和 docker 的混合运行时，适用于运行非可信应用，如多租户和 NFV 等场景

- [Rktlet](https://github.com/kubernetes-incubator/rktlet)：支持 [rkt](https://github.com/rkt/rkt) 容器引擎（rknetes 代码已在 v1.10 中弃用）

- [Virtlet](https://github.com/Mirantis/virtlet)：Mirantis 开源的虚拟机容器引擎，直接管理 libvirt 虚拟机，镜像须是 qcow2 格式

- [Infranetes](https://github.com/apporbit/infranetes)：直接管理 IaaS 平台虚拟机，如 GCE、AWS 等

  

## 接口描述

https://github.com/kubernetes/cri-api