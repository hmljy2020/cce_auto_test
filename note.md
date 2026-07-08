## CCE Instruction 自动性能采集框架计划初稿
项目构成：python脚本，结构化且指向清晰的markdown文件丛，c++代码模板，SQlite数据库
功能描述：在该项目下启动Agent，并告诉Agent需要micro-benchmark的cce指令，Agent会开始长时间micro-benchmark，自动提出测试参数范围，分批micro-benchmark，建立和维护cce指令的性能`数据库`。
目标：建立一个可持续更新的cce指令性能数据库，在日积月累的自动化测试中，持续覆盖大量参数范围。

### cce指令文档
Agent维护一个指导文档，文档储存着对cce指令的`理解`，包含两部分：
1. 一个极其精简的metadata文件描述cce指令的函数形式，与`数据库`格式类似
2. 一个.md文件描述更详细的入参含义，注意事项，参数限制 (TO DO, 2026.7.6)

文档内容的注意事项：
- 同名指令可能存在多个重载，每个实现赋予一个签名（signature）
- 涉及GM的搬运指令涉及L2 cache control的问题，在2201及以前的架构上没有cce入参的开关，而是靠虚拟地址映射完成，这时metadata和.md文件需要增加这个额外参数的语义，测试模板也需要特例化。
- 复杂运算（如vdiv，vexp）的耗时可能会受到`set_mask`的影响，需要在metadata和.md文件需要增加这个额外参数的语义，测试模板也需要特例化。
- 其他在后续实验过程中发现的其他额外影响，需要以谨慎且合适的方式,在和用户确认后，补充到`理解`中。

该指令文档是动态更新的，当Agent收到`micro-benchmark某个cce指令名称`的命令时：
1. 查阅`cce_reference`下的内容，`理解`该cce指令，如果存在重载，向用户确认需要micro-benchmark哪个实现，给出推荐，确认signature。
2. 检查`cce指令文档`中是否已经`理解`该signature的cce指令，如果
    1. 已`理解`：参考`cce指令文档`，提出micro-benchmark参数范围，生成测试代码，储存测试结果到数据库
    2. 未`理解`：新增`理解`，严谨地向用户确认理解内容后，添加到`cce指令文档`中，跳转到`已理解`

`cce_reference` path: /home/chao/microbenchmark/cce_reference

### 测试模板代码
#### kernel
基于GetSystemCycle的cce指令profiling基本结构为：
```cpp
template<Type srcType, Type dstType, ...>
__global__ __aicore__ void benchmark(parameters...){
    // Some necessary preparation for cce parameters
    // Like addr offset for L2 bypass on arch 2201
    // Or set_mask, etc...

    // Warm up
    <cce>(parameters...);

    // Iteration
    pipe_barrier(PIPE_ALL);
    int64_t start = AscendC::GetSystemCycle();
    for (i=0; i<ITER; i++){
        <cce>(parameters...);
    }
    pipe_barrier(PIPE_ALL);
    int64_t end = AscendC::GetSystemCycle();
    store_result(result, end - start);
}
```

对于每一族cce指令，参考这个共同简易模板，特化这一族cce指令的模板以满足参数覆盖需求，比如：
1. 涉及GM搬运的指令，需要考虑是否Bypass L2，在2201架构上需要在前期对GM地址加一个offset。

对于要测试的cce指令，micro-benchmark的参数的作为变量传到kernel启动函数中（比如burst、burst_len等），一次编译，运行时input不同参数。变量可以是cce的直接入参，也可以是指导参数准备（比如2201架构上L2 Bypass的虚拟地址映射等）。

*对于无法用变量直接控制的功能，生成特化模板代码

ITER作为循环测试次数，以变量形式传入，ITER参数的大小需要保证`result>100`，以减小误差。

#### host
```cpp
int main(){
    // Read input file, total N_case input parameters
    GetInput(benchmark_cases, '.../input.?');

    // micro-benchmark
    for (i=0; i<N_case; i++){
        benchmark_args = cases[i];
        benchmark<<<1, nullptr, stream>>>(src_device, dst_device, benchmark_args...);
    }
}
```
提前生成好批量的测试参数`input.?`，`input.?`里包含大量case，每个case包含按顺序排号的kernel侧输入，`input.?`的读取速度越快越好。

host侧读取`input.?`，反复拉起kernel测试。


### 数据库
使用SQlite储存数据，保持精简：
1. arch version，SoC version，cce指令名称，signature
2. addr
    - cce指令中的操作数地址src、dst
    - addr不储存src和dst的具体值，储存alignment（地址对齐粒度），cache_mode（涉及GM时，是否bypass L2）
