
当集群中需要升级 Mount Pod 时，目前推荐的方式是更新配置后重新挂载应用 Pod 进行滚动升级，但这种升级方式的问题在于需要业务重启。


如果对业务的使用模式很清楚时，比如没有数据写入等，也可以选择手动重建 Mount Pod 的方式。在更新配置后，手动删除已有的 Mount Pod，并等待其重建，同时依赖 CSI 对挂载点的自动恢复功能，等待应用 Pod 中挂载点的恢复。这种升级的过程有几个问题：


1. **操作繁琐**：需要用户自己使用 kubectl 命令，通过应用 Pod 找到对应的 Mount Pod；
2. **升级时业务中断**：CSI 需要先等待旧的 Mount Pod 完全被删除，才会去创建新的 Mount Pod。这就导致了在旧的 Mount Pod 删除后、新的 Mount Pod 起来前，业务处于不可用状态；
3. **挂载点自动恢复对原 I/O 操作的影响**：由于 CSI 使用了重新 bind 应用挂载点的方式实现了自动恢复，所以即使在恢复后，原先的 I/O 操作都会报错。


为了解决现有的升级过程遇到的问题，JuiceFS CSI Driver 在 v0\.25\.0 版本中，实现了 Mount Pod 的平滑升级，即在应用不停服的情况下升级 Mount Pod。


相比有损升级的方式，使用平滑升级的好处在于：


1. 操作简单：在 CSI Dashboard 中，可以轻易的在应用 Pod 的页面找到 Mount Pod，且点击按钮即可触发平滑升级；
2. 业务不中断：升级过程中，业务不会受到影响，用户也可以使用平滑升级功能来进行 Mount Pod 的参数和配置修改。


## 01 CSI 如何实现 Mount Pod 的平滑升级


目前 JuiceFS CSI 支持两种平滑升级方式，即二进制升级和 Pod 重建升级。


### 二进制升级


二进制升级不会重建 Mount Pod，而是升级 Mount Pod 中的客户端二进制。其依赖 JuiceFS 客户端自身的守护进程，即社区版版本在 v1\.2\.0 以上，商业版版本在 v5\.0\.0 以上。


整个二进制升级的过程如下：


