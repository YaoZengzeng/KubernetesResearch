## 深入理解CNI

### 1 为什么会有CNI?

CNI是Container Network Interface的缩写，简单地说，就是一个标准的，通用的接口。已知我们现在有各种各样的容器平台：docker，kubernetes，mesos，我们也有各种各样的容器网络解决方案：flannel，calico，weave，并且还有各种新的解决方案在不断涌现。如果每出现一个新的解决方案，我们都需要对两者进行适配，那么由此带来的工作量必然是巨大的，而且也是重复和不必要的。事实上，我们只要提供一个标准的接口，更准确的说是一种协议，就能完美地解决上述问题。一旦有新的网络方案出现，只要它能满足这个标准的协议，那么它就能为同样满足该协议的所有容器平台提供网络功能，而CNI正是这样的一个标准接口协议。



### 2 什么是CNI?

通俗地讲，CNI是一个接口协议，用于连接容器管理系统和网络插件。前者提供一个容器所在的network namespace（从网络的角度来看，network namespace和容器是完全等价的），后者负责将network interface插入该network namespace中（比如veth的一端），并且在宿主机做一些必要的配置（例如将veth的另一端加入bridge中），最后对namespace中的interface进行IP和路由的配置。那么CNI的工作其实主要是从容器管理系统处获取运行时信息，包括network namespace的路径，容器ID以及network interface name，再从容器网络的配置文件中加载网络配置信息，再将这些信息传递给对应的插件，由插件进行具体的网络配置工作，并将配置的结果再返回到容器管理系统中。

最后，需要注意的是，在之前的CNI版本中，网络配置文件只能描述一个network，这也就表明了一个容器只能加入一个容器网络。但是在后来的CNI版本中，我们可以在配置文件中定义一个所谓的NetworkList，事实上就是定义一个network序列，CNI会依次调用各个network的插件对容器进行相应的配置，从而允许一个容器能够加入多个容器网络。



### 3 怎么使用CNI?

在进一步探索CNI之前，我觉得首先来看看CNI是怎么使用的，先对CNI有一个直观的认识，是很有必要的，这对我们之后的理解也将非常有帮助。现在，我们将依次执行如下步骤来演示如何使用CNI，并对每一步的操作进行必要的说明。

(1) 编译安装CNI的官方插件

现在官方提供了三种类型的插件：main，meta和ipam。其中main类型的插件主要提供某种网络功能，比如我们在示例中将使用的brdige，以及loopback，ipvlan，macvlan等等。meta类型的插件不能作为独立的插件使用，它通常需要调用其他插件，例如flannel，或者配合其他插件使用，例如portmap。最后ipam类型的插件其实是对所有CNI插件共有的IP管理部分的抽象，从而减少插件编写过程中的重复工作，官方提供的有dhcp和host-local两种类型。

接着执行如下命令，完成插件的下载，编译，安装工作：

```go
$ mkdir -p $GOPATH/src/github.com/containernetworking/plugins
$ git clone https://github.com/containernetworking/plugins.git  $GOPATH/src/github.com/containernetworking/plugins
$ cd $GOPATH/src/github.com/containernetworking/plugins
$ ./build.sh

```

最终所有的插件都将以可执行文件的形式存在在目录$GOPATH/src/github.com/containernetworking/plugins/bin之下。

(2) 创建配置文件，对所创建的网络进行描述

工作目录"/etc/cni/net.d"是CNI默认的网络配置文件目录，当没有特别指定时，CNI就会默认对该目录进行查找，从中加载配置文件进行容器网络的创建。至于对配置文件各个字段的详细描述，我将在后续章节进行说明。

现在我们只需要执行如下命令，描述一个我们想要创建的容器网络"mynet"即可。为了简单起见，我们的NetworkList中仅仅只有"mynet"这一个network。

