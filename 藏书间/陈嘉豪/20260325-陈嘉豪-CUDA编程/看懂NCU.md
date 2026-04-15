# NCU（Nsight Compute Utils）

是官方提供的评估(profile)kernel的工具，有官方的图形化界面，也有命令行模式(Nsight Compute CLI)，

官方文档 [官方文档](https://docs.nvidia.com/nsight-compute/NsightCompute/index.html#profiler-report)


```
ncu --set full -o <report_name> <app> [app arguments] 
```
- 如果想要查看PTX SASS SOURCE相互对应的代码，需要在nvcc编译阶段加上参数 `-lineinfo`
- `--set <extend>` extend决定了report的细节程度， 从高到低：`full`, `detailed`，后面的不记得了
- `--kernel-name <kernel_name>` 只看某个kernel 
- `--metrics <metric_names>` 只看某些指标
- `--section <section_name>` 只看某个板块
- `--force-overwrite(-f)` 强制覆盖同名report 
- `--launch-skip <M> --launch-count <N>` 用于忽略掉前M次Warm-up,计数后面N次







# PPT展示纲要

### 1. Throughput
全局来看kernel效果
### 2. Memory Wrokload analysis
看这个kernel运行过程中哪些线路发生了数据传输 
### 3. warp state statistics
看运行过程中的warp状态，是在被运行还是被阻塞，在被什么阻塞
### 4. occupyancy 
占用率计算 $occupancy = \frac{\text{active threads on each SM}}{\text{max threads per SM}}$，反映kernel的并行能力，有时也不是越高越好，不需要太高

展示 occupyancy operator

<!-- ### 1. compute workload analysis
看大部分的计算都是什么类型的计算 -->









# 记录一次GEMM sharedMem优化

## 1. Throughput——直观效果
    
    Compute (SM) Throughput [%]	51.52
    Memory Throughput [%]	49.14
计算吞吐低则为计算受限，内存吞吐低则为（内存）带宽受限

两者都低，牛逼

经Gemini老师指点发现是延迟高，（但我不太信）即转到

## 针对 warp state statistics

- #### LG Throttle,Top 1,
本地/全局访存单元拥堵。发出的访存指令太多，硬件队列塞满了。,元凶。由于你用了 float4，单次指令搬运数据量大，加上 Shared Memory 的频繁读写，导致访存流水线（LSU）处理不过来。
- #### Wait,Top 2,
执行资源等待。通常是因为 __syncthreads() 或等待特定指令完成。,关键点。在 Tile 方案中，Block 必须等所有线程把数据搬进 Shared Memory。如果有人慢（比如下面提到的冲突），所有人都在这儿等。
- #### Selected,Top 3
已被选中。这不是一种“坏”停顿，它表示 Warp 已经准备好执行。,如果它排前三，说明你的 Warp 其实很活跃，只是受限于上述两个原因无法更进一步。
- #### MIO Throttle,辅助,
存储指令发射器拥堵。类似于 LG Throttle。,
进一步证实了你的计算量和搬运量配比不平衡，指令发得太猛。
- #### Not Selected,辅助,
调度竞争失败。
由于 SM 内部有多个 Warp 竞争同一个调度器。,说明你的 Occupancy（占用率）还可以，有很多 Warp 在竞争资源，这是好事。
- #### Math Pipe Throttle,较低,
计算流水线拥堵。计算单元太忙。
这说明计算不是你的瓶颈。
- #### Barrier / Short Scoreboard,较低
同步栅栏/共享内存读写等待。
这说明你的逻辑同步和简单的 Shared Memory 依赖处理得还行。