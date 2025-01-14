# CDC节点创建
PolarDB-X CDC 组件内置于 PolarDB-X 实例中，想要体验 PolarDB-X CDC 的功能，需要拉起一个 PolarDB-X 集群。

## 全局Binlog

* 通过 PXD 部署：参考 [通过PXD部署集群](https://doc.polardbx.com/quickstart/topics/quickstart-pxd-cluster.html)，可以在拓扑文件中编辑 CDC 相关的标签值指定 CDC 集群的配置。 
  * `image`：CDC 节点的镜像 
  * `replica`：CDC 节点的个数 
  * `nodes`：每个 CDC 节点的具体配置 
  * `resources`：分配给 CDC 节点的内存等资源 
* 通过 K8S 部署：参考 [通过K8S部署](https://doc.polardbx.com/quickstart/topics/quickstart-k8s.html)，默认会创建一个 CDC 节点，负责全局 Binlog 的生成。

## Binlog多流

Binlog 多流目前只支持使用 K8S 进行部署，在进行部署之前需要准备好`minikube`和`PolarDB-X Operator`环境，环境配置方法参考 [准备工作](https://doc.polardbx.com/operator/deployment/1-installation.html) 。

接下来，我们需要准备一个描述 PolarDB-X 集群的 YAML 文件，文件名为`create-cluster.yaml`，内容如下：
```yaml
apiVersion: polardbx.aliyun.com/v1
kind: PolarDBXCluster
metadata:
  name: binlogx-example
spec:
  config:
    cdc:
      envs:
        binlogx_stream_group_name: "group1"
        binlogx_stream_count: "3"
        binlogx_transmit_hash_level: "RECORD"
  topology:
    nodes:
      cdc:
        replicas: 2
        xReplicas: 2
        template:
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 500Mi
          image: polardbx/polardbx-cdc:latest
      cn:
        replicas: 1
        template:
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 1Gi
          image: polardbx/polardbx-sql:latest
      dn:
        replicas: 2
        template:
          engine: galaxy
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 500Mi
          image: polardbx/polardbx-engine-2.0:latest
      gms:
        template:
          engine: galaxy
          resources:
            limits:
              cpu: "2"
              memory: 4Gi
            requests:
              cpu: 500m
              memory: 500Mi
          image: polardbx/polardbx-engine-2.0:latest
  ```
注：目前 `PolarDB-X Operator` 仅支持拉起单个多流group，并且需要同时拉起全局Binlog。

其中多流相关的配置如下：
* `xReplicas`: 多流节点个数
* `binlogx_stream_group_name`：多流流组名称
* `binlogx_stream_count`：流的个数
* `binlogx_transmit_hash_level`：多流数据分发的哈希规则，目前支持三种规则：
  * `RECORD`：按行哈希
  * `TABLE`：按表哈希
  * `DATABASE`：按库哈希

使用下面的命令创建 PolarDB-X Cluster 对象：
```shell
kubectl create -f create-cluster.yaml
```
期望看到以下输出：
```text
polardbxcluster.polardbx.aliyun.com/binlogx-example created
```

如果有以下报错，则需要参考 [升级](https://doc.polardbx.com/operator/deployment/3-upgrade.html?h=%E5%8D%87%E7%BA%A7) 更新一下`CRD`。
```text
Error from server (BadRequest): error when creating "create-cluster.yaml": 
PolarDBXCluster in version "v1" cannot be handled as a PolarDBXCluster: 
strict decoding error: unknown field "spec.config.cdc"
```

使用下面的命令观察 PolarDB-X Cluster 对象的状态：
```shell
kubectl get pxc binlogx-example -w
```
```text
NAME              GMS   CN    DN    CDC   COLUMNAR   PHASE      DISK        AGE
binlogx-example   1/1   0/1   2/2   0/4    -         Creating               71s
binlogx-example   1/1   1/1   2/2   0/4    -         Creating               83s
binlogx-example   1/1   1/1   2/2   2/4    -         Creating               106s
binlogx-example   1/1   1/1   2/2   4/4    -         Running    10.7 GiB    2m13s
```
当状态中 PHASE 为 Running 时，PolarDB-X 集群就创建完成了。

集群拉起成功之后，可以使用下面的命令获得所有Binlog多流Pod的名称：
```shell
kubectl get pods -l polardbx/group=g-1
```