3. cce指令控制参数
    - 直接按顺序储存成列表，后续可以对照`cce指令文档`按顺序确认参数含义
4. 额外控制参数
    - L2 Bypass等

数据储存的形式要尽可能简单

测试范围集合可以抽象成若干高维超立方体，重叠的立方体如何描述

### 工作流
1. Agent询问用户需要测试的cce指令名称`cce_name`
2. `理解`该`cce_name`，确认测试的具体实现`cce_instrucion`
3. 参考`cce指令文档中`中该`cce_instrucion`的相关内容，确认所有的控制参数形式，生成测试模板代码，编译出可执行文件

loop：
1. Agent自主提出测试参数范围，利用python脚本生成批量参数输入`input.?`
2. 运行测试
3. 将测试结果合并到数据库
4. 跳到1

先确认metadata和数据库的准确格式

---

## 已确认：metadata格式

`cce_docs`下每个cce指令维护一个metadata JSON文件。metadata只保存人工确认所需的最小信息：指令名、metadata版本、重载列表。

示例：
```json
{
  "instruction": "copy_gm_to_ubuf",
  "metadata_version": 1,
  "overloads": [
    {
      "status": "approved_sweep",
      "signature": "void copy_gm_to_ubuf(__ubuf__ void* dst, __gm__ void* src, uint8_t sid, uint16_t nBurst, uint16_t lenBurst, uint16_t srcStride, uint16_t dstStride);"
    }
  ]
}
```

多个重载时，每个重载写入`overloads`中的一个元素。

`status`只保留三种：
- `draft`：已记录但未确认，不允许smoke或正式测试。
- `approved_smoke`：允许临时smoke，不允许正式入库。
- `approved_sweep`：允许正式micro-benchmark并写入数据库。

`signature`是canonical signature，由脚本从`stub_fun.h`原始声明解析并规范化得到。要求单行、结尾带`;`、等价于选中的`stub_fun.h`重载。DB写入时逐字复制metadata中的`signature`，不再二次格式化。

不在metadata中手写`signature_id`、参数列表、hash、参数范围、baseline、模板选择或coverage策略。这些信息由脚本解析，或放入更详细的理解文档和后续测试计划。

## 已确认：数据库格式

SQLite第一版只保留一个核心表`measurements`。一行表示一个唯一case的当前成功测试结果；同一个case重复测试时，直接覆盖非unique结果字段。

```sql
CREATE TABLE measurements (
  id INTEGER PRIMARY KEY AUTOINCREMENT,

  arch_version TEXT NOT NULL,
  soc_version TEXT NOT NULL,
  cce_name TEXT NOT NULL,
  signature TEXT NOT NULL,

  cce_args_json TEXT NOT NULL,
  extra_params_json TEXT NOT NULL,

  template_id TEXT NOT NULL,
  time_ns REAL NOT NULL,
  updated_at TEXT NOT NULL,

  UNIQUE (
    arch_version,
    soc_version,
    cce_name,
    signature,
    cce_args_json,
    extra_params_json
  )
);
```

唯一case由以下字段定义：
- `arch_version`
- `soc_version`
- `cce_name`
- `signature`
- `cce_args_json`
- `extra_params_json`

非unique字段：
- `template_id`
- `time_ns`
- `updated_at`

`cce_name`必须等于metadata中的`instruction`。`signature`必须逐字等于metadata中某个`overloads[].signature`。

`cce_args_json`保存完整cce函数入参，严格按照signature中的入参顺序。地址参数也占一个位置，但只保存alignment，不保存真实地址、space、offset或overlap。

示例：
```json
[{"alignment":32},{"alignment":512},0,1,32,0,0]
```

地址参数对象目前只允许：
```json
{"alignment":32}
```

`extra_params_json`保存不是cce直接入参、但会影响性能语义的额外控制参数。不同指令可以不同。当前约定：
- `l2_mode`：只允许`"use_l2"`或`"bypass_l2"`；不涉及时省略。
- `set_mask`：整数；不涉及时省略。

所有JSON字段写入SQLite前必须canonicalize：
- compact JSON，不写多余空格。
- object key按字典序排序。
- list保持语义顺序。
- 空对象写`{}`，空列表写`[]`。
- 数字保持数字，不写成字符串。

`time_ns`表示单次cce指令平均耗时，单位ns，保留一位小数。使用`GetSystemCycle`时：
```text
time_ns = round((end_cycle - start_cycle) * 20.0 / iter, 1)
```

`iter`只参与计算，不入库。

第一版不保存：
- `device_id`
- `cann_version`
- `iter`
- 失败case
- start/end cycle、total cycle、per-iteration cycle
- schema version / migration表
- 额外查询索引