```go
$ mkdir -p /etc/cni/net.d
$ cat >/etc/cni/net.d/10-mynet.conflist <<EOF
{
        "cniVersion": "0.3.0",
        "name": "mynet",
        "plugins": [
          {
                "type": "bridge",
                "bridge": "cni0",
                "isGateway": true,
                "ipMasq": true,
                "ipam": {
                        "type": "host-local",
                        "subnet": "10.22.0.0/16",
                        "routes": [
                                { "dst": "0.0.0.0/0" }
                        ]
                }
          }
        ]
}
EOF
$ cat >/etc/cni/net.d/99-loopback.conf <<EOF
{
    "cniVersion": "0.3.0",
    "type": "loopback"
}
EOF
```

(3) 模拟CNI的执行过程，创建network namespace，加入上文中描述的容器网络"mynet"

首先我们从github上下载、编译CNI的源码，最终将在bin目录下生成一个名为"cnitool"的可执行文件。事实上，可以认为cnitool是一个模拟程序，我们先创建一个名为ns的network namespace，用来模拟一个新创建的容器，再调用cnitool对该network namespace进行网络配置，从而模拟一个新建的容器加入一个容器网络的过程。

从cnitool的执行结果来看，它会返回一个包含了interface，IP，路由等等各种信息的json串，事实上它正是CNI对容器进行网络配置后生成的结果信息，对此我们将在后续章节进行详细的描述。

最终，我们可以看到network namespace内新建的网卡eth0的IP地址为10.22.0.5/16，正好包含在容器网络"mynet"的子网范围10.22.0.0/16之内，因此我们可以认为容器已经成功加入了容器网络之中，演示成功。

```go
$ git clone https://github.com/containernetworking/cni.git $GOPATH/src/github.com/containernetworking/cni

$ cd $GOPATH/src/github.com/containernetworking/cni
 
$ ./build.sh
 
$ cd $GOPATH/src/github.com/containernetworking/cni/bin
 
$ export CNI_PATH=$GOPATH/src/github.com/containernetworking/plugins/bin<br><br>$ ip netns add ns
 
$ ./cnitool add mynet /var/run/netns/ns
{
    "cniVersion": "0.3.0",
    "interfaces": [
        {
            "name": "cni0",
            "mac": "0a:58:0a:16:00:01"
        },
        {
            "name": "vetha418f787",
            "mac": "c6:e3:e9:1c:2f:20"
        },
        {
            "name": "eth0",
            "mac": "0a:58:0a:16:00:05",
            "sandbox": "/var/run/netns/ns"
        }
    ],
    "ips": [
        {
            "version": "4",
            "interface": 2,
            "address": "10.22.0.5/16",
            "gateway": "10.22.0.1"
        }
    ],
    "routes": [
        {
            "dst": "0.0.0.0/0"
        }
    ],
    "dns": {}
}
 
$ ip netns exec ns ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.22.0.5  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::646e:89ff:fea6:f9b5  prefixlen 64  scopeid 0x20<link>
        ether 0a:58:0a:16:00:05  txqueuelen 0  (Ethernet)
        RX packets 0  bytes 0 (0.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 8  bytes 648 (648.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```



### 4 怎么配置CNI?

从上文中我们可以知道，CNI只支持三种操作：ADD， DEL，VERSION，而三种操作所需要配置的参数和结果如下：

* 将container加入network（Add）：
  * Parameters:
    * Version：CNI版本信息
    * Container ID： 这个字段是可选的，但是建议使用，在容器活着的时候要求该字段全局唯一的。比如，存在IPAM的环境可能会要求每个container都分配一个独立的ID，这样每一个IP的分配都能和一个特定的容器相关联。在appc implementations中，container ID其实就是pod ID
    * Network namespace path：这个字段表示要加入的network namespace的路径。例如，/proc/[pid]/ns/net或者对于该目录的bind-mount/link
    * Network configuration： 这是一个JSON文件用于描述container可以加入的network，具体内容在下文中描述
    * Extra arguments：该字段提供了一种可选机制，从而允许基于每个容器进行CNI插件的简单配置
    * Name of the interface inside the container：该字段提供了在container （network namespace）中的interface的名字；因此，它也必须符合Linux对于网络命名的限制
  * Result:
    * Interface list：根据插件的不同，这个字段可以包括sandbox (container or hypervisor) interface的name，以及host interface的name，每个interface的hardware address，以及interface所在的sandbox（如果存在的话）的信息。
    * IP configuration assigned to each interface：IPv4和/或者IPv6地址，gateways以及为sandbox或host interfaces中添加的路由
    * DNS inormation：包含nameservers，domains，search domains和options的DNS information的字典
