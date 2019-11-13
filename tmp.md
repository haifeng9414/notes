#### 如何防止死锁
一次性申请所有的资源，这样就不存在持有锁时等待另一个资源
占用部分资源的线程进一步申请其他资源时，如果申请不到，可以主动释放它占有的资源
可以按序申请资源。所谓按序申请，是指资源是有线性顺序的，申请的时候可以先申请资源序号小的，再申请资源序号大的

#### Docker
Docker最大的优势，镜像，直接打包了应用运行所需要的整个操作系统，从而保证了本地环境和云端环境的高度一致
Cgroups技术是用来制造约束的主要手段，而Namespace技术则是用来修改进程视图的主要方法
通过clone()系统调用创建进程，参数中指定CLONE_NEWPID参数使得进程在一个新的PID Namespace，Linux操作系统还提供了Mount、UTS、IPC、Network 和User这些 Namespace，用来对各种不同的进程上下文进行“障眼法”操作
为了限制容器的资源使用，使用Cgroups，限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等
Cgroups以文件和目录的方式组织在操作系统的/sys/fs/cgroup 路径下，使用mount -t cgroup可以看到，该目录下包含cpuset、cpu、 memory 这样的目录，如在cpu目录下创建container目录，系统会自动生成若干文件，其中tasks文件指定作用的进程PID，再分别编辑其中的cfs_period和cfs_quota文件，可以指定进程在cfs_period的时间内，可以使用cfs_quota的运行时间，达到显示cpu使用率的目的
容器中的top命令会显示宿主机的/proc文件内容，解决方案是使用lxvfs，原理是重新挂载容器的/proc内的文件内容，读取使用率时从Cgroups中读取资源限制
使用chroot可以修改进程的根目录，所以使用chroot，可以将镜像中的根目录（包含了容器需要的操作系统的文件系统）设置为容器的根目录
该目录一般包含：
bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var
该目录也叫rootfs，rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像
所以所有容器共享宿主机内核
Docker在镜像的设计中，引入了层（layer）的概念。也就是说，用户制作镜像的每一步操作，都会生成一个层，也就是一个增量rootfs，这是用的是联合文件系统功能，即UnionFS，最主要的功能是将多个不同位置的目录联合挂载（union mount）到同一个目录下，如
现有A和B目录
$ tree
.
├── A
│  ├── a
│  └── x
└── B
  ├── b
  └── x

A和B都挂到C目录，执行
$ mkdir C
$ mount -t aufs -o dirs=./A:./B none ./C

结果

$ tree ./C
./C
├── a
├── b
└── x
如果在C中对文件进行修改，会体现在A和B目录

Docker使用的联合文件系统实现是AuFS。它最关键的目录结构在 /var/lib/docker 路径下的 diff 目录
/var/lib/docker/aufs/diff/<layer_id>

查看一个镜像
$ docker image inspect ubuntu:latest
...
     "RootFS": {
      "Type": "layers",
      "Layers": [
        "sha256:f49017d4d5ce9c0f544c...",
        "sha256:8f2b771487e9d6354080...",
        "sha256:ccd4d61916aaa2159429...",
        "sha256:c01d74f99de40e097c73...",
        "sha256:268a067217b5fe78e000..."
      ]
    }

这个 Ubuntu 镜像，实际上由五个层组成。这五个层就是五个增量 rootfs，每一层都是 Ubuntu 操作系统文件与目录的一部分；而在使用镜像时，Docker 会把这些增量联合挂载在一个统一的挂载点上，这个挂载点就是 /var/lib/docker/aufs/mnt/，如
/var/lib/docker/aufs/mnt/6e3be5d2ecccae7cc0fcfa2a2f5c89dc21ee30e166be823ceaeba15dce645b3e

$ ls /var/lib/docker/aufs/mnt/6e3be5d2ecccae7cc0fcfa2a2f5c89dc21ee30e166be823ceaeba15dce645b3e
bin boot dev etc home lib lib64 media mnt opt proc root run sbin srv sys tmp usr var

