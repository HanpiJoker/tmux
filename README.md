# 驱动 mperf 使用指南(仅限公司内部使用)

## mperf 简要介绍

mperf 是驱动组内部开发的一个设备内存相关的 debug 工具，集成了 SMMU / LLC 的 DFX 工具和性能分析工具；
以及驱动的设备内存管理模块的部分 DFX 工具；

目前已经支持的特性有：

1. smmu:
    - 批量读写相同 offset 的 smmu 寄存器；
    - 获取当前平台支持的所有 smmu 的寄存器基址；
    - 获取指定 smmu 的 inputQueue 的状态信息；
    - 输出指定 smmu 的所有寄存器的值；
    - 获取指定 smmu 的 TLB / PTW / L2L3 cache 的数据信息；(仅限 L1D1 以及后续平台)
    - 配置指定 smmu 的指定 stream 使能 TRAP 模式；

2. llc:
    - 批量读写相同 offset 的 llc 寄存器；
    - 获取当前平台支持的所有 llc 的寄存器基址；
    - 输出指定 llc 的所有寄存器的值；

3. record:
    - 指定配置和读取 SMMU / LLC 的 PMU 寄存器；(一个上下文只支持一个硬件)
    - 获取当前 SMMU / LLC 支持的可追踪的事件类型；
    - 支持自动按照指定的时间周期刷新 PMU 计数器；
    - 支持手动在需要的时候输出 PMU 计数器结果；
    - 单次可追踪的事件上限是 8个；

4. 其他：
    - 支持获取指定 IOVA 的页表信息；
    - 支持获取当前的全部的页表信息；
    - 支持重定向日志输出文件；

mperf 目前支持的平台有 mlu500s 以及后续平台；

## mperf 依赖与限制

### 驱动版本依赖

\>= driver_v5.10.30

### 限制

- mperf 目前仅支持在物理机上运行，MIM 场景下创建的 docker 需要有 root 权限，并且透传 sysfs；

- mperf 由于需要访问 sysfs，如果用户没有 sudo 权限。无法支持 SMMU / LLC 的部分功能；

- 当前版本的 mperf 仅支持物理卡环境，如果需要在 MIM 场景下使用。需要联系驱动组同事编译一个依赖 libcndrv.so 的版本；

## mperf 安装

```shell
git clone http://gitlab.software.cambricon.com/yujiahao/mperf.git
```

## mperf 使用

### mperf 基础功能介绍

```shell
# ./mperf help

 Usage: mperf -- Internal memory software and hardware perf and debug tools


 Commands:

 llc       Tools to handle llc pmu and register 
 smmu      Tools to handle smmu pmu and register 
 record    Tools to record pmu events


 Options:

 -h/--help                Show help information
 -v/--version             Show current version
 --list-registers         list-registers
 --iova                   Show iova pte in arm log [ --iova <addresss> <local/remote> ]
 --iopgtable              Show page table in arm log [ --iopgtable <local/remote> ]
 -o/--output              Specific log output file path [--output <filepath> ]
 -c/--card                Specific mperf running on which mlu device [--card <num>]


 See 'mperf help COMMAND' for more information on a specific command.

```

mperf 的参数分为两部分： 子命令 和 选项；每个子命令又有自己的选项和子命令。特殊的是 `record`，它只
能作为 `smmu` 和 `llc` 的子命令出现。mperf 的参数和命令识别是从左往后依次识别的，因此 mperf 自己的
参数一定要在子命令之前进行配置。否则会被识别为子命令的参数。

- 执行 `mperf help` 获取 mperf 工具的参数介绍日志
- `-c <dec num>` 指定需要访问 mlu device 的 ID，如果没有指定默认访问 card0；
    > - 当前的 mperf 不识别 CN_VISIBLE_DEVICES 的环境变量，指定的 Card Id 是 /dev/cambricon_devX 的ID；
    > - 所有功能都依赖首先指定 device ID，如果没有指定设备 ID。后续命令无法继续执行；