* 将container从network中删除（Delete）：
  * Parameter:
    * Version：CNI版本信息
    * ContainerID：定义同上
    * Network namespace path：定义同上
    * Network configuration：定义同上
    * Extra argument：定义同上
    * Name of the interface inside the container：定义同上
* 版本信息：
  * Parameter：无
  * Result：返回插件支持的所有CNI版本



在上文的叙述中我们省略了对Network configuration的描述。事实上，它的内容和上文演示实例中的"/etc/cni/net.d/10-mynet.conf"网络配置文件是一致的，用于描述容器了容器需要加入的网络，下面是对其中一些重要字段的描述：

* cniVersion（string）：cniVersion以Semantic Version 2.0的格式指定了插件使用的CNI版本
* name （string）：Network name。这应该在整个管理域中都是唯一的
* type （string）：插件类型，也代表了CNI插件可执行文件的文件名
* args （dictionary）：由容器运行时提供的可选的参数。比如，可以将一个由label组成的dictionary传递给CNI插件，通过在args下增加一个labels字段来实现
* ipMasqs (boolean)：可选项（如果插件支持的话）。为network在宿主机创建IP masquerade。如果需要将宿主机作为网关，为了能够路由到容器分配的IP，这个字段是必须的
* ipam：由特定的IPAM值组成的dictionary
  * type （string）：IPAM插件的类型，也表示IPAM插件的可执行文件的文件名
* dns：由特定的DNS值组成的dictionary
  * nameservers (list of strings)：一系列对network可见的，以优先级顺序排列的DNS nameserver列表。列表中的每一项都包含了一个IPv4或者一个IPv6地址
  * domain （string）：用于查找short hostname的本地域
  * search （list of strings）：以优先级顺序排列的用于查找short domain的查找域。对于大多数resolver，它的优先级比domain更高
  * options（list of strings）：一系列可以被传输给resolver的可选项

插件可能会定义它们自己能接收的额外的字段，但是遇到一个未知的字段可能会产生错误。例外的是args字段，它可以被用于传输一些额外的字段，但也可能会被插件忽略



### 5 CNI是怎么实现的？

到目前为止，我们对CNI的使用、配置和原理都已经有了基本的认识，所以也是时候基于源码来对CNI做一个透彻的理解了。下面，我们将以上文中的演示实例作为线索，以模拟程序cnitool作为切入口，来对整个CNI的执行过程进行详尽的分析。

（1）加载容器网络配置信息

首先我们来看一下容器网络配置的数据结构表示：

```go
type NetworkConfigList struct {
    Name       string
    CNIVersion string
    Plugins    []*NetworkConfig
    Bytes      []byte
}
 
type NetworkConfig struct {
    Network *types.NetConf
    Bytes   []byte
}
 
// NetConf describes a network.
type NetConf struct {
    CNIVersion string `json:"cniVersion,omitempty"`
 
    Name         string          `json:"name,omitempty"`
    Type         string          `json:"type,omitempty"`
    Capabilities map[string]bool `json:"capabilities,omitempty"`
    IPAM         struct {
        Type string `json:"type,omitempty"`
    } `json:"ipam,omitempty"`
    DNS DNS `json:"dns"`
}
```

经过粗略的分析之后，我们可以发现，数据结构表示的内容和演示实例中的json配置文件基本是一致的。因此，这一步的源码实现很简单，基本流程如下：