前面提到的五个镜像层，又是如何被联合挂载成这样一个完整的 Ubuntu 文件系统的呢？这个信息记录在 AuFS 的系统目录 /sys/fs/aufs 下面。首先，通过查看 AuFS 的挂载信息，我们可以找到这个目录对应的 AuFS 的内部 ID（也叫：si）：

$ cat /proc/mounts| grep aufs
none /var/lib/docker/aufs/mnt/6e3be5d2ecccae7cc0fc... aufs rw,relatime,si=972c6d361e6b32ba,dio,dirperm1 0 0

然后使用这个 ID，你就可以在 /sys/fs/aufs 下查看被联合挂载在一起的各个层的信息：
$ cat /sys/fs/aufs/si_972c6d361e6b32ba/br[0-9]*
/var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...=rw -> 可读写层
/var/lib/docker/aufs/diff/6e3be5d2ecccae7cc...-init=ro+wh -> init层
/var/lib/docker/aufs/diff/32e8e20064858c0f2...=ro+wh -> 下面都是只读层
/var/lib/docker/aufs/diff/2b8858809bce62e62...=ro+wh
/var/lib/docker/aufs/diff/20707dce8efc0d267...=ro+wh
/var/lib/docker/aufs/diff/72b0744e06247c7d0...=ro+wh
/var/lib/docker/aufs/diff/a524a729adadedb90...=ro+wh
从这些信息里，我们可以看到，镜像的层都放置在 /var/lib/docker/aufs/diff 目录下，然后被联合挂载在 /var/lib/docker/aufs/mnt 里面

如果我现在要做的，是删除只读层里的一个文件呢？为了实现这样的删除操作，AuFS 会在可读写层创建一个 whiteout 文件，把只读层里的文件“遮挡”起来。比如，你要删除只读层里一个名叫 foo 的文件，那么这个删除操作实际上是在可读写层创建了一个名叫.wh.foo 的文件。这样，当这两个层被联合挂载之后，foo 文件就会被.wh.foo 文件“遮挡”起来，“消失”了。这个功能，就是“ro+wh”的挂载方式，即只读 +whiteout 的含义。我喜欢把 whiteout 形象地翻译为：“白障”
所以，最上面这个可读写层的作用，就是专门用来存放你修改 rootfs 后产生的增量，无论是增、删、改，都发生在这里。而当我们使用完了这个被修改过的容器之后，还可以使用 docker commit 和 push 指令，保存这个被修改过的可读写层，并上传到 Docker Hub 上，供其他人使用；而与此同时，原先的只读层里的内容则不会有任何变化。这，就是增量 rootfs 的好处

Init 层是 Docker 项目单独生成的一个内部层，专门用来存放 /etc/hosts、/etc/resolv.conf 等信息，用户执行 docker commit 只会提交可读写层，所以是不包含这些内容的。

在使用 Dockerfile 时，你可能还会看到一个叫作 ENTRYPOINT 的原语。实际上，它和 CMD 都是 Docker 容器进程启动所必需的参数，完整执行格式是：“ENTRYPOINT CMD”。但是，默认情况下，Docker 会为你提供一个隐含的 ENTRYPOINT，即：/bin/sh -c。所以，在不指定 ENTRYPOINT 时，比如CMD ["python", "app.py"]，实际上运行在容器里的完整进程是：/bin/sh -c “python app.py”，即 CMD 的内容就是 ENTRYPOINT 的参数

查看容器真正的PID
$docker inspect --format '{{ .State.Pid }}' 4ddf4638572d
25686

这时，你可以通过查看宿主机的 proc 文件，看到这个 25686 进程的所有 Namespace 对应的文件

$ ls -l  /proc/25686/ns
total 0
lrwxrwxrwx 1 root root 0 Aug 13 14:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 ipc -> ipc:[4026532278]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 mnt -> mnt:[4026532276]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 net -> net:[4026532281]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 pid_for_children -> pid:[4026532279]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Aug 13 14:05 uts -> uts:[4026532277]