- `-o <file path>` 指定日志重定向的文件名，如果文件存在则会追加写入；文件不存在则会自动新建文件；
    > - 并不是 mperf 所有的输出日志都会重定向到文件中；
    > - 目前支持重定向的日志如下：
    >   - SMMU / LLC 的 PMU 的计数器的统计日志；
    >   - `--list-registers` 和 `--print-registers` 输出的日志；
    >   - 读写寄存器输出的日志；
    > - 重定向的文件中会有按照 CST 时区打印的时间戳；
    > 
    > 重定向文件日志示例
    > ![重定向文件的示例](./figures/output_redirect_demo.png)

- `--iova` 和 `--iopgtable` 这两个命令的日志输出需要通过串口或者是 procfs 的 mlumsg 查看；
    > 这两个命令的可选参数都是 `local` 和 `remote`：
    > - `local` : 表示需要获取指定设备本地的 smmu 的页表映射关系，硬件通过自己的 SMMU 进行页表翻译；
    > - `remote`: 表示需要获取指定设备建立的远端映射的页表映射关系，硬件通过 PCIE 的 Outbound SMMU
    >   进行页表翻译；
    >
    > 当前只有 L1D1 往后平台支持 `remote` 参数，在 mlu500s 平台上不解析 `local` 和 `remote`

### mperf smmu 命令介绍

```shell
# mperf help smmu

 Usage: mperf smmu [<options>] {record}

    -d, --dump-cache <cache>
                          Dump smmu cahe data, valie cache type <ptw/tlb/l2l3>
    -i, --print-iq        Dump smmu tcu iq status, add record sub command to record multi times
    -l, --list-registers  Show all smmu registers base
    -p, --print-registers
                          Print all smmu registers value
    -r, --read[=<n>]      Operation for read smmu register. use like --smmu <smmu> --read <hex>
    -s, --smmu <smmu>     Smmu indexed by <type,group,index> for other operation (default all)
    -t, --trap <enable/disable>
                          Enable or Disable smmu trap mode. use like --smmu <smmu> --trap <enable/disable> <stream>
    -w, --write[=<n>]     Operation for write smmu register. use like --smmu <smmu> --write <address> <value>

```

#### -l / --list-registers
第一次使用工具的时候可以通过 `mperf -c 0 smmu -l` 来获取当前平台支持的所有 smmu 的 ID 和寄存器基址；

输出日志示例图如下：
![smmu_list_registers_demo](./figures/smmu_list_registers_demo.png)

smmu 的 ID 分为 3 部分组成：
- 类型：ID 中的第一列的字符串，指定 smmu 所属的 master 种类(pcie，ipu，etc)
- group：ID 中的第二列的数字，指定 smmu 对应的 master 的group id。这个 ID 的含义由硬件和驱动共同定
  义。这里不再赘述；
- index：ID 中的第三列的数字，指定 smmu 对应的 master 的index id。这个 ID 的含义由硬件和驱动共同定
  义。这里不再赘述；

#### -s / --smmu
在使用 `smmu` 这个命令的时候，除了上面介绍的`-l / --list-registers` 以外，其他的参数或者子命令都必
须通过 `-s / --smmu` 来指定需要访问的 smmu。

- `-s / --smmu` 后面不跟参数或者跟 `all` 的时候，代表访问所有的 smmu；
- `-s / --smmu` 后面跟的 smmuId 通过 `,` 分割 type、group、index。也可能通过 `,` 分割多个smmu
    > - `-s pcie,0,0` 表示指定 pcie 的 group 0 和 index 0 的smmu；
    > - `-s ipu,1,0,ipu,2,0` 表示指定 ipu 的group 1 index 0 的smmu 以及 ipu 的group 2 index 0 的 smmu