* 首先确定配置文件所在的目录netdir，如果没有特别指定，则默认为"/etc/cni/net.d"
* 调用netconf, err := libcni.LoadConfList(netdir, os.Args[2])，其中参数os.Args[2]为用户指定的想要加入的network的名字，在演示示例中即为"mynet"。该函数首先会查找netdir中是否有以".conflist"作为后缀的配置文件，如果有，且配置信息中的"Name"和参数os.Args[2]一致，则直接用配置信息填充并返回NetConfigList即可。否则，查找是否存在以".conf"或".json"作为后缀的配置文件。同样，如果存在"Name"一致的配置，则加载该配置文件。由于".conf"或".json"中都是单个的网络配置，因此需要将其包装成仅有一个NetConfig的NetworkConfigList再返回。到此为止，容器网络配置加载完成。

(2) 配置容器运行时信息

同样，我们先来看一下容器运行时信息的数据结构：

```go
type RuntimeConf struct {
    ContainerID string
    NetNS       string
    IfName      string
    Args        [][2]string
    // A dictionary of capability-specific data passed by the runtime
    // to plugins as top-level keys in the 'runtimeConfig' dictionary
    // of the plugin's stdin data.  libcni will ensure that only keys
    // in this map which match the capabilities of the plugin are passed
    // to the plugin
    CapabilityArgs map[string]interface{}
}
```

其中最重要的字段无疑是"NetNS"，它指定了需要加入容器网络的network namespace路径。而Args字段和CapabilityArgs字段都是可选的，用于传递额外的配置信息。具体的内容参见上文中的配置说明。在上文的演示实例中，我们并没有对Args和CapabilityArgs进行任何的配置，为了简单起见，我们可以直接认为它们为空。因此，cnitool对RuntimeConf的配置也就极为简单了，只需要将参数指定的netns赋值给NetNS字段，而ContainerID和IfName字段随意赋值即可，默认将它们分别赋值为"cni"和"eth0"，具体代码如下：

```go
rt := &libcni.RuntimeConf{
    ContainerID:    "cni",
    NetNS:          netns,
    IfName:         "eth0",
    Args:           cniArgs,
    CapabilityArgs: capabilityArgs,
}
```

(3) 加入容器网络

根据加载的容器网络配置信息和容器运行时信息，执行加入容器网络的操作，并将执行的结果打印输出

```go
switch os.Args[1] {
case CmdAdd:
    result, err := cninet.AddNetworkList(netconf, rt)
    if result != nil {
        _ = result.Print()
    }
    exit(err)
    ......
}
```

接下来我们进入AddNetworkList函数中

```go
// AddNetworkList executes a sequence of plugins with the ADD command
func (c *CNIConfig) AddNetworkList(list *NetworkConfigList, rt *RuntimeConf) (types.Result, error) {
    var prevResult types.Result
    for _, net := range list.Plugins {
        pluginPath, err := invoke.FindInPath(net.Network.Type, c.Path)
                .....
        newConf, err := buildOneConfig(list, net, prevResult, rt)
                ......
        prevResult, err = invoke.ExecPluginWithResult(pluginPath, newConf.Bytes, c.args("ADD", rt))
                ......
    }
 
    return prevResult, nil
}
```

从函数上方的注释我们就可以了解到，该函数的作用就是按顺序对NetworkList中的各个network执行ADD操作。该函数的执行过程也非常清晰，利用一个循环遍历NetworkList中的各个network，并对每个network进行如下三步操作：

* 首先，调用FindInPath函数，根据newtork的类型，在插件的存放路径，也就是上文中的CNI_PATH中查找是否存在对应插件的可执行文件。若存在则返回其绝对路径pluginPath
* 接着，调用buildOneConfig函数，从NetworkList中提取分离出当前执行ADD操作的network的NetworkConfig结构。这里特别需要注意的是preResult参数，它是上一个network的操作结果，也将被编码进NetworkConfig中。需要注意的是，当我们在执行NetworkList时，必须将前一个network的执行结果作为参数传递给当前正在进行执行的network。并且在buildOneConfig函数构建每个NetworkConfig时会默认将其中的"name"和"cniVersion"和NetworkList中的配置保持一致，从而避免冲突
* 最后，调用invoke.ExecPluginWithResult(pluginPath, netConf.Bytes, c.args("ADD", rt))真正执行network的ADD操作。这里我们需要注意的是netConf.Bytes和c.args("ADD", rt)这两个参数。其中netConf.Bytes用于存放NetworkConfig中的NetConf结构以及例如上文中的prevResult进行json编码形成的字节流。而c.args()函数用于构建一个Args类型的实例，其中主要存储容器运行时信息，以及执行的CNI操作的信息，例如"ADD"或"DEL"，和插件的存储路径

