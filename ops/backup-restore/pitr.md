指定时间点恢复
======

PolarDB-X Operator 从 1.4.0 版本开始支持指定时间点恢复。本文介绍如何对 PolarDB-X 进行指定时间点恢复。

## 前置条件

1. PolarDB-X Operator 升级到 1.4.0 及以上版本
2. 已经配置增量日志备份，并配置支持指定时间点恢复（默认为支持）
3. 恢复的时间点之前存在一个全量备份集
4. 全量备份集中的恢复时间到指定的恢复时间之间有连续的增量日志文件

## 相同K8S集群恢复 PolarDB-X 集群

```yaml
apiVersion: polardbx.aliyun.com/v1
kind: PolarDBXCluster
metadata:
  name: pxc-pitr-restore   # 恢复出的集群名字
spec:
  topology: # 集群规格
    nodes:
      cn:
        template:
          image: polardbx/galaxysql:latest
      dn:
        template:
          image: polardbx/galaxyengine:latest
  restore: # 指定集群的创建方式是恢复
    from:
      clusterName: polardb-x-2      # 源PolarDB-X 集群名称
    time: "2023-03-20T02:06:46Z"    # 恢复的时间点
```

参照上述示例编写恢复用的yaml文件，通过以下命令进行恢复：

```bash
kubectl apply -f pxc-pitr-restore.yaml
```

## 跨K8S集群恢复 PolarDB-X 集群
```yaml
apiVersion: polardbx.aliyun.com/v1
kind: PolarDBXCluster
metadata:
  name: pxc-pitr-restore-cross   # 恢复出的集群名字
spec:
  topology: # 集群规格
    nodes:
      cn:
        template:
          image: polardbx/galaxysql:latest
      dn:
        template:
          image: polardbx/galaxyengine:latest 
  restore:  # 指定集群的创建方式是恢复
    storageProvider:   # 数据备份集的存储配置
      storageName: xxx 
      sink: xxx
    from:
      backupSetPath: "polardbx-backup/polardb-x/pxcbackup-xxx"  #备份集的远程存储路径
    time: "2023-10-20T05:57:56Z"   # 恢复的时间点
    binlogSource:
      namespace: default  # 日志备份集的源实例的namespace，用于确定备份文件的路径
      storageProvider:    # 日志备份集的存储配置
        storageName: xx
        sink: xxx
```
参数说明
* restore.from.backupSetPath: 数据备份集的远程存储路径
* restore.storageProvider: 数据备份使用的存储配置，可参照[集群备份](./2-cluster-backup.md)
* restore.binlogSource: 日志备份的来源，包括源实例的namespace和存储配置

参照上述示例编写恢复用的yaml文件，通过以下命令进行恢复：

```bash
kubectl apply -f pxc-pitr-restore-cross.yaml
```

> 注意：
> * 如果全量备份集的时间点到指定的恢复时间之间，存在数据节点的增删操作，会导致恢复失败；
> * 如果全量备份集的时间点到指定的恢复时间之间，数据备节点发生过备库重搭等原因导致增量日志没有连续产生，会导致恢复失败;
> * 指定的恢复时间点附近如果有DDL操作，会有元数据不一致的问题

> 建议：
> * 定期做全量备份
> * 在发生数据节点的增删后，发起一次全量备份任务

其余操作步骤，可参考 [集群恢复](./3-cluster-restore.md)

