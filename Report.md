
## 使用UTM安装Ubuntu24.04

### 注意事项：
* 创建虚拟机的时候分配的硬盘空间可能不会被虚拟机完全利用（默认会用10Gi），可以在安装后手动扩展。
* Unbuntu安装过程中会提示输入镜像源和代理并测试连通性，实际上不是必须的。
* 如在虚拟机中无法访问宿主机的网络代理，可尝试将网络模式修改为桥接

### 如何扩展硬盘空间
#### 参考资料
 https://siytek.com/increase-utm-virtual-machine-disk-size
##### 修改UTM虚拟磁盘大小（注意改完之后还要点一下"Resize"/"调整大小"）
##### 扩展Linux分区
###### Step 1: Check if `/dev/vda3` is part of the volume group
First, let's confirm if `/dev/vda3` is fully part of the `ubuntu-vg` volume group. Run the following command to list physical volumes:

```bash
sudo pvs
```

This will show if `/dev/vda3` has been fully added to the volume group. If not, it needs to be extended.

###### Step 2: Resize the physical volume (if necessary)
If `/dev/vda3` is already a physical volume, but not all its space has been utilized, you can resize the physical volume to use the remaining space.

```bash
sudo pvresize /dev/vda3
```

###### Step 3: Extend the Volume Group
If `/dev/vda3` is not fully allocated to the volume group, you can add the unallocated space to the volume group:

```bash
sudo vgextend ubuntu-vg /dev/vda3
```

###### Step 4: Extend the Logical Volume
After ensuring that the volume group has more free space, extend your logical volume:

```bash
sudo lvextend -l +100%FREE /dev/mapper/ubuntu--vg-ubuntu--lv
```

This will allocate all the free space in the volume group to the logical volume.

###### Step 5: Resize the Filesystem
Finally, resize the filesystem to use the newly allocated space:

- **For ext4**:

```bash
sudo resize2fs /dev/mapper/ubuntu--vg-ubuntu--lv
```

- **For XFS**:

```bash
sudo xfs_growfs /dev/mapper/ubuntu--vg-ubuntu--lv
```

### Step 6: Verify the changes
Check the updated filesystem size with:

```bash
df -h
```


## 部署k8s
### 参考资料
https://medium.com/@rabbi.cse.sust.bd/kubernetes-cluster-setup-on-ubuntu-24-04-lts-server-c17be85e49d1

### 注意事项
* 现在k8s一般使用conntainerd而非dockerd来管理镜像
* 参考资料中的部署过程中需要访问国外镜像, 可通过为containerd配置代理来解决，如在/etc/systemd/system/containerd.service.d中创建一个http-proxy.conf文件, **并将k8s所使用的局域网地址添加到NOPROXY列表中（否则集群会因网络问题无法启动）**，如：
    ```
    [Service]
    Environment="HTTP_PROXY=http://192.168.1.101:1082"
    Environment="HTTPS_PROXY=http://192.168.1.101:1082"
    Environment="NO_PROXY=localhost,127.0.0.0/8,10.0.0.0/8,172.16.0.0/12,192.168.0.0/16,.svc,.cluster.local,.ewhisper.cn,<nodeCIDR>,<APIServerInternalURL>,<serviceNetworkCIDRs>,<etcdDiscoveryDomain>,<clusterNetworkCIDRs>,<platformSpecific>,<REST_OF_CUSTOM_EXCEPTIONS>,192.168.1.0/24"
    ```
    然后重启containerd服务
    ```
    sudo systemctl daemon-reload
    sudo systemctl restart containerd
    ```
## 部署MinIO + Flink + Iceberg
### 参考资料
https://gordonmurray.com/data/2023/11/09/apache-flink-and-apache-iceberg.html
### 将AWS替换为MinIO
* 修改docker-compose.yml为 https://github.com/laotao/minio_flink_and_iceberg/blob/main/docker-compose.yml
* 修改conf/flink-conf.yaml为 https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf/flink-conf.yaml
* 添加conf/core-site.xml https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf/core-site.xml
* 修改jobs/job.sql为 https://github.com/laotao/minio_flink_and_iceberg/blob/main/jobs/job.sql

### 将部署方式由Docker-compose转换为Kubernetes

* 先将jar打包到docker镜像仓库（需要将其中的your-docker-repo替换成自己的镜像仓库地址）
    ```
    docker build -t your-docker-repo/flink-custom:1.18.0 .
    docker push your-docker-repo/flink-custom:1.18.0
    ```
* 添加相关配置文件

    conf-k8s/minio.yaml https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf-k8s/minio.yaml

    conf-k8s/mariadb.yaml https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf-k8s/mariadb.yaml

    conf-k8s/flink-configmap.yaml  https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf-k8s/flink-configmap.yaml

    conf-k8s/flink-deployment.yaml https://github.com/laotao/minio_flink_and_iceberg/blob/main/conf-k8s/flink-deployment. （需要将其中的your-docker-repo替换成自己的镜像仓库地址）

* 依次部署mariadb, minio和flink（含iceberg）
    ```
    kubectl apply -f conf-k8s/mariadb.yaml
    kubectl apply -f conf-k8s/minio.yaml
    kubectl apply -f conf-k8s/flink-configmap.yaml
    kubectl apply -f conf-k8s/flink-deployment.yaml
    ```

### 验证
* MinIO
1. 登陆http://[k8s地址]:30001
2. 创建一个bucket: my-test-bucket

* Flink/Iceberg
1. 进入flink-jobmanager的pod，运行sql-client.sh
    ```
    kubectl exec -it [flink-jobmanager-pod] -- /bin/bash
    bin/sql-client.sh
    ```
2. 执行以下Flink SQL: 
https://github.com/laotao/minio_flink_and_iceberg/blob/main/jobs/job.sql
部分结果如下：