事实上ExecPluginWithResult仅仅是一个包装函数，它仅仅只是调用了函数defaultPluginExec.WithResult(pluginPath, netconf, args)之后，就直接返回了。

```go
func (e *PluginExec) WithResult(pluginPath string, netconf []byte, args CNIArgs) (types.Result, error) {
    stdoutBytes, err := e.RawExec.ExecPlugin(pluginPath, netconf, args.AsEnv())
        .....
    // Plugin must return result in same version as specified in netconf
    versionDecoder := &version.ConfigDecoder{}
    confVersion, err := versionDecoder.Decode(netconf)
        ....
    return version.NewResult(confVersion, stdoutBytes)
}
```

可以看得出WithResult函数的执行流也是非常清晰的，同样也可以分为以下三步执行：

* 首先调用e.RawExec.ExecPlugin(pluginPath, netconf, args.AsEnv())函数执行具体的CNI操作，对于它的具体内容，我们将在下文进行分析。此处需要注意的是它的第三个参数args.AsEnv()，该函数做的工作其实就是获取已有的环境变量，并且将args内的信息，例如CNI操作命令，以环境变量的形式保存起来，以例如"CNI_COMMAND=ADD"的形式传输给插件。由此我们可以知道，容器运行时信息、CNI操作命令以及插件存储路径都是以环境变量的形式传递给插件的
* 接着调用versionDecoder.Decode(netconf)从network配置中解析出CNI版本信息
* 最后，调用version.NewResult(confVersion, stdoutBytes)，根据CNI版本，构建相应的返回结果

最后，我们来看看e.RawExecPlugin函数是如何操作的，代码如下所示：

```go
func (e *RawExec) ExecPlugin(pluginPath string, stdinData []byte, environ []string) ([]byte, error) {
    stdout := &bytes.Buffer{}
 
    c := exec.Cmd{
        Env:    environ,
        Path:   pluginPath,
        Args:   []string{pluginPath},
        Stdin:  bytes.NewBuffer(stdinData),
        Stdout: stdout,
        Stderr: e.Stderr,
    }
    if err := c.Run(); err != nil {
        return nil, pluginErr(err, stdout.Bytes())
    }
 
    return stdout.Bytes(), nil
}
```

在看完代码之后，我们可能有些许的失望。因为这个理论上最为核心的函数却出乎意料的简单，它所做的工作仅仅只是exec了插件的可执行文件。话虽如此，我们仍然有以下几点需要注意：

* 容器运行时信息以及CNI操作命令等都是以环境变量的形式传递给插件的，这点在上文中已经有所提及
* 容器网络的配置信息是通过标准输入的形式传递给插件的
* 插件的运行结果是以标准输出的形式返回给CNI的

到此为止，整个CNI的执行流已经非常清楚了。简单地说，一个CNI插件就是一个可执行文件，我们从配置文件中获取network配置信息，从容器管理系统处获取运行时信息，再将前者以标准输入的形式，后者以环境变量的形式传递传递给插件，最终以配置文件中定义的顺序依次调用各个插件，并且将前一个插件的执行结果包含在network配置信息中传递给下一个执行的插件，整个过程就是这样。鉴于篇幅所限，本文仅仅只分析了CNI的ADD操作，不过相信有了上文的基础之后，理解DEL操作也不会太难。



### 参考链接

* [CNI源码](https://github.com/containernetworking/cni)

* [CNI plugin源码](https://github.com/containernetworking/plugins)

* [Kubernetes指南](https://feisky.gitbooks.io/kubernetes/network/cni/)

* [CNI：容器网络接口](http://cizixs.com/2017/05/23/container-network-cni)

