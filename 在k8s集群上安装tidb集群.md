# 在k8s集群上安装tidb集群

## 1.环境

centos7.5

k8s1.17.3

## 2.安装

### 1) 安装helm

\#从官网下载最指定版本的二进制安装包到本地：https://github.com/kubernetes/helm/releases 

tar -zxvf helm-2.16.3.tar.gz # 解压压缩包

把 helm 指令放到bin目录下 

mv helm-2.16.3/helm /usr/local/bin/helm 

helm help # 验证

```javascript
[root@node1 ~]# helm version
Client: &version.Version{SemVer:"v2.16.3", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
Error: could not find tiller
```

### 2）安装tiller

#### 账号与角色绑定

1. 创建名为tiller的ServiceAccount：

```javascript
kubectl create serviceaccount --namespace kube-system tiller
```

1. 把tiller与角色tiller-cluster-rule进行绑定：

```javascript
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

1. helm初始化，其中tiller的镜像来自阿里云，并且将默认仓库也设置为阿里云的：

```javascript
helm init --upgrade -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.16.3 --stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts --service-account tiller
```

1. 等待控制台提示成功后再次执行helm version，输出如下，可见helm的服务端已经返回了信息：

```javascript
[root@node1 ~]# helm version
Client: &version.Version{SemVer:"v2.16.3", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.16.3", GitCommit:"bbdfe5e7803a12bbdf97e94cd847859890cf4050", GitTreeState:"clean"}
```

### 3）为k8s提供 local volumes

参照下面：

https://github.com/kubernetes-sigs/sig-storage-local-static-provisioner/blob/master/docs/operations.md

tidb-operator启动会为pd和tikv绑定pv，需要在discovery directory下创建多个目录

格式化并挂载磁盘

```
mkfs.ext4 /dev/sdb
DISK_UUID=$(blkid -s UUID -o value /dev/sdb) 
mkdir /mnt/$DISK_UUID
mount -t ext4 /dev/sdb /mnt/$DISK_UUID
```

/etc/fstab持久化mount

```
echo UUID=`sudo blkid -s UUID -o value /dev/sdb` /mnt/$DISK_UUID ext4 defaults 0 2 | su
```

创建多个目录并mount到discovery directory

```
for i in $(seq 1 10); do
sudo mkdir -p /mnt/${DISK_UUID}/vol${i} /mnt/disks/${DISK_UUID}_vol${i}
sudo mount --bind /mnt/${DISK_UUID}/vol${i} /mnt/disks/${DISK_UUID}_vol${i}
done
```

/etc/fstab持久化mount

```
for i in $(seq 1 10); do
echo /mnt/${DISK_UUID}/vol${i} /mnt/disks/${DISK_UUID}_vol${i} none bind 0 0 | sudo tee -a /etc/fstab
done
```

为tidb-operator创建local-volume-provisioner

$ kubectl apply -f https://raw.githubusercontent.com/pingcap/tidb-operator/master/manifests/local-dind/local-volume-provisioner.yaml 

$ kubectl get po -n kube-system -l app=local-volume-provisioner 

$ kubectl get pv --all-namespaces | grep local-storage 

### 4)安装tidb-operation

```shell
mkdir -p /home/tidb/tidb-operator && \
helm inspect values pingcap/tidb-operator --version=<chart-version> > /home/tidb/tidb-operator/values-tidb-operator.yaml
```

**注意：**

`` 在后续文档中代表 chart 版本，例如 `v1.0.0`，可以通过 `helm search -l tidb-operator` 查看当前支持的版本。最好装最新版本,之前安装v1.0.3版本时，导致pd一直起不来，后来换了v1.0.6.

TiDB Operator 里面会用到 `k8s.gcr.io/kube-scheduler` 镜像，如果下载不了该镜像，可以修改 `/home/tidb/tidb-operator/values-tidb-operator.yaml` 文件中的 `scheduler.kubeSchedulerImageName` 为 `registry.cn-hangzhou.aliyuncs.com/google_containers/kube-scheduler`。

安装tidb-operation

```shell
helm install pingcap/tidb-operator --name=tidb-operator --namespace=tidb-admin --version=<chart-version> -f /home/tidb/tidb-operator/values-tidb-operator.yaml && \
kubectl get po -n tidb-admin -l app.kubernetes.io/name=tidb-operator
```

### 5）安装tidb集群

通过下面命令获取待安装的 tidb-cluster chart 的 `values.yaml` 配置文件：

```shell
mkdir -p /home/tidb/<release-name> && \
helm inspect values pingcap/tidb-cluster --version=<chart-version> > /home/tidb/<release-name>/values-<release-name>.yaml
```

mkdir -p /home/tidb/tidb-cluster && \
helm inspect values pingcap/tidb-cluster --version=v1.0.6 > /home/tidb/tidb-cluster/values-tidb-cluster.yaml

注意：**

- `/home/tidb` 可以替换为你想用的目录。
- `release-name` 将会作为 Kubernetes 相关资源（例如 Pod，Service 等）的前缀名，可以起一个方便记忆的名字，要求全局唯一，通过 `helm ls -q` 可以查看集群中已经有的 `release-name`。
- `chart-version` 是 tidb-cluster chart 发布的版本，可以通过 `helm search -l tidb-cluster` 查看当前支持的版本。
- 下文会用 `values.yaml` 指代 `/home/tidb/values-.yaml`。

#### 集群拓扑

默认部署的集群拓扑是：3 个 PD Pod，3 个 TiKV Pod，2 个 TiDB Pod 和 1 个监控 Pod。在该部署拓扑下根据数据高可用原则，TiDB Operator 扩展调度器要求 Kubernetes 集群中至少有 3 个节点。如果 Kubernetes 集群节点个数少于 3 个，将会导致有一个 PD Pod 处于 Pending 状态，而 TiKV 和 TiDB Pod 也都不会被创建。

Kubernetes 集群节点个数少于 3 个时，为了使 TiDB 集群能启动起来，可以将默认部署的 PD 和 TiKV Pod 个数都减小到 1 个，或者将 `values.yaml` 中 `schedulerName` 改为 Kubernetes 内置调度器 `default-scheduler`。

> **警告：**
>
> `default-scheduler` 仅适用于演示环境，改为 `default-scheduler` 后，TiDB 集群的调度将无法保证数据高可用，另外一些其它特性也无法支持，例如 [TiDB Pod StableScheduling](https://github.com/pingcap/tidb-operator/blob/master/docs/design-proposals/tidb-stable-scheduling.md) 等。
>
> 这里我只有2个node节点所以将pd、tikv的副本数都改了1.

![image-20200323220808502](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200323220808502.png)

![image-20200323220852108](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20200323220852108.png)

#### 部署 TiDB 集群

TiDB Operator 部署并配置完成后，可以通过下面命令部署 TiDB 集群：

```shell
helm install pingcap/tidb-cluster --name=<release-name> --namespace=<namespace> --version=<chart-version> -f /home/tidb/<release-name>/values-<release-name>.yaml
```

helm install pingcap/tidb-cluster --name=tidb-cluster --namespace=tidb-cluster --version=v1.0.6-f /home/tidb/tidb-cluster/values-tidb-cluster.yaml



注意：**

`namespace` 是[命名空间](https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/)，你可以起一个方便记忆的名字，比如和 `release-name` 相同的名称, 这里我都叫tidb-cluster

通过下面命令可以查看 Pod 状态：

```shell
kubectl get po -n <namespace> -l app.kubernetes.io/instance=<release-name>
```

kubectl get po -n tidb-cluster -l app.kubernetes.io/instance=tidb-cluster

单个 Kubernetes 集群中可以利用 TiDB Operator 部署管理多套 TiDB 集群，重复以上命令并将 `release-name` 替换成不同名字即可。不同集群既可以在相同 `namespace` 中，也可以在不同 `namespace` 中，可根据实际需求进行选择。

#### 安装mysql

wget http://dev.mysql.com/get/mysql57-community-release-el7-8.noarch.rpm

mv mysql57-community-release-el7-8.noarch.rpm mysql

cd mysql

yum localinstall mysql57-community-release-el7-8.noarch.rpm
yum repolist enabled | grep "mysql.*-community.*"
yum install mysql-community-server
systemctl start mysqld

也可以使用远程的mysql访问参照下面的映射。

#### 映射tidb端口

- 查看tidb serveice

  ```
  kubectl get svc --all-namespaces
  ```

- 映射tidb端口

  ```
  # 仅本地访问
  kubectl port-forward svc/tidb-cluster 4000:4000 --namespace=tidb-cluster
  
  # 其他主机访问
  kubectl port-forward --address 0.0.0.0 svc/tidb-cluster 4000:4000 --namespace=tidb-cluster
  ```

- 首次登录mysql

  ```
  mysql -h 127.0.0.1 -P 4000 -u root -D test
  ```

- 修改tidb密码

  ```
  SET PASSWORD FOR 'root'@'%' = '123456'; FLUSH PRIVILEGES;
  ```

  