- `-s / --smmu` 在指定 smmu ID 的时候 type 是必须的，group 和 index 均可以选择忽略，此时代表访问当前类型
  下的所有 smmu；
    > - `-s ipu` 表示访问 ipu 所有的smmu；
    > - `-s ipu,tinycore` 表示ipu 和 tinycore 所有的 smmu
    > - `-s ipu,tinycore,1,0` 表示ipu 所有的 smmu 以及 tinycore 的 group1 index0 的smmu
    > - `-s ipu,1` 表示ipu group1 下所有的 smmu
    > - `-s ipu,1,tinycore,2` 表示ipu group1 下所有的 smmu 以及 tinycore 的 group2 下所有的smmu
- `-s / --smmu` 在指定 smmu ID 的group 和 index 的时候可以传入 `-1` 表明全匹配 group 或 index；
    > - `-s ipu,-1,0` 表示访问 ipu 所有 index 是 0 的smmu, 全匹配 group；
    > - `-s ipu,0,-1` 表示访问 ipu group0 下所有的 smmu 等效于 `-s ipu,0`
- `-s / --smmu` 在指定 smmu ID 的group 和 index 的时候是顺序识别的，类型后面跟的数字一定是 group。
  不能跳过group 指定 index

#### -i / --print-iq
iq 是 smmu 硬件上的一组 DFX 的寄存器用于观察 TCU 的工作状态，这部分信息主要是在硬件需要的时候提供给
硬件进行分析；

输出日志示例图如下：
![smmu_iq_status_demo](./figures/smmu_iq_status_demo.png)

普通情况下 `-s <smmu> -i` 只会输出一次 iq 的状态值。如果有长时间检测 iq 状态的需求，可以追加 `record` 命令。
```shell
mperf -c 0 smmu -s pcie -i record
```
此时就可以通过 record 的框架实现自动或者手动记录多次 iq 的状态信息，从而分析其数据变化。

#### -p / --print-registers

print-registers 命令可以打印指定smmu 所有寄存器的值。

#### -t / --trap

trap 是 smmu 硬件提供的一种用于定位多个 master 之间发生数据踩踏的硬件机制。

实现原理如下：
通过在页表上以及smmu 的 cau config 寄存器中增加相应的标志位，当页表和指定 smmu 的 cau 的标志位均被
置位的前提下。后续此 smmu 通过指定 stream 访问指定虚拟地址的属性就会被强制改为只读。此时如果有写操
作就会触发 SMMU 的中断告警。

功能使用说明：
通过 `-s <smmu> -t enable <stream>` 使能 trap 功能之后，后续的所有内存申请请求创建的页表项都会置位
trap 的标志位。同时通过 `-s <smmu>` 指定的 smmu 的 cau 上的 trap 标志位也会被置位。此后如果被指定的
smmu 存在写地址的行为就会报错；也可以通过 `<stream>` 选择使能指定 smmu 的特定 stream，stream 传参要
求是 **十六进制的数**

> 当前一次修改所有内存申请请求的页表属性粒度不够细化。因此未来考虑可以支持指定 iova 配置单一地址使
> 能或关闭 trap 功能。

#### -d / --dump-cache

dump-cache 是在 L1D1 后续新增平台新增的硬件特性，支持获取指定 SMMU 的指定 cache 内的数据信息。当前
版本会将 cache 中的数据打印在 ARM 侧的串口中可以通过 procfs 的mlumsg 获取信息。

- `-d/--dump-cache all` 支持传入 `all` 表示 dump 当前 smmu 支持的所有 cache 的数据；
- `l2l3` 仅 ipu/tinycore 支持，其他类型 smmu 传此参数不会执行；

#### -r / --read / -w / --write

读写寄存器功能，相比通过 sysfs 或者是在 ARM侧通过 devmem 直接读写寄存器。优势在于不需要了解各个
master 的smmu 的寄存器机制。只需要考虑 SMMU 的寄存器内部的偏移