有了这样一个可以“hold 住”所有 Linux Namespace 的文件，我们就可以对 Namespace 做一些很有意义事情了，比如：加入到一个已经存在的 Namespace 当中。
这也就意味着：一个进程，可以选择加入到某个进程已有的 Namespace 当中，从而达到“进入”这个进程所在容器的目的，这正是 docker exec 的实现原理。
实现方式是使用setns()系统调用，运行指定进程如/bin/bash，并加入到指定Namespace中

Docker 还专门提供了一个参数，可以让你启动一个容器并“加入”到另一个容器的 Network Namespace 里，这个参数就是 -net，比如:
$ docker run -it --net container:4ddf4638572d busybox ifconfig
这样，我们新启动的这个容器，就会直接加入到 ID=4ddf4638572d 的容器
而如果我指定–net=host，就意味着这个容器不会为进程启用 Network Namespace。这就意味着，这个容器拆除了 Network Namespace 的“隔离墙”，所以，它会和宿主机上的其他普通进程一样，直接共享宿主机的网络栈。这就为容器直接操作和使用宿主机网络提供了一个渠道。

容器技术使用了 rootfs 机制和 Mount Namespace，构建出了一个同宿主机完全隔离开的文件系统环境。这时候，我们就需要考虑这样两个问题：
容器里进程新建的文件，怎么才能让宿主机获取到？
宿主机上的文件和目录，怎么才能让容器里的进程访问到？
这正是 Docker Volume 要解决的问题：Volume 机制，允许你将宿主机上指定的目录或者文件，挂载到容器里面进行读取和修改操作。
如
$ docker run -v /test ...
$ docker run -v /home:/test ...
第一种情况下，由于你并没有显示声明宿主机目录，那么 Docker 就会默认在宿主机上创建一个临时目录 /var/lib/docker/volumes/[VOLUME_ID]/_data，然后把它挂载到容器的 /test 目录上。而在第二种情况下，Docker 就直接把宿主机的 /home 目录挂载到容器的 /test 目录上。

实现原理是，当容器进程被创建之后，尽管开启了 Mount Namespace，但是在它执行 chroot 之前，容器进程一直可以看到宿主机上的整个文件系统。而宿主机上的文件系统，也自然包括了我们要使用的容器镜像。这个镜像的各个层，保存在 /var/lib/docker/aufs/diff 目录下，在容器进程启动后，它们会被联合挂载在 /var/lib/docker/aufs/mnt/ 目录中，这样容器所需的 rootfs 就准备好了。所以，我们只需要在 rootfs 准备好之后，在执行 chroot 之前，把 Volume 指定的宿主机目录（比如 /home 目录），挂载到指定的容器目录（比如 /test 目录）在宿主机上对应的目录（即 /var/lib/docker/aufs/mnt/[可读写层 ID]/test）上，这个 Volume 的挂载工作就完成了。更重要的是，由于执行这个挂载操作时，“容器进程”已经创建了，也就意味着此时 Mount Namespace 已经开启了。所以，这个挂载事件只在这个容器里可见。你在宿主机上，是看不见容器内部的这个挂载点的。这就保证了容器的隔离性不会被 Volume 打破。
注意：这里提到的 " 容器进程 "，是 Docker 创建的一个容器初始化进程 (dockerinit)，而不是应用进程 (ENTRYPOINT + CMD)。dockerinit 会负责完成根目录的准备、挂载设备和目录、配置 hostname 等一系列需要在容器内进行的初始化操作。最后，它通过 execv() 系统调用，让应用进程取代自己，成为容器里的 PID=1 的进程。

那么，这个 /test 目录里的内容，既然挂载在容器 rootfs 的可读写层，它会不会被 docker commit 提交掉呢？也不会。这个原因其实我们前面已经提到过。容器的镜像操作，比如 docker commit，都是发生在宿主机空间的。而由于 Mount Namespace 的隔离作用，宿主机并不知道这个绑定挂载的存在。所以，在宿主机看来，容器中可读写层的 /test 目录（/var/lib/docker/aufs/mnt/[可读写层 ID]/test），始终是空的。

