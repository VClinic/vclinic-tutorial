# 安装

## 源码安装：

依赖安装：
- git
- gcc
- g++
- make
- cmake>=3.20

拉取源码：
```
git clone --recursive https://github.com/VClinic/VClinic.git
```

进入源码目录并编译：
```
cd VClinic && ./build.sh
```

配置可执行目录：
```
export DRRUN=\`pwd`/build/bin64/drrun
```

安装后，可用vclinic内置的值分析工具对目标程序执行过程进行分析：

ZeroSpy:
```
$DRRUN –t zerospy -- <EXE> <ARGS>
```

TrivialSpy:  
```
$DRRUN –t trivialspy -- <EXE> <ARGS>
```

## 预装版本：
提供了VClinic的docker镜像（包含X86和ARM），可在[zenodo](https://doi.org/10.5281/zenodo.7311322)上下载

---

[Prev: 工具集实现架构](Overview.md)

[Next: ZeroSpy 使用示例](ZeroSpy.md)