# 示例 - 远程Web文件访问加速
通过[Alluxio](https://www.alluxio.io)和[Fuse](https://github.com/libfuse/libfuse)，Fluid为用户提供了一种更为简单的文件访问接口，使得任意运行在Kubernetes集群上的程序能够像访问本地文件一样轻松访问存储在远程文件系统中的文件。更为重要的是，Fluid借助Alluxio提供了强大的文件缓存能力，这意味着用户在访问远程文件时，尤其是那些具有较高访问频率的远程文件时，用户可以享受到大幅度的文件访问速度的提升。

本文档通过一个简单的例子演示了上述功能特性

## 前提条件
在运行该示例之前，请参考[安装文档](../userguide/install.md)完成安装，并检查Fluid各组件正常运行：
```shell
$ kubectl get pod -n fluid-system
NAME                                  READY   STATUS    RESTARTS   AGE
alluxioruntime-controller-5b64fdbbb-84pc6   1/1     Running   0          8h
csi-nodeplugin-fluid-fwgjh                  2/2     Running   0          8h
csi-nodeplugin-fluid-ll8bq                  2/2     Running   0          8h
dataset-controller-5b7848dbbb-n44dj         1/1     Running   0          8h
```
通常来说，你会看到一个名为`controller-manager`的Pod和多个名为`csi-nodeplugin`的Pod正在运行。其中，`csi-nodeplugin`这些Pod的数量取决于你的Kubernetes集群中结点的数量。

## 新建工作环境
```shell
$ mkdir <any-path>/accelerate
$ cd <any-path>/accelerate
```

## 运行示例

**查看待创建的Dataset资源对象**
```shell
$ cat<<EOF >dataset.yaml
apiVersion: data.fluid.io/v1alpha1
kind: Dataset
metadata:
  name: hbase
spec:
  mounts:
    - mountPoint: https://mirrors.tuna.tsinghua.edu.cn/apache/hbase/stable/
      name: hbase
EOF
```
> 注意: 上述`mountPoint`中使用了Apache清华镜像源进行演示，如果你的环境位于海外，请更换为`https://downloads.apache.org/hbase/stable/`进行尝试

在这里，我们将要创建一个kind为`Dataset`的资源对象(Resource object)。`Dataset`是Fluid所定义的一个Custom Resource Definition(CRD)，该CRD被用来告知Fluid在哪里可以找到你所需要的数据。Fluid将该CRD对象中定义的`mountPoint`属性挂载到Alluxio之上，因此该属性可以是任何合法的能够被Alluxio识别的UFS地址。在本示例中，为了简单，我们使用[WebUFS](https://docs.alluxio.io/os/user/stable/cn/ufs/WEB.html)进行演示。

更多有关UFS的信息，请参考[Alluxio文档-底层存储系统](https://docs.alluxio.io/os/user/stable/cn/ufs/OSS.html)部分。

> 本示例将以Apache镜像站点上的Hbase v2.25相关资源作为演示中使用的远程文件。这个选择并没有任何特殊之处，你可以将这个远程文件修改为任意你喜欢的远程文件。但是，如果你想要和我们一样使用WebUFS进行操作的话，最好还是选择一个Apache镜像源站点( e.g. [清华镜像源](https://mirrors.tuna.tsinghua.edu.cn/apache) )，因为根据目前WebUFS的实现，如果你选择其他更加复杂的网页作为WebUFS，你可能需要进行更多[更复杂的配置](https://docs.alluxio.io/os/user/stable/cn/ufs/WEB.html#%E9%85%8D%E7%BD%AEalluxio) 

**创建Dataset资源对象**
```shell
$ kubectl create -f dataset.yaml
dataset.data.fluid.io/hbase created
```

**查看Dataset资源对象状态**
```shell
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE      AGE
hbase                                                                  NotBound   13s
```

如上所示，`status`中的`phase`属性值为`NotBound`，这意味着该`Dataset`资源对象目前还未与任何`AlluxioRuntime`资源对象绑定，接下来，我们将创建一个`AlluxioRuntime`资源对象。

**查看待创建的AlluxioRuntime资源对象**
```shell 
$ cat<<EOF >runtime.yaml
apiVersion: data.fluid.io/v1alpha1
kind: AlluxioRuntime
metadata:
  name: hbase
spec:
  replicas: 2
  tieredstore:
    levels:
      - mediumtype: MEM
        path: /dev/shm
        quota: 2Gi
        high: "0.95"
        low: "0.7"
EOF
```

**创建AlluxioRuntime资源对象**
```shell
$ kubectl create -f runtime.yaml
alluxioruntime.data.fluid.io/hbase created
```

**检查AlluxioRuntime资源对象是否已经创建**
```shell
$ kubectl get alluxioruntime
NAME    AGE
hbase   55s
```

`AlluxioRuntime`是另一个Fluid定义的CRD。一个`AlluxioRuntime`资源对象描述了在Kubernetes集群中运行一个Alluxio实例所需要的配置信息。

等待一段时间，让AlluxioRuntime资源对象中的各个组件得以顺利启动，你会看到类似以下状态：
```shell
$ kubectl get pod
NAME                 READY   STATUS    RESTARTS   AGE
hbase-master-0       2/2     Running   0          62s
hbase-worker-0       2/2     Running   0          27s
hbase-worker-1       2/2     Running   0          27s
```

**再次查看Dataset资源对象状态**
```shell
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED   CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hbase   443.5MiB         0B       4GiB             0%                  Bound   2m39s
```
因为已经与一个成功启动的AlluxioRuntime绑定，该Dataset资源对象的状态得到了更新，此时`PHASE`属性值已经变为`Bound`状态。通过上述命令可以获知有关资源对象的基本信息

**查看AlluxioRuntime状态**
```shell
$ kubectl get alluxioruntime hbase -o wide
NAME    READY MASTERS   DESIRED MASTERS   MASTER PHASE   READY WORKERS   DESIRED WORKERS   WORKER PHASE   READY FUSES   DESIRED FUSES   FUSE PHASE   AGE
hbase   1               1                 Ready          2               2                 Ready          0             0               Ready        2m50s
```
`AlluxioRuntime`资源对象的`status`中包含了更多更详细的信息

**查看与远程文件关联的PersistentVolume以及PersistentVolumeClaim**
```shell
$ kubectl get pv
NAME    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM           STORAGECLASS   REASON   AGE
hbase   100Gi      RWX            Retain           Bound    default/hbase                           18m
```

```shell
$ kubectl get pvc
NAME    STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
hbase   Bound    hbase    100Gi      RWX                           18m
```
`Dataset`资源对象准备完成后（即与Alluxio实例绑定后），与该资源对象关联的PV, PVC已经由Fluid生成，应用可以通过该PVC完成远程文件在Pod中的挂载，并通过挂载目录实现远程文件访问

## 远程文件访问

**查看待创建的应用**
```shell
$ cat<<EOF >nginx.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - mountPath: /data
          name: hbase-vol
  volumes:
    - name: hbase-vol
      persistentVolumeClaim:
        claimName: hbase
EOF
```

**启动应用进行远程文件访问**
```shell
$ kubectl create -f nginx.yaml
```

登录Nginx Pod:
```shell
$ kubectl exec -it nginx -- bash
```

查看远程文件挂载情况：
```shell
$ ls -1 /data/hbase
CHANGES.md
RELEASENOTES.md
api_compare_2.2.5RC0_to_2.2.4.html
hbase-2.2.5-bin.tar.gz
hbase-2.2.5-client-bin.tar.gz
hbase-2.2.5-src.tar.gz
```

```shell
$ du -h /data/hbase/*
174K    /data/hbase/CHANGES.md
106K    /data/hbase/RELEASENOTES.md
115K    /data/hbase/api_compare_2.2.5RC0_to_2.2.4.html
211M    /data/hbase/hbase-2.2.5-bin.tar.gz
200M    /data/hbase/hbase-2.2.5-client-bin.tar.gz
34M     /data/hbase/hbase-2.2.5-src.tar.gz
```

登出Nginx Pod:
```shell
$ exit
```

正如你所见，WebUFS上所存储的全部文件(也就是hbase v2.2.5的相关文件)可以和本地文件完全没有区别的方式存在于某个Pod中，并且可以被该Pod十分方便地访问

## 远程文件访问加速

为了演示在访问远程文件时，你能获得多大的加速效果，我们提供了一个测试作业的样例:

**查看待创建的测试作业**
```shell
$ cat<<EOF >app.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: fluid-copy-test
spec:
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: busybox
          image: busybox
          command: ["/bin/sh"]
          args: ["-c", "set -x; time cp -r /data/hbase ./"]
          volumeMounts:
            - mountPath: /data
              name: hbase-vol
      volumes:
        - name: hbase-vol
          persistentVolumeClaim:
            claimName: hbase
EOF
```

**启动测试作业**
```shell
$ kubectl create -f app.yaml
job.batch/fluid-test created
```

该测试程序会执行`time cp -r /data/hbase ./`的shell命令，其中`/data/hbase`是远程文件在Pod中挂载的位置，该命令完成后会在终端显示命令执行的时长：

等待一段时间,待该作业运行完成,作业的运行状态可通过以下命令查看:
```shell
$ kubectl get pod
NAME                    READY   STATUS      RESTARTS   AGE
fluid-copy-test-h59w9   0/1     Completed   0          1m25s
...
```
如果看到如上结果,则说明该作业已经运行完成

> 注意: `fluid-copy-test-h59w9`中的`h59w9`为作业生成的标识,在你的环境中,这个标识可能不同,接下来的命令中涉及该标识的地方请以你的环境为准

**查看测试作业完成时间**
```shell
$ kubectl logs fluid-copy-test-h59w9
+ time cp -r /data/hbase ./
real  0m 41.21s
user  0m 0.00s
sys   0m 1.35s
```

可见，第一次远程文件的读取耗费了接近41s的时间。当然，你可能会觉得这并没有你预期的那么快，但是：

**查看Dataset资源对象状态**
```shell
$ kubectl get dataset hbase
NAME    UFS TOTAL SIZE   CACHED     CACHE CAPACITY   CACHED PERCENTAGE   PHASE   AGE
hbase   443.5MiB         443.5MiB   4GiB             100%                Bound   9m27s
```
现在，所有远程文件都已经被缓存在了Alluxio中

**再次启动测试作业**
```shell
$ kubectl delete -f app.yaml
$ kubectl create -f app.yaml
```

由于远程文件已经被缓存，此次测试作业能够迅速完成：
```shell
$ kubectl get pod
NAME                    READY   STATUS      RESTARTS   AGE
fluid-copy-test-d9h2x   0/1     Completed   0          24s
...
```

```shell
$ kubectl logs fluid-copy-test-d9h2x
+ time cp -r /data/hbase ./
real  0m 0.40s
user  0m 0.00s
sys   0m 1.27s
```
同样的文件访问操作仅耗费了0.4s

这种大幅度的加速效果归因于Alluxio所提供的强大的缓存能力，这种缓存能力意味着，只要你访问某个远程文件一次，该文件就会被缓存在Alluxio中，你的所有接下来的重复访问都不再需要进行远程文件读取，而是从Alluxio中直接获取数据，因此对于数据的访问加速也就不难解释了。

> 注意： 上述文件的访问速度与示例运行环境的网络条件有关，如果文件访问速度过慢，请更换更小的远程文件尝试

## 环境清理
```shell
$ kubectl delete -f .
```

