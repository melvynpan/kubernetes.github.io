---
approvers:
- crassirostris
- piosz
title: 日志架构
---

<!--
---
approvers:
- crassirostris
- piosz
title: Logging Architecture
---
-->

<!--
Application and systems logs can help you understand what is happening inside your cluster. The logs are particularly useful for debugging problems and monitoring cluster activity. Most modern applications have some kind of logging mechanism; as such, most container engines are likewise designed to support some kind of logging. The easiest and most embraced logging method for containerized applications is to write to the standard output and standard error streams.
-->

应用和系统日志可以帮助您了解集群内正在发生什么。日志对调试问题和监控集群活动非常有用。大部分现代化应用都有某种日志记录机制；同样地，大多数容器引擎也被设计成支持某种日志记录机制。容器化应用中最简单且受欢迎的日志记录方式就是写入标准输出和标准错误流。

<!--
However, the native functionality provided by a container engine or runtime is usually not enough for a complete logging solution. For example, if a container crashes, a pod is evicted, or a node dies, you'll usually still want to access your application's logs. As such, logs should have a separate storage and lifecycle independent of nodes, pods, or containers. This concept is called _cluster-level-logging_. Cluster-level logging requires a separate backend to store, analyze, and query logs. Kubernetes provides no native storage solution for log data, but you can integrate many existing logging solutions into your Kubernetes cluster.
-->

但是，由容器引擎或 runtime 提供的原生功能通常不足以满足完整的日志记录方案。例如，如果容器崩溃, pod 被驱逐,或 node 宕机时,您仍然想访问您的应用日志。这种情形下，日志应该具有独立的存储和生命周期,不依赖于 node , pods ,或容器。这个概念叫 _集群级的日志_ 。集群级日志记录需要一个独立的后台来存储，分析和查询日志。Kubernetes 没有为日志数据提供原生存储方案，但是您可以将许多现有的的日志解决方案集成到您的 Kubernetes 集群。


* TOC
{:toc}

<!--
Cluster-level logging architectures are described in assumption that
a logging backend is present inside or outside of your cluster. If you're
not interested in having cluster-level logging, you might still find
the description of how logs are stored and handled on the node to be useful.
-->

集群级日志记录架构假定有一个日志后台在您的集群内部或者外部。如果您对集群级日志不感兴趣，那么有关如何在节点存储和处理日志的介绍，或许对您有帮助。


<!--
## Basic logging in Kubernetes

In this section, you can see an example of basic logging in Kubernetes that
outputs data to the standard output stream. This demonstration uses
a [pod specification](/docs/concepts/cluster-administration/counter-pod.yaml) with
a container that writes some text to standard output once per second.
-->

## Kubernetes中的常规日志记录

本节,您会看到 kubernetes 中向标准输出写入日志的例子。该演示通过 [定义 pod ](/docs/concepts/cluster-administration/counter-pod.yaml) 创建一个每秒向标准输出写入数据的容器。


{% include code.html language="yaml" file="counter-pod.yaml" ghlink="/docs/tasks/debug-application-cluster/counter-pod.yaml" %}

<!--
To run this pod, use the following command:
-->

用下面的命令运行本pod：

```shell
$ kubectl create -f https://k8s.io/docs/tasks/debug-application-cluster/counter-pod.yaml
pod "counter" created
```

<!--
To fetch the logs, use the `kubectl logs` command, as follows:
-->

使用`kubectl logs`命令获取日志:

```shell
$ kubectl logs counter
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

<!--
You can use `kubectl logs` to retrieve logs from a previous instantiation of a container with `--previous` flag, in case the container has crashed. If your pod has multiple containers, you should specify which container's logs you want to access by appending a container name to the command. See the [`kubectl logs` documentation](/docs/user-guide/kubectl/{{page.version}}/#logs) for more details.
-->

一旦容器崩溃的话，您可以使用 `kubectl logs` 和参数 `--previous` 恢复之前的容器日志。如果您的 pod 有多个容器，您应该通过命令指定容器名访问相应的容器日志。详见 [`kubectl logs` 文档](/docs/user-guide/kubectl/{{page.version}}/#logs)。

<!--
## Logging at the node level
-->

## 节点级别日志

![Node level logging](/images/docs/user-guide/logging/logging-node-level.png)

<!--
Everything a containerized application writes to `stdout` and `stderr` is handled and redirected somewhere by a container engine. For example, the Docker container engine redirects those two streams to [a logging driver](https://docs.docker.com/engine/admin/logging/overview), which is configured in Kubernetes to write to a file in json format.
-->

容器化应用写入 `stdout` 和 `stderr` 的任何数据,都会被容器引擎捕获并被重定向到某个位置。例如，Docker 容器引擎重定向这两个输出流到 [日志驱动](https://docs.docker.com/engine/admin/logging/overview) ，该日志驱动在 Kubernetes 中被配置成写入json文件。

<!--
**Note:** The Docker json logging driver treats each line as a separate message. When using the Docker logging driver, there is no direct support for multi-line messages. You need to handle multi-line messages at the logging agent level or higher.
-->

**提示:**  Docker json 日志驱动将日志的每一行当作一条独立的消息。在使用该日志驱动时，不直接支持多行消息。您需要在日志代理级别或更高级别处理多行消息。

<!--
By default, if a container restarts, the kubelet keeps one terminated container with its logs. If a pod is evicted from the node, all corresponding containers are also evicted, along with their logs.
-->

默认情况下，如果容器重启，kubelet 会保留被终止的容器日志。如果 pod 从工作节点驱逐，该 pod 中所有的容器也会被驱逐，包括容器日志。

<!--
An important consideration in node-level logging is implementing log rotation,
so that logs don't consume all available storage on the node. Kubernetes
currently is not responsible for rotating logs, but rather a deployment tool
should set up a solution to address that.
For example, in Kubernetes clusters, deployed by the `kube-up.sh` script,
there is a [`logrotate`](https://linux.die.net/man/8/logrotate)
tool configured to run each hour. You can also set up a container runtime to
rotate application's logs automatically, e.g. by using Docker's `log-opt`.
In the `kube-up.sh` script, the latter approach is used for COS image on GCP,
and the former approach is used in any other environment. In both cases, by
default rotation is configured to take place when log file exceeds 10MB.
-->

节点级别日志中，需要重点考虑实现日志的轮转，以此来保证日志不会消耗节点上所有的可用空间。Kubernetes 当前并不负责轮转日志，而是通过部署工具建立一个解决问题的方案。例如，在 Kubernetes 集群中，用 `kube-up.sh` 部署一个每小时运行的工具 [`logrotate`](https://linux.die.net/man/8/logrotate) 。您也可以设置容器 runtime 来自动地轮转应用日志，比如，使用 Docker的 `log-opt` 选项。在 `kube-up.sh` 脚本中，后一种方式适用于 GCP 的 COS 镜像，而前一种方式适用于任何环境。这两种情况下，默认日志超过10MB大小时触发日志轮转。

<!--
As an example, you can find detailed information about how `kube-up.sh` sets
up logging for COS image on GCP in the corresponding [script]
[cosConfigureHelper].
-->

例如，您可以发现 `kube-up.sh` 怎样为 GCP 环境的 COS 镜像设置日志的详细信息，相应的脚本在 [script][cosConfigureHelper] 。

<!--
When you run [`kubectl logs`](/docs/user-guide/kubectl/{{page.version}}/#logs) as in
the basic logging example, the kubelet on the node handles the request and
reads directly from the log file, returning the contents in the response.
**Note:** currently, if some external system has performed the rotation,
only the contents of the latest log file will be available through
`kubectl logs`. E.g. if there's a 10MB file, `logrotate` performs
the rotation and there are two files, one 10MB in size and one empty,
`kubectl logs` will return an empty response.
-->

和常规日志记录的例子一样，运行 [`kubectl logs`](/docs/user-guide/kubectl/{{page.version}}/#logs) 时，节点上的 kubelet 处理该请求并直接读取日志文件，同时在响应中返回日志文件内容。
**提示：** 当前，如果有其他系统机制执行轮转操作，`kubectl logs` 仅可查询到最新的日志内容，比如，一个10MB大小的文件，`logrotate` 执行轮转操作后有两个文件，一个10MB大小，一个为空，所以 `kubectl logs` 将返回空。


[cosConfigureHelper]: https://github.com/kubernetes/kubernetes/blob/{{page.githubbranch}}/cluster/gce/gci/configure-helper.sh

<!--
### System component logs

There are two types of system components: those that run in a container and those
that do not run in a container. For example:
-->

### 系统组件日志

有两种类型的系统组件，运行在容器里的和未运行在容器里的。例如：

<!--
* The Kubernetes scheduler and kube-proxy run in a container.
* The kubelet and container runtime, for example Docker, do not run in containers.
-->

* 运行在容器中的 Kubernetes scheduler 和 kube-proxy 。
* 未运行在容器中的 kubelet 和容器 runtime，比如 Docker 。

<!--
On machines with systemd, the kubelet and container runtime write to journald. If
systemd is not present, they write to `.log` files in the `/var/log` directory.
System components inside containers always write to the `/var/log` directory,
bypassing the default logging mechanism. They use the [glog][glog]
logging library. You can find the conventions for logging severity for those
components in the [development docs on logging](https://git.k8s.io/community/contributors/devel/logging.md).
-->

在使用 systemd 机制的服务器上，kubelet 和容器 runtime 写入日志到 journald。如果没有 systemd ，他们写入日志到 `/var/log` 目录的`.log`文件。容器中的系统组件通常将日志写到 `/var/log` 目录，绕过了默认的日志机制。他们使用 [glog][glog] 日志库。您可以在 [development docs on logging](https://git.k8s.io/community/contributors/devel/logging.md) 找到这些组件的日志告警级别协议。

<!--
Similarly to the container logs, system component logs in the `/var/log`
directory should be rotated. In Kubernetes clusters brought up by
the `kube-up.sh` script, those logs are configured to be rotated by
the `logrotate` tool daily or once the size exceeds 100MB.
-->

和容器日志类似，`/var/log` 目录中的系统组件日志应该被轮转，通过脚本
 `kube-up.sh` 启动的 Kubernetes 集群，他们的日志被工具 `logrotate` 配置成每日轮转，或者日志大小超过100MB时轮转。

[glog]: https://godoc.org/github.com/golang/glog

<!--
## Cluster-level logging architectures
-->

## 集群级别日志架构

<!--
While Kubernetes does not provide a native solution for cluster-level logging, there are several common approaches you can consider. Here are some options:

* Use a node-level logging agent that runs on every node.
* Include a dedicated sidecar container for logging in an application pod.
* Push logs directly to a backend from within an application.

-->

虽然 Kubernetes 并未提供原生的集群级记录日志方案，但是您可以考虑几种常见的方式。以下是一些选项：

* 使用运行在每个节点上的节点级的日志代理。
* 在应用的 pod 中，包含专门记录日志的伴生容器。
* 将日志从应用中直接推送到后台。

<!--
### Using a node logging agent
-->

### 使用节点级日志代理

![Using a node level logging agent](/images/docs/user-guide/logging/logging-with-node-agent.png)

<!--
You can implement cluster-level logging by including a _node-level logging agent_ on each node. The logging agent is a dedicated tool that exposes logs or pushes logs to a backend. Commonly, the logging agent is a container that has access to a directory with log files from all of the application containers on that node.
-->

您可以在每个节点上使用 _节点级的日志代理_ 来实现集群级日志记录。日志代理是专门的工具，它会暴露出日志或将日志推送到后台。通常来说，日志代理是一个容器，这个容器可以访问这个节点上所有应用容器的日志目录。

<!--
Because the logging agent must run on every node, it's common to implement it as either a DaemonSet replica, a manifest pod, or a dedicated native process on the node. However the latter two approaches are deprecated and highly discouraged.
-->

因为日志代理必须在每个节点上运行，所以通常的实现方式为，DaemonSet副本，manifest pod，或者专用于本地的进程。然而，后两种方式已被弃用并强烈不推荐。

<!--
Using a node-level logging agent is the most common and encouraged approach for a Kubernetes cluster, because it creates only one agent per node, and it doesn't require any changes to the applications running on the node. However, node-level logging _only works for applications' standard output and standard error_.
-->

对于 Kubernetes 集群来说，使用节点级的日志代理是最常用和被鼓励的方式，因为在每个节点上仅创建一个代理，并且不需要对节点上的应用做修改。但是，节点级的日志 _仅适用于应用程序的标准输出和标准错误输出_。

<!--
Kubernetes doesn't specify a logging agent, but two optional logging agents are packaged with the Kubernetes release: [Stackdriver Logging](/docs/user-guide/logging/stackdriver) for use with Google Cloud Platform, and [Elasticsearch](/docs/user-guide/logging/elasticsearch). You can find more information and instructions in the dedicated documents. Both use [fluentd](http://www.fluentd.org/) with custom configuration as an agent on the node.
-->

Kubernetes 没有指定日志代理，但是有两个可选的日志代理与 Kubernetes 发行版一起打包。[Stackdriver Logging](/docs/user-guide/logging/stackdriver) 适用于 Google Cloud Platform，和[Elasticsearch](/docs/user-guide/logging/elasticsearch)。您可以在专门的文档中找到更多的信息和说明。两者都使用 [fluentd](http://www.fluentd.org/) 配合自定义配置作为节点上的代理。

<!--
### Using a sidecar container with the logging agent
-->

### 使用伴生容器和日志代理

<!--
You can use a sidecar container in one of the following ways:
-->

您可以通过以下方式之一使用伴生容器：

<!--
* The sidecar container streams application logs to its own `stdout`.
* The sidecar container runs a logging agent, which is configured to pick up logs from an application container.
-->

* 伴生容器向本身的标准输出写入应用日志流。
* 伴生容器运行一个日志代理，该日志代理被配置成从应用容器收集日志。

<!--
#### Streaming sidecar container
-->

#### 传输数据流的伴生容器

<!--
![Sidecar container with a streaming container](/images/docs/user-guide/logging/logging-with-streaming-sidecar.png)

By having your sidecar containers stream to their own `stdout` and `stderr`
streams, you can take advantage of the kubelet and the logging agent that
already run on each node. The sidecar containers read logs from a file, a socket,
or the journald. Each individual sidecar container prints log to its own `stdout`
or `stderr` stream.
-->

利用伴生容器向自身的 `stdout` 和 `stderr` 传输流的方式，您就可以利用每个节点上的 kubelet 和日志代理来处理日志。伴生容器从文件，socket 或 journald 读取日志。每个伴生容器打印其自己的 `stdout` 和 `stderr` 流。

<!--
This approach allows you to separate several log streams from different
parts of your application, some of which can lack support
for writing to `stdout` or `stderr`. The logic behind redirecting logs
is minimal, so it's hardly a significant overhead. Additionally, because
`stdout` and `stderr` are handled by the kubelet, you can use built-in tools
like `kubectl logs`.
-->

这种方式允许您分离出不同的日志流，这些日志流来自您应用的不同功能，其中一些可能缺乏对写入 `stdout` 和 `stderr` 的支持。背后重定向的逻辑很小，所以不会是很严重的开销。除此之外，因为kubelet处理 `stdout` 和 `stderr`，所以您也可以使用 `kubectl logs` 工具。

<!--
Consider the following example. A pod runs a single container, and the container
writes to two different log files, using two different formats. Here's a
configuration file for the Pod:
-->

考虑接下来的例子。pod的容器向两个文件写不同格式的日志，下面是这个pod的配置文件:

{% include code.html language="yaml" file="two-files-counter-pod.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod.yaml" %}

<!--
It would be a mess to have log entries of different formats in the same log
stream, even if you managed to redirect both components to the `stdout` stream of
the container. Instead, you could introduce two sidecar containers. Each sidecar
container could tail a particular log file from a shared volume and then redirect
the logs to its own `stdout` stream.
-->

在同一个日志流中有两种不同格式的日志条目，这有点混乱，即使您试图重定向它们到容器的 `stdout` 流。取而代之的是，您可以引入两个伴生容器。每一个伴生容器可以从共享卷跟踪特定的日志文件，并重定向文件内容到各自的 `stdout` 流。

<!--
Here's a configuration file for a pod that has two sidecar containers:
-->

这是运行两个伴生容器的 pod 文件。

{% include code.html language="yaml" file="two-files-counter-pod-streaming-sidecar.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod-streaming-sidecar.yaml" %}

Now when you run this pod, you can access each log stream separately by
running the following commands:

现在当您运行这个 pod 时，您可以分别地访问每一个日志流，运行如下命令：

```shell
$ kubectl logs counter count-log-1
0: Mon Jan  1 00:00:00 UTC 2001
1: Mon Jan  1 00:00:01 UTC 2001
2: Mon Jan  1 00:00:02 UTC 2001
...
```

```shell
$ kubectl logs counter count-log-2
Mon Jan  1 00:00:00 UTC 2001 INFO 0
Mon Jan  1 00:00:01 UTC 2001 INFO 1
Mon Jan  1 00:00:02 UTC 2001 INFO 2
...
```

<!--
The node-level agent installed in your cluster picks up those log streams
automatically without any further configuration. If you like, you can configure
the agent to parse log lines depending on the source container.
-->

无需深入配置，集群中的节点级代理即可自动地收集流日志。如果您愿意，您可以配置代理程序来解析源容器的日志行。

<!--
Note, that despite low CPU and memory usage (order of couple of millicores
for cpu and order of several megabytes for memory), writing logs to a file and
then streaming them to `stdout` can double disk usage. If you have
an application that writes to a single file, it's generally better to set
`/dev/stdout` as destination rather than implementing the streaming sidecar
container approach.
-->

注意，尽管 CPU 和内存使用率都很低（以多个 cpu millicores 指标排序或者按 memory 的兆字节排序），向文件写日志然后输出到 `stdout` 流仍然会成倍地增加磁盘使用率。如果您的应用向单一文件写日志，通常最好设置 `/dev/stdout` 作为目标路径，而不是使用流式的伴生容器方式。

<!--
Sidecar containers can also be used to rotate log files that cannot be
rotated by the application itself. [An example](https://github.com/samsung-cnct/logrotate)
of this approach is a small container running logrotate periodically.
However, it's recommended to use `stdout` and `stderr` directly and leave rotation
and retention policies to the kubelet.
-->

应用本身如果不具备轮转日志文件的功能，可以通过伴生容器实现。该方式的 [例子](https://github.com/samsung-cnct/logrotate) 是运行一个定期轮转日志的容器。然而，还是推荐直接使用 `stdout` 和 `stderr`，将日志的轮转和保留策略交给kubelet。

<!--
#### Sidecar container with a logging agent
-->

### 具有日志代理功能的伴生容器

![Sidecar container with a logging agent](/images/docs/user-guide/logging/logging-with-sidecar-agent.png)

<!--
If the node-level logging agent is not flexible enough for your situation, you
can create a sidecar container with a separate logging agent that you have
configured specifically to run with your application.
-->

如果节点级的日志代理对您的环境来说不够灵活，您可以在伴生容器中创建一个独立的、专门为您的应用而配置的日志代理。

<!--
**Note**: Using a logging agent in a sidecar container can lead
to significant resource consumption. Moreover, you won't be able to access
those logs using `kubectl logs` command, because they are not controlled
by the kubelet.
-->

**提示**：在伴生容器中使用日志代理会导致严重的资源损耗。此外，您不能使用 `kubectl logs` 命令访问日志，因为日志并没有被 kubelet 管理。

<!--
As an example, you could use [Stackdriver](/docs/tasks/debug-application-cluster/logging-stackdriver/),
which uses fluentd as a logging agent. Here are two configuration files that
you can use to implement this approach. The first file contains
a [ConfigMap](/docs/tasks/configure-pod-container/configmap/) to configure fluentd.
-->

例如，您可以使用 [Stackdriver](/docs/tasks/debug-application-cluster/logging-stackdriver/) ，它用 fluentd 作为日志代理。这是实现此种方式的两个配置文件。第一个文件包含配置 fluentd 的 [ConfigMap](/docs/tasks/configure-pod-container/configmap/) 。

{% include code.html language="yaml" file="fluentd-sidecar-config.yaml" ghlink="/docs/concepts/cluster-administration/fluentd-sidecar-config.yaml" %}

<!--
**Note**: The configuration of fluentd is beyond the scope of this article. For
information about configuring fluentd, see the
[official fluentd documentation](http://docs.fluentd.org/).
-->

**提示**：fluentd 的配置文件已经超出了本文的讨论范畴。更多配置 fluentd 的信息，请见 [官方 fluentd 文档](http://docs.fluentd.org/) 。


<!--
The second file describes a pod that has a sidecar container running fluentd.
The pod mounts a volume where fluentd can pick up its configuration data.
-->

第二个文件描述了运行 fluentd 伴生容器的 pod 。flutend 通过 pod 的挂载卷获取它的配置数据。

{% include code.html language="yaml" file="two-files-counter-pod-agent-sidecar.yaml" ghlink="/docs/concepts/cluster-administration/two-files-counter-pod-agent-sidecar.yaml" %}

<!--
After some time you can find log messages in the Stackdriver interface.
-->

一段时间后，您可以在 Stackdriver 界面看到日志消息。

<!--
Remember, that this is just an example and you can actually replace fluentd
with any logging agent, reading from any source inside an application
container.
-->

记住，这只是一个例子，事实上您可以用任何一个日志代理替换 fluentd ，并从应用容器中读取任何资源。

<!--
### Exposing logs directly from the application
-->

### 从应用中直接暴露日志目录

![Exposing logs directly from the application](/images/docs/user-guide/logging/logging-from-application.png)

<!--
You can implement cluster-level logging by exposing or pushing logs directly from
every application; however, the implementation for such a logging mechanism
is outside the scope of Kubernetes.
-->

通过暴露或推送每个应用的日志，您可以实现集群级日志记录；然而，这种日志记录机制的实现已超出 Kubernetes 的范围。