其实，如果你了解 Linux 内核的话，就会明白，绑定挂载实际上是一个 inode 替换的过程。在 Linux 操作系统中，inode 可以理解为存放文件内容的“对象”，而 dentry，也叫目录项，就是访问这个 inode 所使用的“指针”。挂载相当于设置指针的指向

关于inode
文件储存在硬盘上，硬盘的最小存储单位叫做"扇区"（Sector）。每个扇区储存512字节（相当于0.5KB）。
操作系统读取硬盘的时候，不会一个个扇区地读取，这样效率太低，而是一次性连续读取多个扇区，即一次性读取一个"块"（block）。这种由多个扇区组成的"块"，是文件存取的最小单位。"块"的大小，最常见的是4KB，即连续八个 sector组成一个 block。
文件数据都储存在"块"中，那么很显然，我们还必须找到一个地方储存文件的元信息，比如文件的创建者、文件的创建日期、文件的大小等等。这种储存文件元信息的区域就叫做inode，中文译名为"索引节点"。每一个文件都有对应的inode，里面包含了与该文件有关的一些信息，如文件所有者，文件读写执行权限，文件字节数，上次打开时间，上次修改时间，block位置等，用stat命令可以查看文件inode信息
stat example.txt
总之，除了文件名以外的所有文件信息，都存在inode之中
inode也会消耗硬盘空间，所以硬盘格式化的时候，操作系统自动将硬盘分成两个区域。一个是数据区，存放文件数据；另一个是inode区（inode table），存放inode所包含的信息。
每个inode节点的大小，一般是128字节或256字节。inode节点的总数，在格式化时就给定，一般是每1KB或每2KB就设置一个inode。假定在一块1GB的硬盘中，每个inode节点的大小为128字节，每1KB就设置一个inode，那么inode table的大小就会达到128MB，占整块硬盘的12.8%。
查看每个硬盘分区的inode总数和已经使用的数量，可以使用df -i命令。
每个inode都有一个号码，操作系统用inode号码来识别不同的文件。
这里值得重复一遍，Unix/Linux系统内部不使用文件名，而使用inode号码来识别文件。对于系统来说，文件名只是inode号码便于识别的别称或者绰号。
表面上，用户通过文件名，打开文件。实际上，系统内部这个过程分成三步：首先，系统找到这个文件名对应的inode号码；其次，通过inode号码，获取inode信息；最后，根据inode信息，找到文件数据所在的block，读出数据。
使用ls -i命令，可以看到文件名对应的inode号码。
一般情况下，文件名和inode号码是"一一对应"关系，每个inode号码对应一个文件名。但是，Unix/Linux系统允许，多个文件名指向同一个inode号码。
这意味着，可以用不同的文件名访问同样的内容；对文件内容进行修改，会影响到所有文件名；但是，删除一个文件名，不影响另一个文件名的访问。这种情况就被称为"硬链接"（hard link）。
ln命令创建硬链接
删除一个文件名，就会使得inode节点中的"链接数"减1。当这个值减到0，表明没有文件名指向这个inode，系统就会回收这个inode号码，以及其所对应block区域。
除了硬链接以外，还有一种特殊情况。
文件A和文件B的inode号码虽然不一样，但是文件A的内容是文件B的路径。读取文件A时，系统会自动将访问者导向文件B。因此，无论打开哪一个文件，最终读取的都是文件B。这时，文件A就称为文件B的"软链接"（soft link）或者"符号链接（symbolic link）。
这意味着，文件A依赖于文件B而存在，如果删除了文件B，打开文件A就会报错："No such file or directory"。这是软链接与硬链接最大的不同：文件A指向文件B的文件名，而不是文件B的inode号码，文件B的inode"链接数"不会因此发生变化。
ln -s命令可以创建软链接

