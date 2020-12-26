# pprof

golang pprof是golang的可视化和性能分析的工具。其提供了可视化的web页面，火焰图等更直观的工具。

可以使用 go tool pprof 进行使用

```sh
$ go tool pprof
usage:

Produce output in the specified format.

   pprof <format> [options] [binary] <source> ...

Omit the format to get an interactive shell whose commands can be used
to generate various views of a profile

   pprof [options] [binary] <source> ...

Omit the format and provide the "-http" flag to get an interactive web
interface at the specified host:port that can be used to navigate through
various views of a profile.

   pprof -http [host]:[port] [options] [binary] <source> ...

Details:
  Output formats (select at most one):
    -callgrind       Outputs a graph in callgrind format
    -comments        Output all profile comments
    -disasm          Output assembly listings annotated with samples
    -dot             Outputs a graph in DOT format
    -eog             Visualize graph through eog
    -evince          Visualize graph through evince
    -gif             Outputs a graph image in GIF format
    -gv              Visualize graph through gv
    -kcachegrind     Visualize report in KCachegrind
    -list            Output annotated source for functions matching regexp
    -pdf             Outputs a graph in PDF format
    -peek            Output callers/callees of functions matching regexp
    -png             Outputs a graph image in PNG format
    -proto           Outputs the profile in compressed protobuf format
    -ps              Outputs a graph in PS format
    -raw             Outputs a text representation of the raw profile
    -svg             Outputs a graph in SVG format
    -tags            Outputs all tags in the profile
    -text            Outputs top entries in text form
    -top             Outputs top entries in text form
    -topproto        Outputs top entries in compressed protobuf format
    -traces          Outputs all profile samples in text form
    -tree            Outputs a text rendering of call graph
    -web             Visualize graph through web browser
    -weblist         Display annotated source in a web browser

  Options:
    -call_tree       Create a context-sensitive call tree
    -compact_labels  Show minimal headers
    -divide_by       Ratio to divide all samples before visualization
    -drop_negative   Ignore negative differences
    -edgefraction    Hide edges below <f>*total
    -focus           Restricts to samples going through a node matching regexp
    -hide            Skips nodes matching regexp
    -ignore          Skips paths going through any nodes matching regexp
    -mean            Average sample value over first value (count)
    -nodecount       Max number of nodes to show
    -nodefraction    Hide nodes below <f>*total
    -noinlines       Ignore inlines.
    -normalize       Scales profile based on the base profile.
    -output          Output filename for file-based outputs
    -prune_from      Drops any functions below the matched frame.
    -relative_percentages Show percentages relative to focused subgraph
    -sample_index    Sample value to report (0-based index or name)
    -show            Only show nodes matching regexp
    -show_from       Drops functions above the highest matched frame.
    -source_path     Search path for source files
    -tagfocus        Restricts to samples with tags in range or matched by regexp
    -taghide         Skip tags matching this regexp
    -tagignore       Discard samples with tags in range or matched by regexp
    -tagshow         Only consider tags matching this regexp
    -trim            Honor nodefraction/edgefraction/nodecount defaults
    -trim_path       Path to trim from source paths before search
    -unit            Measurement units to display

  Option groups (only set one per group):
    cumulative       
      -cum             Sort entries based on cumulative weight
      -flat            Sort entries based on own weight
    granularity      
      -addresses       Aggregate at the address level.
      -filefunctions   Aggregate at the function level.
      -files           Aggregate at the file level.
      -functions       Aggregate at the function level.
      -lines           Aggregate at the source code line level.

  Source options:
    -seconds              Duration for time-based profile collection
    -timeout              Timeout in seconds for profile collection
    -buildid              Override build id for main binary
    -add_comment          Free-form annotation to add to the profile
                          Displayed on some reports or with pprof -comments
    -diff_base source     Source of base profile for comparison
    -base source          Source of base profile for profile subtraction
    profile.pb.gz         Profile in compressed protobuf format
    legacy_profile        Profile in legacy pprof format
    http://host/profile   URL for profile handler to retrieve
    -symbolize=           Controls source of symbol information
      none                  Do not attempt symbolization
      local                 Examine only local binaries
      fastlocal             Only get function names from local binaries
      remote                Do not examine local binaries
      force                 Force re-symbolization
    Binary                  Local path or build id of binary for symbolization
    -tls_cert             TLS client certificate file for fetching profile and symbols
    -tls_key              TLS private key file for fetching profile and symbols
    -tls_ca               TLS CA certs file for fetching profile and symbols

  Misc options:
   -http              Provide web interface at host:port.
                      Host is optional and 'localhost' by default.
                      Port is optional and a randomly available port by default.
   -no_browser        Skip opening a browser for the interactive web UI.
   -tools             Search path for object tools

  Legacy convenience options:
   -inuse_space           Same as -sample_index=inuse_space
   -inuse_objects         Same as -sample_index=inuse_objects
   -alloc_space           Same as -sample_index=alloc_space
   -alloc_objects         Same as -sample_index=alloc_objects
   -total_delay           Same as -sample_index=delay
   -contentions           Same as -sample_index=contentions
   -mean_delay            Same as -mean -sample_index=delay

  Environment Variables:
   PPROF_TMPDIR       Location for saved profiles (default $HOME/pprof)
   PPROF_TOOLS        Search path for object-level tools
   PPROF_BINARY_PATH  Search path for local binary files
                      default: $HOME/pprof/binaries
                      searches $name, $path, $buildid/$name, $path/$buildid
   * On Windows, %USERPROFILE% is used instead of $HOME
no profile source specified
```