- 上述读写寄存器的参数中，地址和写入的数据都要求是 **十六机制数**。并且地址要求 4 字节对齐；
- 读写寄存器支持指定区间进行读取或写入，在传入地址的时候通过 `:` 连接区间的开始和结束；
    > - `-r 0x100:0x140` 表示读取从偏移 0x100 开始到 0x140 结束的所有寄存器的值；
    > - `-w 0x100:0x140 0x10` 表示从偏移 0x100 开始到 0x140 结束的所有寄存器均写入 0x10；

#### smmu record command

在后续介绍 record 命令的时候统一说明


### mperf llc 命令介绍

```shell
# mperf llc help

 Usage: mperf llc [<options>] {record}

    -l, --list-registers  Show all llc registers base
    -p, --print-registers
                          Print all llc registers value
    -r, --read[=<n>]      Operation for read llc register. use like --llc <llc> --read <hex>
    -s, --specific <specific>
                          Specific llc index for other options, without this option, will config all llc
    -w, --write[=<n>]     Operation for write llc register. use like --llc <llc> --write <address> <value>

```

llc 的命令的行为与 smmu 类似，个别命令存在差异

#### -s / --specific
LLC 的 ID 是一个 8 字节的数，所以 `-s/--specific` 后面跟的 ID 需要是 16进制的数，依旧可以通过 `,`
连接多个 ID。具体支持的 LLC ID 有哪些可以通过 `-l / --list-regsiters` 获取

- LLC 的 `-s / --specific` 不是必须的，默认情况下会操作所有的 LLC；

#### -r / -w / --read / --write

读写寄存器的行为是和 smmu 一致的，需要注意的是 LLC 在个别平台的硬件上，内部的寄存器排布会进一步划分
system 和 slice。在读写寄存器的时候需要用户自己添加 slice / system 的偏移；

### mperf record 命令介绍
```shell
# mperf help record

 Usage: mperf <smmu/llc> [<options>] record [<options>] [<command>]
    or: mperf <smmu/llc> [<options>] record [<options>] -- <command> [<options>]

    -c, --cycle <n>       Automatically flush print event counts time interval, unit: ms, default: 1000ms
    -e, --event <[event,event]/[event,index,event,index]>
                          event selector. use 'mperf smmu/llc record -e list' to list available events
    -m, --manual          Dump event counts while user enter
    -v, --verbose         be more verbose (show counter open errors, etc)

```

`record` 命令支持统计 SMMU / LLC event 的计数，因此 `record` 目前只支持作为 `smmu` 和 `llc` 的子命
令而存在；

- `-c / --cycle` 仅自动模式(没有指定 `-m / --manual`)下生效，指定自动模式下数据刷新的间隔时间，单位是 ms。默认值是 1000ms；
- `-m / --manual` 手动模式，输出一个 cli 交互界面，用户输入回车更新一次数据；
- 不论是手动模式还是自动模式，想要退出数据更新。可以通过 ctrl-c 或者输入 q 来退出数据更新；

- `-e list` 可以获得当前支持的所有事件类型；
- `-e <event>,<event>` 可以通过 `,` 来追加多个 event，最多支持同时追踪 8 个事件；

- 目前 record 暂不支持追踪指定的应用程序，所以不支持后续追加要追踪的应用程序及其参数。

SMMU 的事件有些特殊情况，SMMU 事件存在多个统计维度(stream,tbu,tcu)。在统计 stream 和 tbu 类型的事件
的时候可以在他event 后面追加 10 进制数来指定需要统计的 stream 或者是 tbu。如果不指定默认获取
stream0 / tbu0 的数据。

实例如下：
> `mperf -c 0 smmu -s pcie record -e smmu__stream_wr_access,0,smmu__stream_wr_access,2 -m`
> 上述命令的含义是通过手动模式追踪 PCIE 下所有 smmu 的 stream 0 和 stream 2 的写请求的个数
>
> `mperf -c 0 smmu -s pcie,ipu record -e smmu__stream_wr_access,0,smmu__tbu_wr_access,0 -m`
> 上述命令的含义是通过手动模式追踪 PCIE 和 ipu 下所有 smmu 的 stream 0 和 tbu0 的写请求的个数

