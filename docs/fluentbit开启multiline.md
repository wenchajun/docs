# docs

#### 收集多行日志的测试

**tail** 输入插件允许监测一个或多个文本文件。它具有类似于 `tail -f` 的 shell 命令行功能。

该插件读取 *Path* 模式中的每个匹配文件，并为每个新行(分隔符为`\n`)生成一条新纪录。作为可选的，可以使用数据库文件，以便插件可以跟踪文件的历史记录和偏移状态，这对于重启服务时的状态恢复非常有用。



# **配置参数**

该插件支持以下配置参数:

| 键                | 描述                                                         | 默认值            |
| ----------------- | ------------------------------------------------------------ | ----------------- |
| Buffer_Chunk_Size | 设置读取文件数据的初始缓冲区大小。该值也用于增加缓冲区大小。该值必须符合[单位](https://hulining.gitbook.io/fluentbit/administration/configuring-fluent-bit/unit-sizes)章节中的规范 | 32k               |
| Buffer_Max_Size   | 设置每个监控文件的缓冲区大小。当需要增加缓冲区时，此值用于限制内存缓冲区可以增加多少。如果超过此限制，则将从监控文件列表中删除该文件。该值必须符合[单位](https://hulining.gitbook.io/fluentbit/administration/configuring-fluent-bit/unit-sizes)章节中的规范 | Buffer_Chunk_Size |
| Path              | 通过使用通配符指定一个或多个日志文件的                       |                   |
| Path_Key          | 如果启用，它将附加监控文件的名称作为记录的一部分。指定的值成为映射中的键 |                   |
| Exclude_Path      | 设置一个或多个用逗号分隔的模式，以排除符合特定条件的文件。例如 exclude_path=*.gz,*.zip |                   |
| Refresh_Interval  | 刷新监控文件列表的时间间隔(秒)                               | 60                |
| Rotate_Wait       | 指定监控文件的额外时间，以防止日志文件滚动丢失某些数据       | 5                 |
| Ignore_Older      | 忽略比该时间旧的记录(秒)。支持 m,h,d(分钟,小时,天)。默认行为是从指定文件中读取所有记录。 仅在指定了解析器并且可以解析记录时间时才可用 |                   |
| Skip_Long_Lines   | 当监控的文件由于行(Buffer_Max_Size)很长而达到缓冲区容量时，默认行为是停止监视该文件。Skip_Long_Lines 会更改该行为，并指示 Fluent Bit 跳过长行并继续处理适合缓冲区大小的其他行 | Off               |
| DB                | 指定跟踪监控文件的偏移量的数据库文件                         |                   |
| DB.Sync           | 设置默认的同步方法。可选值: Extra, Full, Normal, Off.此标志影响内部 SQLite 引擎与磁盘同步的方式，有关选项的更多详细信息，请参阅 [sqlite 文档](https://www.sqlite.org/pragma.html#pragma_synchronous). | Full              |
| Mem_Buf_Limit     | 设置将数据追加到引擎时的内存限制。如果达到此限制，它将被暂停；刷新数据后，它将恢复. |                   |
| Parser            | 指定解析器的名称，将记录转化为结构化消息                     |                   |
| Key               | 当消息是非结构化数据时(未应用解析器)，消息将以字符串形式作为 *`log`* 键的值。此选项允许为该键指定名称 | log               |
| Tag               | 为读取的行设置标签(带有正则表达式字段)。如 `kube.<namespace_name>.<pod_name>.<container_name>`.请注意支持如下"标签扩展"规则: 如果标签包含星号(*)，则星号(*)将被替换为文件的绝对路径(请参阅 [Workflow of Tail + Kubernetes Filter](https://hulining.gitbook.io/fluentbit/pipeline/filters/kubernetes#workflow-of-tail-kubernetes-filter) |                   |
| Tag_Regex         | 设置正则表达式以从文件中提取字段.如 `(?<pod_name>[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*)_(?<namespace_name>[^_]+)_(?<container_name>.+)-` |                   |

请注意，如果未指定数据库参数 `*db*`，默认情况下，插件将从头开始读取每个目标文件。

## **多行配置参数**

此外，还存在以下选项来配置多行文件的处理:

| 键               | 描述                                                         | 默认值 |
| ---------------- | ------------------------------------------------------------ | ------ |
| Multiline        | 如果启用，该插件将尝试发现多行消息并使用适当的解析器组成输出消息。请注意，启用此选项后，将不使用 Parser 选项 | Off    |
| Multiline_Flush  | 处理队列中多行消息的等待时间                                 | 4      |
| Parser_Firstline | 多行消息的开头匹配的解析器的名称。请注意，解析器中定义的正则表达式必须包含组名(名为capture ) |        |
| Parser_N         | 可选的额外解析器，用于解析并结构化多行条目。此选项可用于定义多个解析器，例如：Parser_1 ab1，Parser_2 ab2，Parser_N abN |        |

输入信息

```

Dec 14 06:41:08 Exception in thread main java.lang.RuntimeException:
 Something has gone wrong, aborting!
 com.myproject.module.MyProject.badMethod(MyProject.java:22)
 at com.myproject.module.MyProject.oneMoreMethod(MyProject.java:18)
 at com.myproject.module.MyProject.anotherMethod(MyProject.java:14)
 at com.myproject.module.MyProject.someMethod(MyProject.java:10)
 at com.myproject.module.MyProject.main(MyProject.java:6)
```

在这里为了测试多行输出情况：

```

apiVersion: logging.kubesphere.io/v1alpha2
kind: Input
metadata:
  annotations:
  labels:
    logging.kubesphere.io/component: logging
    logging.kubesphere.io/enabled: 'true'
  name: tail
  namespace: kubesphere-logging-system
spec:
  tail:
    bufferChunkSize: 6000K
    bufferMaxSize: 30MB
    db: /fluent-bit/tail/pos.db
    dbSync: Normal
    excludePath: /var/log/containers/*_kube*-system_*.log
    multiline: true
    parserFirstline: capture
    path: /var/log/containers/*.log
    tag: kube.*





```

对应的conf文件：

```

[Input]
    Name    systemd
    Path    /var/log/journal
    DB    /fluent-bit/tail/docker.db
    DB.Sync    Normal
    Tag    service.docker
    Systemd_Filter    _SYSTEMD_UNIT=docker.service
[Input]
    Name    systemd
    Path    /var/log/journal
    DB    /fluent-bit/tail/kubelet.db
    DB.Sync    Normal
    Tag    service.kubelet
    Systemd_Filter    _SYSTEMD_UNIT=kubelet.service
[Input]
    Name    tail
    Buffer_Chunk_Size    6000K
    Buffer_Max_Size    30MB
    Path    /var/log/containers/*.log
    Exclude_Path    /var/log/containers/*_kube*-system_*.log
    DB    /fluent-bit/tail/pos.db
    DB.Sync    Normal
    Tag    kube.*
    Multiline    true
    Parser_Firstline    capture
    
```

parser对应的yaml

```

apiVersion: logging.kubesphere.io/v1alpha2
kind: Parser
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: >
      {"apiVersion":"logging.kubesphere.io/v1alpha2","kind":"Parser","metadata":{"annotations":{},"labels":{"logging.kubesphere.io/enabled":"true"},"name":"capture","namespace":"kubesphere-logging-system"},"spec":{"regex":{"regex":"(?\u003ccapture\u003e\\d+-\\d+-\\d+\\s\\d+:\\d+:\\d+,\\d+\\s[^\\s]+:.{24})"}}}
  labels:
    logging.kubesphere.io/enabled: 'true'
  name: capture
  namespace: kubesphere-logging-system
spec:
  regex:
    regex: '/(?<time>Dec \d+ \d+\:\d+\:\d+)(?<capture>.*)/'
    timeFormat: '%b %d %H:%M:%S'
    timeKey: time

```

对应的conf文件：

```

[PARSER]
    Name    capture
    Format    regex
    Regex    /(?<time>Dec \d+ \d+\:\d+\:\d+)(?<capture>.*)/
    Time_Key    time
    Time_Format    %b %d %H:%M:%S
```

tail中对应的参数：

> Buffer_Chunk_Size    6000K
>
>设置读取文件数据的初始缓冲区大小
>
>Buffer_Max_Size    30MB
>
>设置每个监控文件的缓冲区大小
>
>multiline: true
>parserFirstline: capture

而且值得注意的是：

>- Buffer_Max_Size 必须大于Buffer_Chunk_Size
>- Parser_Firstline                                                                                                                                                                                          多行消息的开头匹配的解析器的名称。请注意，解析器中定义的正则表达式必须包含组名(名为capture )
>- on multiline mode 'Parser' is not allowed (parser disabled)

output中yaml文件：

```
apiVersion: logging.kubesphere.io/v1alpha2
kind: Output
metadata:
  labels:
    logging.kubesphere.io/enabled: 'true'
  name: stdout
  namespace: kubesphere-logging-system
spec:
  match: '*'
  stdout: {}

```

对应的conf：

```
[Output]                                                                  
    Name    stdout                                                        
    Match    *   

```

log查看输出：

````

[2021/06/27 07:54:50] [ info] [input:tail:tail.2] inotify_fs_add(): inode=8257618 watch_fd=3 name=/var/log/containers/fb-test-pod_default_fb-test01-142eedfd1d0c65cf40377b486160ea6da7a33eb7198310cb0fda626fd5999f91.log
[0] kube.var.log.containers.fb-test-pod_default_fb-test01-142eedfd1d0c65cf40377b486160ea6da7a33eb7198310cb0fda626fd5999f91.log: [1639464068.000000000, {"capture"=>" Exception in thread main java.lang.RuntimeException: Something has gone wrong, aborting!\n","stream":"stdout","time":"2021-06-27T07:54:33.640178105Z"}
....
]

````

