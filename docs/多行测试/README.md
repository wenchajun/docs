# docs
#### 收集多行日志的测试

在fluent bit中日志格式如下

```
2019-08-01 18:58:05,898 ERROR:Exception on main handler
Traceback (most recent call last):
  File "python-logger.py", line 9, in make_log
    return word[13]
IndexError: string index out of range
```

文章中采用的config的方式进行配置，config配置如下

```
[PARSER]
    Name         log_date
    Format       regex
    Regex        /\d{4}-\d{1,2}-\d{1,2}/

[PARSER]
    Name        log_attributes
    Format      regex
    Regex       /(?<timestamp>[^ ]* [^ ]*) (?<level>[^\s]+:)(?<message>[\s\S]*)/

 [INPUT]
    Name              tail
    tag               sample.tag
    path              /path/to/pythonApp.log
    Multiline         On
    Parser_Firstline  log_date
    Parser_1          log_attributes
```

文中通过以上配置参数收集到多行日志文件，采用了fluent-bit插件中的Multiline，Parser_Firstline以及Parser_1 。

而在fluentbit-operator中通过yaml文件配置fluentbit的config，相关参数及定义如下：

```
     // If enabled, the plugin will try to discover multiline messages
    // and use the proper parsers to compose the outgoing messages.
   // Note that when this option is enabled the Parser option is not used.
Multiline *bool `json:"multiline,omitempty"`
	// Wait period time in seconds to process queued multiline messages
MultilineFlushSeconds *int64 `json:"multilineFlushSeconds,omitempty"`
	// Name of the parser that matchs the beginning of a multiline message.
	// Note that the regular expression defined in the parser must include a          group name (named capture)
ParserFirstline string `json:"parserFirstline,omitempty"`
	// Optional-extra parser to interpret and structure multiline entries.
	// This option can be used to define multiple parsers.
ParserN []string `json:"parserN,omitempty"`
	
```

制作日志打印工具，模仿打印输出信息如下

>2021-06-22 18:58:05,898 ERROR:Exception on main handler
>name:eloncheng||fluent-bit-test|The current number is600
>name:eloncheng||fluent-bit-test|The current number is600
>name:eloncheng||fluent-bit-test|The current number is600
>name:eloncheng||fluent-bit-test|The current number is600
>name:eloncheng||fluent-bit-test|The current number is600

因为fluentbit- operator对fluentbit进行了修改，所以根据operator中的代码配置tail.yaml文件。其中，将第一行2021-06-22 18:58:05,898设为parserFirstline的值。由于parserN是可选的，为了测试暂时不将它赋值。

```
multiline: true
multilineFlushSeconds: 1
parserFirstline: '(?<capture>\d+-\d+-\d+\s\d+:\d+:\d+,\d+)'
```



在这种情况下收集日志条数为0。
