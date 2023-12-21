# VClinic 使用示例

## 示例准备：字节零统计工具

准备字节零统计工具源码文件夹：
```
cd VClinic && mkdir -p src/clients/zerobyte
```
参考已有工具的CMake配置文件进行名称上的修改：
```
cp src/clients/zerospy/CMakeLists.txt src/clients/zerobyte/
sed -i “s/zerospy/zerobyte” CMakeLists.txt
```

## zerobyte字节零统计工具实现

创建并实现字节零统计工具：`vim src/clients/zerobyte/zerobyte.cpp`
1. 首先添加相关头文件以及重要变量声明定义：
```
#include <unordered_map>
#include "vprofile.h"
using namespace std;

vtrace_t* vtrace;
uint64_t grandTotBytesLoad = 0;
uint64_t grandTotBytesRedLoad = 0;
```
2. 实现每个值跟踪记录的处理函数
```
void trace_update_cb(val_info_t *info) {
    uint8_t *val = (uint8_t*)info->val;
    int size = info->size;
    uint64_t red=0;
    for(int i=0; i<size; ++i) {
        red += (val[i]==0)?1:0;
    }
    __sync_fetch_and_add(&grandTotBytesRedLoad,red);
    __sync_fetch_and_add(&grandTotBytesLoad,size);
}
```
3. 实现工具加载和结束时的处理函数
```
static void
ClientInit(int argc, const char *argv[]) {}

static void
ClientExit(void)
{
    dr_fprintf(STDOUT, "\n#Redundant Read:");
    dr_fprintf(STDOUT, "\nTotalBytesLoad: %lu \n",grandTotBytesLoad);
    dr_fprintf(STDOUT, "\nRedundantBytesLoad: %lu %.2f\n",grandTotBytesRedLoad, grandTotBytesRedLoad * 100.0/grandTotBytesLoad);
    vprofile_unregister_trace(vtrace);
    vprofile_exit();
}
```
4. 操作数以及指令过滤器实现
```
// We only interest in memory loads
bool
VPROFILE_FILTER_OPND(opnd_t opnd, vprofile_src_t opmask) {
    uint32_t user_mask = ANY_DATA_TYPE | MEMORY | READ | BEFORE;
    return ((user_mask & opmask) == opmask);
}

bool
filter_read_mem_access_instr(instr_t *instr)
{
    return instr_reads_memory(instr) && !instr_is_prefetch(instr);
}

#define FILTER_READ_MEM_ACCESS_INSTR filter_read_mem_access_instr
```
5. 工具主函数实现，包括注册退出回调函数、过滤器、值跟踪记录处理函数等
```
#ifdef __cplusplus
extern "C" {
#endif
DR_EXPORT void dr_client_main(client_id_t id, int argc, const char *argv[]) {
    dr_set_client_name("DynamoRIO Client 'zerobytes’”, "http://dynamorio.org/issues");
    ClientInit(argc, argv);
    dr_register_exit_event(ClientExit);
    vprofile_init(FILTER_READ_MEM_ACCESS_INSTR, NULL, NULL, NULL, VPROFILE_DEFAULT);
    vtrace = vprofile_allocate_trace(VPROFILE_TRACE_VALUE);
    uint32_t opnd_mask = ANY_DATA_TYPE | MEMORY | READ | BEFORE;
    vprofile_register_trace_cb(vtrace, VPROFILE_FILTER_OPND, opnd_mask, ANY, trace_update_cb);
}
#ifdef __cplusplus
}
#endif
```

## 编译安装zerobyte

**总共实现行数**：72行，无任何插桩细节实现

重新编译VClinic来生成新实现的zerobyte工具：
```
./build.sh
```
编译生成的zerobyte工具可以用类似zerospy、trivialspy的方式使用：
```
$DRRUN -t zerobyte -- <EXE> <ARGS>
```
例如使用zerobyte来检测backprop程序中的字节零比例：
```
# time $DRRUN -t zerobyte -- /backprop 65536
Random number generator seed: 7
Input layer size : 65536
Starting training kernel
Performing CPU computation
Training done

#Redundant Read:
TotalBytesLoad: 291240697

RedundantBytesLoad: 113784384 39.07
```



---

[Prev: TrivialSpy 使用示例](TrivialSpy.md)