#### k8s
Kubernetes 项目最主要的设计思想是，从更宏观的角度，以统一的方式来定义任务（容器）之间的各种关系，并且为将来支持更多种类的关系留有余地。
k8s由控制节点和计算节点组成
控制节点，即 Master 节点，由三个紧密协作的独立组件组合而成，它们分别是负责 API 服务的 kube-apiserver、负责调度的 kube-scheduler，以及负责容器编排的 kube-controller-manager。整个集群的持久化数据，则由 kube-apiserver 处理后保存在 Etcd 中。
而计算节点上最核心的部分，则是一个叫作 kubelet 的组件。
在 Kubernetes 项目中，kubelet 主要负责同容器运行时（比如 Docker 项目）打交道。而这个交互所依赖的，是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。
而 kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储。这两个插件与 kubelet 进行交互的接口，分别是 CNI（Container Networking Interface）和 CSI（Container Storage Interface）。
POD存在的意义在于，部署的应用，往往都存在着类似于“进程和进程组”的关系，POD是多个进程关系紧密，同时方便调度，共享通过一组资源，如Volume。
在 Kubernetes 项目里，Pod 的实现需要使用一个中间容器，这个容器叫作 Infra 容器。在这个 Pod 中，Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。
Pod 中，所有 Init Container 定义的容器，都会比 spec.containers 定义的用户容器先启动。并且，Init Container 容器会按顺序逐一启动，而直到它们都启动并且退出了，用户容器才会启动。
Secret
Volume
ServiceAccount
livenessProbe
restartPolicy
Deployment -> ReplicaSet
StatefulSet：拓扑状态，存储状态
Headless Service，会绑定地址
<pod-name>.<svc-name>.<namespace>.svc.cluster.local
pvc
StorageClass
Ingress：根据host、path等配置，转发请求到svc
DaemonSet：这个 Pod 运行在 Kubernetes 集群里的每一个节点（Node）上；每个节点上只有一个这样的 Pod 实例；当有新的节点加入 Kubernetes 集群后，该 Pod 会自动地在新节点上被创建出来；而当旧节点被删除后，它上面的 Pod 也相应地会被回收掉。
istio：实现灰度发布，即导流，每个容器上有个Envoy容器，管理pod的流量
Dynamic Admission Control
所谓“声明式”，指的就是我只需要提交一个定义好的 API 对象来“声明”，我所期望的状态是什么样子。其次，“声明式 API”允许有多个 API 写端，以 PATCH 的方式对 API 对象进行修改，而无需关心本地原始 YAML 文件的内容
声明式 API，才是 Kubernetes 项目编排能力“赖以生存”的核心所在
一个 API 对象在 Etcd 里的完整资源路径，是由：Group（API 组）、Version（API 版本）和 Resource（API 资源类型）三个部分组成的。
容器通过网桥互通，使用Veth pair，Veth Pair 设备的特点是：它被创建出来后，总是以两张虚拟网卡（Veth Peer）的形式成对出现的。并且，从其中一个“网卡”发出的数据包，可以直接出现在与它对应的另一张“网卡”上，哪怕这两个“网卡”在不同的 Network Namespace 里。
创建容器时会创建Veth pair，一端连接容器网卡eth0，一端连接网桥，一般就是docker0，相当于交换机，通过 route 命令查看 nginx-1 容器的路由表，我们可以看到，这个 eth0 网卡是这个容器里的默认路由设备；所有对 172.17.0.0/16 网段的请求，也会被交给 eth0 来处理（第二条 172.17.0.0 路由规则）。而另一端查看宿主机的网络设备可以看到它，如下所示
# 在宿主机上
$ ifconfig
...
docker0   Link encap:Ethernet  HWaddr 02:42:d8:e4:df:c1  
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:d8ff:fee4:dfc1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:309 errors:0 dropped:0 overruns:0 frame:0
          TX packets:372 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:0 
          RX bytes:18944 (18.9 KB)  TX bytes:8137789 (8.1 MB)
veth9c02e56 Link encap:Ethernet  HWaddr 52:81:0b:24:3d:da  
          inet6 addr: fe80::5081:bff:fe24:3dda/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:288 errors:0 dropped:0 overruns:0 frame:0
          TX packets:371 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:0 
          RX bytes:21608 (21.6 KB)  TX bytes:8137719 (8.1 MB)
          