![](https://img2024.cnblogs.com/blog/2544292/202411/2544292-20241101105525734-387835725.png)


1. 在触发平滑升级后，CSI Node 启动一个使用新镜像的 Job，在 Job 中将 JuiceFS 客户端二进制 copy 到 Mount Pod 中；
2. 等 Job 完成后，CSI Node 向 Mount Pod 发送 SIGHUP 信号；
3. Mount Pod 接收到 SIGHUP 信号后，将目前的 I/O 请求状态信息保存到临时文件中，此时服务进程退出；
4. Mount Pod 的服务进程退出后，守护进程会用新的二进制再次启动新的服务进程；
5. 新的服务进程启动后读取中间状态文件，并继续处理之前的请求；
6. 新的服务进程向守护进程拿当前的 FUSE fd。


二进制升级使用于仅需要升级客户端的情况。但升级后查看 Pod 的 yaml，其镜像依然是旧的。由于没有重建 Pod，这种升级的好处在于速度快且风险小，缺点在于不能更新 Mount Pod 的其他配置。


### Pod 重建升级


Pod 重建升级指的是重建 Mount Pod 进行平滑升级。这种升级方式依赖 JuiceFS 本身的平滑升级功能，即社区版版本在 v1\.2\.1 以上，商业版版本在 v5\.1\.0 以上。


整个 Pod 重建升级的过程如下：


![](https://img2024.cnblogs.com/blog/2544292/202411/2544292-20241101105533721-167345320.png)


1. 每个 Mount Pod 在启动后，都会将自身使用的 FUSE fd 传给 CSI Node，CSI Node 维护了每个 Mount Pod 使用的 FUSE fd 的关系表；
2. 在触发了平滑升级后，CSI Node 会先起一个与新 Mount Pod 相同配置的空 Job，这一步的作用是提前将新镜像 pull 在节点上，从而节省升级时间；
3. 等 Job 完成后，CSI Node 向旧的 Mount Pod 发送 SIGHUP 信号；
4. 旧的 Mount Pod 接收到 SIGHUP 信号后，将目前的 I/O 状态信息保存到临时文件中，并退出。此时 Mount Pod 变为 Complete 状态；
5. CSI Node 根据 ConfigMap 中的配置，创建新的 Mount Pod；
6. 新的 Mount Pod 起来后，读取临时文件中的中间状态信息，从而继续处理之前的请求；
7. 新的 Mount Pod 向 CSI Node 拿当前的 FUSE fd。


其中 Mount Pod 和 CSI Node 之间通过 [Unix domain socket](https://github.com) 来传递文件句柄。当某个 FUSE 请求未能在升级期间完成，会被强制中断，建议在负载比较低的时候进行升级操作。


可以看到整个平滑升级的过程，与[宿主机上客户端的平滑升级](https://github.com):[蓝猫机场加速器](https://dahelaoshi.com)类似，唯一的区别在于由 CSI Node 向旧的服务进程发送 SIGHUP 信号以及新的服务进程启动后向 CSI Node 拿 FUSE fd。这是因为 Mount Pod 在重建后，其中的守护进程无法向旧的守护进程发送 SIGHUP 信号以及无法通过 Unix domain socket 传递文件句柄，所以在 K8s 环境中这些工作就交由 CSI Node 来完成。


这种升级方式由于重建了 Pod，缺点在于如果集群环境比较复杂，重建 Pod 的过程中出错的风险比较大；其优点在于可以更新 Mount Pod 的其他配置，且查看 Pod 的 yaml，其镜像是新的。


## 02 如何触发平滑升级


目前平滑升级可以在 CSI Dashboard 或者 kubectl 插件中触发。


### Dashboard 中触发


首先在 CSI Dashboard 中，点击「配置」按钮，更新 Mount Pod 需要升级的新镜像版本。


![](https://img2024.cnblogs.com/blog/2544292/202411/2544292-20241101105544614-1604175746.png)


其次，在 Mount Pod 的详情页，有两个升级按钮，分别是「Pod 重建升级」和「二进制升级」。


* Pod 重建升级：Mount Pod 会重建，可用于更新镜像、调整挂载参数、调整 pod 资源等。Mount Pod 的最低版本要求为 1\.2\.1（社区版）或 5\.1\.0（企业版）；
* 二进制升级：Mount Pod 不重建，只升级其中的二进制，不可变更其他配置，且升级完成后 Pod yaml 中所看到的依然是原来的镜像。Mount Pod 的最低版本要求为 1\.2\.0（社区版）或 5\.0\.0（企业版）。


![](https://img2024.cnblogs.com/blog/2544292/202411/2544292-20241101105553406-1330353784.png)


点击升级按钮，即可触发 Mount Pod 的平滑升级。


![](https://img2024.cnblogs.com/blog/2544292/202411/2544292-20241101105605058-702995954.png)


触发后可以看到整个过程，完成后页面会自动跳转到新的 Mount Pod 的详情页。


### kubectl 插件中触发


使用 kubectl 在 CSI ConfigMap 配置中更新 Mount Pod 所需要升级的镜像版本。



```
apiVersion: v1
data:
   config.yaml: |
      mountPodPatch:
         - ceMountImage: juicedata/mount:ce-v1.2.0
           eeMountImage: juicedata/mount:ee-5.1.1-ca439c2
kind: ConfigMap

```

使用 JuiceFS kubectl plugin 触发 Mount Pod 的平滑升级。



```
# Pod 重建升级
kubectl jfs upgrade juicefs-kube-node-1-pvc-52382ebb-f22a-4b7d-a2c6-1aa5ac3b26af-ebngyg --recreate
# 二进制升级
kubectl jfs upgrade juicefs-kube-node-1-pvc-52382ebb-f22a-4b7d-a2c6-1aa5ac3b26af-ebngyg

```

## 03 总结


鉴于目前的有损升级方案存在诸多缺陷，JuiceFS CSI Driver 在 v0\.25\.0 版本中，支持了 Mount Pod 的平滑升级。CSI 提供了两种平滑升级方案，包括二进制升级和 Pod 重建升级。二进制升级风险小，但不支持更新 Mount Pod 的其他配置；Pod 重建升级在集群环境较复杂的情况下有失败的风险，但支持更新 Mount Pod 的其他配置，比如可以根据 Mount Pod 的实际资源使用情况，动态调整资源配比等。用户可以根据需要，选择更适合的升级方式，同时建议在负载比较低的时候进行升级操作。


