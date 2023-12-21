# TrivialSpy 使用示例

## 示例准备：NPB IS

源码获取与编译：NPB基准测试程序集中的整数并行排序程序
- 拉取源码：`wget https://www.nas.nasa.gov/assets/npb/NPB3.4.2.tar.gz`
- 解压缩：`tar xvzf NPB3.4.2.tar.gz`
- 进入源码目录并编译：
```
cd NPB3.4.2/NPB3.4-OMP/
cp config/make.def.template config/make.def
vim config/make.def # 所有FFLAG、CFLAG都增加-g
make IS CLASS=C
```

## 分析

运行初始版本并计时：
```
time ./bin/is.C.x
```
使用TrivialSpy工具分析IS程序执行：
```
$DRRUN -t trivialspy -- ./bin/is.C.x
```

产生的结果文件在`x86-<host>-<PID>-trivialspy`文件夹中，其中trivialspy.log为无效操作检测指标的摘要报告，`thread-<id>.log`为各个线程的无效操作检测报告，其中部分摘要报告如下：
```
…
=== Overall Triviality Metric ===
Total Speculate Benifit: 1.624 (21464759128 benifit / 1321642318877 total cost)
Total Benifit: 1.564 (20669750937 benifit / 1321642318877 total cost)
Total Heavy Instruction: 0.000 (9176 / 21464759128 SB)
Total Trivial Chain: 0.000 (7327 / 21464759128 SB)
Total Redundant Backward Slice: 94.444 (20272256251 / 21464759128 SB)
Total Absorbing Breakpoints: 0.000 (16568 / 21464759128 SB)
```

其中，*IS具有1.624的期望收益，1.564的分支收益*

## 指导调优

具有最高性能优化潜力的无效操作代码块报告（含STC标注）如下：
```
------- Trivial Hotspots ordered by BB --------
Total Speculate Benifit: 1.617 (766602882 benifit / 47395586086 total cost)
Total Benifit: 1.558 (738209956 benifit / 47395586086 total cost)
Total Heavy Instruction: 0.000 (102 / 766602882 SB)
Total Trivial Chain: 0.000 (111 / 766602882 SB)
Total Redundant Backward Slice: 94.444 (724013597 / 766602882 SB)
Total Absorbing Breakpoints: 0.000 (354 / 766602882 SB)
===============================
Benifit: 25.011 (184635828 local benifit / 738209956 total benifit)
Importance: 0.390 (184635828 benifit / 47395586086 total cost)

Speculate Benifit: 25.011 (191737206 local SB / 766602882 SB)
Importance: 0.405 (191737206 benifit / 47395586086 total cost)

^^ Trivial Rate: 18.518 (3550689 / 19173780) ^^
+++ DFG Caller CCT Info +++
...
+++ DFG Summary Info +++
======= DFGLog from Thread 518 =======
...
++++++ Singular Trivial Condition(s):
  <59>  <38, 39>: opnd=xmm3, val=ZERO, isSingular=1
==> detailed node info:
  [0]   movsd  xmm3, qword ptr [fs:-0x00001410] @randlc[is.c:326]
  [1]   movsd  xmm2, qword ptr [fs:-0x00001420] @randlc[is.c:326]
  [2]   movsd  xmm0, qword ptr [fs:-0x00001418] @randlc[is.c:326]
  [3]   movsd  xmm1, qword ptr [fs:-0x00001428] @randlc[is.c:326] [D][B]
...
```

其中，标注的STC为以下数据流中无效计算的无效条件：
```
   ...
  [38]  cvtsi2sd xmm3, eax @randlc[is.c:368]
  [39]  mulsd  xmm1, xmm3 @randlc[is.c:369] [D]
  [40]  subsd  xmm2, xmm1 @randlc[is.c:369] [P]
  [41]  mulsd  xmm0, xmm2 @randlc[is.c:370]
  [42]  movsd  qword ptr [rdi], xmm2 @randlc[is.c:369]
  [43]  ret @randlc[is.c:371]
```

即：*无效计算聚集于`is.c:369`，其中xmm3为0值（T4），导致链式无效计算*
```
      j  = R23 * T1;
      T2 = j;
      Z = T1 - T23 * T2;
      T3 = T23 * Z + A2 * X2;
      j  = R46 * T3;
      T4 = j;
      *X = T3 - T46 * T4;
```

## 调优

消除浮点数转整数后形成的0值导致的无效计算传播。
```
      j  = R23 * T1;
      T2 = j;
      Z = T1 - T23 * T2;
      T3 = T23 * Z + A2 * X2;
      j  = R46 * T3;
      T4 = j;
      if(T4==0) {
              *X = T3;
              return R46*T3;
      }
      *X = T3 - T46 * T4;
      return(R46 * *X);
```

调优后，IS程序性能提升`1.05x`。


---

[Prev: ZeroSpy 使用示例](ZeroSpy.md)

[Next: VClinic 使用示例](VClinic.md)