## 安装pprof

go tool pprof 来源于 google/pprof 项目，可以以如下方式安装。

```sh
go get -u github.com/google/pprof
```

## 安装Graphviz

如果使用  -http 选项指定需要web交互页面，则需要安装 dot 。

```sh
Failed to execute dot. Is Graphviz installed?
exec: "dot": executable file not found in $PATH
```

ubuntu 上通过以下方式安装：

```sh
sudo apt install graphviz
```

## 启用功能

需要我们的程序开放了pprof web端点。一般建议的方式为,在需要使用的地方引用`net/http/pprof`包。

```go
import _ "net/http/pprof"
```

该方式会在默认的 http.DefaultServeMux 中插入debug pprof端点。

```go
// pprof.go:79
func init() {
	http.HandleFunc("/debug/pprof/", Index)
	http.HandleFunc("/debug/pprof/cmdline", Cmdline)
	http.HandleFunc("/debug/pprof/profile", Profile)
	http.HandleFunc("/debug/pprof/symbol", Symbol)
	http.HandleFunc("/debug/pprof/trace", Trace)
}
```

不过在一般的开发中不使用该方式，而是使用自定义的handler，如下。

```go
	m := http.NewServeMux()
	m.Handle("/debug/vars", expvar.Handler()) //用于查看exvar包中的存储的数据，由于一般无人使用该包，所以意义不大。
	m.HandleFunc("/debug/pprof/", pprof.Index))
	m.HandleFunc("/debug/pprof/cmdline", pprof.Cmdline)
	m.HandleFunc("/debug/pprof/profile", pprof.Profile)
	m.HandleFunc("/debug/pprof/symbol", pprof.Symbol)
	m.HandleFunc("/debug/pprof/trace", pprof.Trace)
```

pprof包内调用`runtime`包中函数以获取各种运行时信息，其包含如下分析指标。

allocs: 过去所有内存分配的样本
block: 导致同步原语阻塞的堆栈跟踪
cmdline:  当前程序的命令行调用，与/proc/中的 cmdline相同
goroutine: 当前所有goroutine的堆栈跟踪
heap: A活动对象的内存分配的采样。您可以指定gc GET参数以在获取堆样本之前运行GC。
mutex: 竞争互斥体持有人的堆栈痕迹
profile: CPU配置文件。您可以在秒GET参数中指定持续时间。获取概要文件后，使用go tool pprof命令调查概要文件。
threadcreate: 导致创建新OS线程的堆栈跟踪
trace: 当前程序执行的痕迹。您可以在秒GET参数中指定持续时间。获取跟踪文件后，请使用go工具trace命令调查跟踪。


## 分析

### CPU

```sh
# 执行CPU性能分析，会默认从当前开始执行30scpu采样。
$ go tool pprof http://localhost:6060/debug/pprof/profile
# 也可以通过参数 seconds 指定采样时间。
$ go tool pprof http://localhost:6060/debug/pprof/profile\?seconds=60
# 采样结束后默认进入命令行界面
Fetching profile over HTTP from http://localhost:6060/debug/pprof/profile
Saved profile in /home/user/pprof/pprof.-.samples.cpu.001.pb.gz
File: -
Type: cpu
Time: Dec 24, 2020 at 6:24pm (CST)
Duration: 30.06s, Total samples = 590ms ( 1.96%)
Entering interactive mode (type "help" for commands, "o" for options)
(pprof) 
(pprof) top
Showing nodes accounting for 250ms, 42.37% of 590ms total
Showing top 10 nodes out of 285
      flat  flat%   sum%        cum   cum%
      50ms  8.47%  8.47%       50ms  8.47%  runtime.futex
      30ms  5.08% 13.56%      100ms 16.95%  runtime.findrunnable
      30ms  5.08% 18.64%       30ms  5.08%  syscall.Syscall
      20ms  3.39% 22.03%       20ms  3.39%  aeshashbody
      20ms  3.39% 25.42%       20ms  3.39%  encoding/json.(*decodeState).rescanLiteral
      20ms  3.39% 28.81%       30ms  5.08%  encoding/json.checkValid
      20ms  3.39% 32.20%       20ms  3.39%  runtime.epollwait
      20ms  3.39% 35.59%       30ms  5.08%  runtime.gentraceback
      20ms  3.39% 38.98%       20ms  3.39%  runtime.memclrNoHeapPointers
      20ms  3.39% 42.37%       20ms  3.39%  runtime.memmove
```

flat：函数上运行耗时 
flat%：函数上运行耗时 总比例 
sum%：函数累积使用 CPU 时间比例
cum：函数及之上的调用运行总耗时 
cum%：函数及之上的调用运行总耗时比例



### 内存

更方便的场景为使用web的交互页面代替命令行页面。与cpu性能分析相同，进行内存占用分析。

```sh
# 在当前机器 8080 端口运行pprof heap分析。
go tool pprof -http :8080 http://localhost:6060/debug/pprof/heap
```