$ brctl show
bridge name bridge id  STP enabled interfaces
docker0  8000.0242d8e4dfc1 no  veth9c02e56
在 Docker 的默认配置下，一台宿主机上的 docker0 网桥，和其他宿主机上的 docker0 网桥，没有任何关联，它们互相之间也没办法连通。所以，连接在这些网桥上的容器，自然也没办法进行通信了。可以通过Overlay Network（覆盖网络）技术解决这个问题，构建这种容器网络的核心在于：我们需要在已有的宿主机网络上，再通过软件构建一个覆盖在已有宿主机网络之上的、可以把所有容器连通在一起的虚拟网络。
而这个 Overlay Network 本身，可以由每台宿主机上的一个“特殊网桥”共同组成。比如，当 Node 1 上的 Container 1 要访问 Node 2 上的 Container 3 的时候，Node 1 上的“特殊网桥”在收到数据包之后，能够通过某种方式，把数据包发送到正确的宿主机，比如 Node 2 上。而 Node 2 上的“特殊网桥”在收到数据包后，也能够通过某种方式，把数据包转发给正确的容器，比如 Container 3。
要理解容器“跨主通信”的原理，就一定要先从 Flannel 这个项目说起。Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。目前，Flannel 支持三种后端实现，分别是：
VXLAN；
host-gw；
UDP。

#### 流程
1. 创建订单到数据库，为订单创建流程
2. 根据Model或者说是Deploy获取流程定义
3. 新建一个FlowProcess对象并保存到数据库，表示一个流程实体，如虚拟机申请流程，该对象还存有订单id，即BusinessKey
4. 从Deploy中获取流程的JSON，该JSON中包含了流程的所有步骤和每个步骤的执行人等信息，记为activities
5. 为流程中第一个步骤提交申请创建FlowTask对象并保存到数据库
6. 提交申请步骤默认直接成功，直接执行下一个步骤
7. 下一步动作时所有流程环节的通用逻辑，首先判断执行完成的流程环节是否是流程的最后一个环节
   - 是则设置流程状态为完成，即上面创建的FlowProcess，之后发送流程完成事件，发送时传入参数为ProcessConstants.EventOperation.COMPLETE，流程事件是流程的属性，保存在flow_event表，用model_id和流程关联，并且还有一列operation，表示指定流程时间发生时触发，flow_event表中存了一列exector和arguments，分别表示处理流程事件的执行类和流程事件执行时使用的参数，用反射创建执行类处理流程完成事件，对于ProcessConstants.EventOperation.COMPLETE事件，默认是现实VmOrderEvent，并且argument为APPROVED，VmOrderEvent将订单的状态设置为argument，即订单已审批，同时后台有一个定时任务检查所有APPROVED状态的订单并执行，流程完成后就不会再影响订单了，流程也不会再有其他更新
   - 否则从activities中获取下一个流程环节并获取流程的执行人，不为空则为每个执行人创建一个FlowTask，这么做是为了每个执行人能从自己的待办中获取自己需要处理的流程，之后发起流程Pending事件，默认没有处理Pending事件的流程事件
8. 当某个执行人同意了审批，更新所有这一环节的FlowTask为完成，发送FlowTask完成事件，和流程事件一样完成事件的处理类保存在flow_event表中，但是默认没有处理FlowTask完成事件的类
9. 进行下一个流程环节，即回到第7步
10. 如果某人拒绝了审批，更新流程为TERMINATED状态，更新所有这一环节的FlowTask为TERMINATED，这样审批人在其待办中就看不到这个审批，发送流程TERMINATED事件，默认还是VmOrderEvent，flow_event表中有一条operation为TERMINATED，argument为Rejected的VmOrderEvent，在流程TERMINATED时更新订单状态为Rejected，流程结束
11. 上面的各个环节都有发送消息的步骤，略

