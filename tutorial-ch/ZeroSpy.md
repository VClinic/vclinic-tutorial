# ZeroSpy使用示例

## 示例准备：backprop
源码获取与编译：机器学习/深度学习训练中的反向传播算法

拉取源码：
```
wget http://www.cs.virginia.edu/~skadron/lava/Rodinia/Packages/rodinia_3.1.tar.bz2
```
解压缩：
```
tar xvf rodinia_3.1.tar.bz2
```
进入源码目录并编译：
```
cd rodinia_3.1/openmp/backprop && make
```
记录原始程序执行时间：
```
time ./backprop 6553600
```

## 分析

使用小规模输入对程序执行过程进行分析：
- ZeroSpy: `$DRRUN –t zerospy -- ./backprop 65536`
- 分析结果文件为`x86-<hostname>-<PID>-zerospy`文件夹中：
- 其中，`zerospy.log`为冗余零的统计摘要，`thread-<id>.topn.log`为各个线程的冗余零详细报告，部分报告内容如下：

`zerospy.log`:
```
#THREAD 1 Redundant Read:TotalBytesLoad: 8318744
RedundantBytesLoad: 2314390 27.82
ApproxRedundantBytesLoad: 1047288 12.59

#THREAD 2 Redundant Read:TotalBytesLoad: 8400644
RedundantBytesLoad: 2379120 28.32
ApproxRedundantBytesLoad: 1047288 12.47

…
```

`thread-<id>.topn.log`
```
--------------------  Dumping Approximation Redundancy Info -----------------
************** Dump Data(delta=1.00%) from Thread 1 ************
 Total redundant bytes = 12 589497 %

 INFO : Total redundant bytes = 12.589497 % (1047288 / 8318744)

=== (12.499809) % of total Redundant, with local redundant 100.0 % (130909 Zeros / 130909 Reads) ===

=== Redundant byte map : [ sign | exponent | mantissa ]
xx | 00 | 00 00 00
=== [ AccessLen=4, typesize-4] ===
---------- Redundant load with ----------
#0 0x00007fdb0e9525f0 "cvtss2sd xmm.dword ptr [rdi]" in bpnn_adjust_weights._omp_fn.0 at [backprop.c:323]
#1 0x00007fdb0f67e86b “call r12” in <MISSING> at [backprop.c:0]
#2 0x00007fdb0f61c603 “call qword ptr [rax+0x00000640]” in start_thread at [backprop.c:0]
#3 0x00007fdb0f523351 “call rax” in clone at [backprop.c:0]
#4 0xFFFFFFFFFFFFFFFE “<NULL>” in THREAD[1]_ROOT_CTXT at [<NULL>:0]
#5 0x000000000000000 “<NULL>” in PROCESS[389]_ROOT_CTXT at [<NULL>:0]

=== (12.499809) % of total Redundant, with local redundant 100.0 % (130909 Zeros / 130909 Reads) ===
….
```

## 调优

指导调优：冗余零聚集于`backprop.c:323`，存在大量完全冗余零（`exp`, `man`为0）


```
=== (12.499809) % of total Redundant, with local redundant 100.0 % (130909 Zeros / 130909 Reads) ===

=== Redundant byte map : [ sign | exponent | mantissa ]
xx | 00 | 00 00 00
=== [ AccessLen=4, typesize-4] ===
---------- Redundant load with ----------
#0 0x00007fdb0e9525f0 "cvtss2sd xmm.dword ptr [rdi]" in bpnn_adjust_weights._omp_fn.0 at [backprop.c:323]
#1 0x00007fdb0f67e86b “call r12” in <MISSING> at [backprop.c:0]
#2 0x00007fdb0f61c603 “call qword ptr [rax+0x00000640]” in start_thread at [backprop.c:0]
#3 0x00007fdb0f523351 “call rax” in clone at [backprop.c:0]
#4 0xFFFFFFFFFFFFFFFE “<NULL>” in THREAD[1]_ROOT_CTXT at [<NULL>:0]
#5 0x000000000000000 “<NULL>” in PROCESS[389]_ROOT_CTXT at [<NULL>:0]
```

进一步分析后可知`delta`、`oldw`值为0:
```
#ifdef OPEN
  omp_set_num_threads(NUM_THREAD);
  #pragma omp parallel for  \
      shared(oldw, w, delta) \
          private(j, k, new_dw) \
          firstprivate(ndelta, nly) 
#endif
  for (j = 1; j <= ndelta; j++) {
    for (k = 0; k <= nly; k++) {
      new_dw = ((ETA * delta[j] * ly[k]) + (MOMENTUM * oldw[k][j]));
          w[k][j] += new_dw;
          oldw[k][j] = new_dw;
    }
  }
```

调优后代码为：
```
#ifdef OPEN
  omp_set_num_threads(NUM_THREAD);
  #pragma omp parallel for  \
      shared(oldw, w, delta) \
          private(j, k, new_dw) \
          firstprivate(ndelta, nly) 
#endif
  for (j = 1; j <= ndelta; j++) {
    if(delta[j]==0) {
      for (k = 0; k <= nly; k++) {
        if(oldw[k][j]!=0) {
            new_dw = ((ETA * delta[j] * ly[k]) + (MOMENTUM * oldw[k][j]));
            w[k][j] += new_dw;
            oldw[k][j] = new_dw;
        }
      }
    } else {
      for (k = 0; k <= nly; k++) {
        new_dw = ((ETA * delta[j] * ly[k]) + (MOMENTUM * oldw[k][j]));
            w[k][j] += new_dw;
            oldw[k][j] = new_dw;
      }
    }
  }
```

**调优后性能提升`1.21x`**。

---

[Prev: 安装](Build.md)

[Next: TrivialSpy使用示例](TrivialSpy.md)