#### client-go
首先有个基础队列，用lock保证了线程安全，该队列有几个重要属性，数组queue、集合dirty和集合processing，添加新元素是先判断元素是否在dirty中，在则直接返回，繁殖同一个对象重复入队，否则将元素加到dirty，判断元素是否在processing中，在则直接返回，否则入队到queue
获取元素是，如果queue为空，wait，否则取出队列的第一个元素，添加到processing中，删除dirty中该元素，以便能再次添加元素到dirty
当元素处理完成后需要调用done方法，该方法删除processing中的该元素，如果dirty中存在该元素，添加到queue中，这就实现了同一个对象只能从队列中被获取一次，再次入队会被缓存到dirty中

之后是延时队列，继承自基础队列，新增AddAfter方法，在指定时间后才入队元素，延时队列有一个clock获取系统事件，一个ticker定时器，延时队列还有个chan，元素是waitFor类型的，所有入队的元素先被封装为waitFor，waitFor持有需要入队的元素和入队时间
还定义了waitForPriorityQueue数组，用于保存waitFor类型的元素，waitForPriorityQueue数组实现了go/src/container/heap/heap.go的Interface接口，所以是个优先级队列，按照入队时间排序
延时队列加入新元素时先判断入队时间是否小于0，小于0直接入基础队列，否则根据入队时传入的时间参数将元素封装为waitFor添加到waitingForAddCh这个chan中，而延时队列在初始化后会启动一个协程调用waitingLoop方法，该方法从waitingForAddCh获取元素
waitingLoop方法创建waitForPriorityQueue数组用于保存所有等待入队的waitFor，按时间排序，然后开启无限循环，根据队头元素的时间创建一个定时器（如果没有元素就是一个never），并select在该定时器和waitingForAddCh上，等待动作，如果定时器先发生，说明有元素能够入队了，结束select并重写循环，而循环开始会判断队列头元素时间是否到了，到了就入基础队列，从而实现定时入队，而如果select在waitingForAddCh上的等待返回了，则判断元素时间是否能够入队，能则直接入队，否则添加到waitForPriorityQueue数组中（调用heap.push方法实现按时间排序），同时有个map会记录waitForPriorityQueue数组中的元素，已存在的入数组只会更新时间

本地缓存，显示Store接口的定义，定义了本读缓存的基本操作：
```
type Store interface {
    Add(obj interface{}) error
    Update(obj interface{}) error
    Delete(obj interface{}) error
    List() []interface{}
    ListKeys() []string // 列举对象键
    Get(obj interface{}) (item interface{}, exists bool, err error) // 返回指定对象对应对象键对应的对象
    GetByKey(key string) (item interface{}, exists bool, err error) // 返回对象键指定的对象
    Replace([]interface{}, string) error // 使用指定的元素替换存储中的所有元素，等同于删除全部原有对象再逐一添加新的对象
    Resync() error // 重新同步存储对象
}
```
本地缓存能够保存一个map，key为对象的唯一表示，value为对象，同时维护一些索引函数，本地缓存中的元素被添加时会被所有索引函数计算一次，并缓存计算结果

之后是DeltaFIFO，Delta持有一个对象和对应的操作，如增删改，DeltaFIFO的replace方法接收一个list，list中是所有同步到的对象，DeltaFIFO根据KeyFunc为每个对象计算唯一标识保存到自己的map中，map的value是每个对象的Delta数组，初始情况下每个对象的Delta数组中都是一个sync事件，replace方法在将所有对象保存到自己的map后，根据knownObjects判断对象是否被删除了，如果被删除则添加删除的Delta到指定对象的Delta数组中

之后是Reflector，Reflector会列举出所有现有的符合expectType对象，并入队这些对象的Sync Delta，同时会监控execptType对象的变化，将Add、Update、Delete等Delta入队，如果设置了resyncPeriod，则会定时调用store的Resync方法，store实际上就是DeltaFIFO

Controller就是利用Reflector获取全量对象并监控对象的变化，并且将所有对象对应的Delta从Queue中取出交由ProcessFunc处理