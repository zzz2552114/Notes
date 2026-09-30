# 05 CacheLab 记录
## 01-cache
### 此作业是要手动实现一个cache缓存器，并统计命中数（hits）、不命中数（misses）和驱逐数（evictions）

***代码如下，可以拆成几个部分，附带注释***

### 初始状态
```c
#include "cachelab.h"

int main()
{
    // 下面这个函数是在上面的库中的，输出三者次数
    printSummary(0，0，0);
    return 0;
}
```

### 成品状态
1. **库初始化**
```c
// scanf 和 printf，malloc，FILE都是需要的
#include<stdio.h>
// 下面三个用来解析命令行参数
#include <getopt.h>
#include <stdlib.h>
#include <unistd.h>
// 题目给我封装好的库
#include "cachelab.h"

// 需要获取传参，用来解析命令行参数
int main(int argc,char* argv[])
```

2. **命令行解析**，详细参考 [getopt](https://github.com/zzz2552114/Notes/blob/main/C-%E5%91%BD%E4%BB%A4%E8%A1%8Copt%E8%A7%A3%E6%9E%90.md)
```c
int opt;
int v = 0,s,e,b;
char* filename;

// optstring格式：不带: -> 无参数，一个: -> 必须参数，两个:: -> 可选参数，可选参数必须紧靠着短 option
const char* optstring = "vs:E:b:t:";

// getopt函数可以解析命令行参数，格式固定如下。
// 返回类型是 int：短选项的 ascii 码
while((opt = getopt(argc,argv,optstring)) != -1){
    switch(opt){
        case 'v':
            v = 1;
            break;
        case 's':
            // optarg 是解析出来的内容，字符串
            s = atoi(optarg);
            break;
        case 'E':
            e = atoi(optarg);
            break;
        case 'b':
            b = atoi(optarg);
            break;
        case 't':
            filename = optarg;
            break;
        default:
            exit(1);
    }
}
```

3. **根据大小申请内存** ，详细参考 [malloc](https://github.com/zzz2552114/Notes/blob/main/C-malloc.md)
```c
// 缓存需要多少空间？地址64位，其中 T = 64-logS-logB 位
// 那我的 tag 就是T位，还有一个 valid 位。所以一共是 65-s-b，但是c语言里这种零散位数的不好实现呀 
// 所以索性tag变成 8 字节，valid变成 1 字节好了。
typedef struct {
    char val;  // 有效位
    long tag;  // 标签位
    int L;     // 驱逐位
} block;

// 缓存其实就是若干个块的数组，下面我们申请内存
block* cache = (block*)malloc((1<<s)*e*sizeof(block));

// 校验内存
if(cache == NULL){
    fprintf(stderr,"failed to malloc");
    exit(1);
}
// 这里内存不会帮我们初始化，有可能有垃圾值，里面的东西不能直接取值使用，要手动初始化一下
for (int i = 0; i < (1 << s) * e; i++)
{
    cache[i].val = 0;
}
```

4. **打开文件** ，详细参考 [fopen](https://github.com/zzz2552114/Notes/blob/main/C-FILE%E4%B8%8Efgets.md)
```c
FILE *fp = fopen(filename,"r");
if(fp == NULL){
    fprintf(stderr,"failed to open file");
    exit(1);
}
```

5. **逐行处理，核心缓存代码**
```c
// 初始化结果
int hit = 0,miss = 0,out = 0;
int cnt = 0;
char line[40];
while(fgets(line,40,fp) != NULL)
{
    // FILE 结构体会记录光标，fgets会往后推移光标
    char mode;
    unsigned long ad;
    int siz;
    sscanf(line," %c %lx,%d",&mode,&ad,&siz);
    // 现在获得了每一行的三个要素，在while循环里可以处理逻辑了
    if(mode == 'I') continue;
    
    // 根据 s,E,b 拆解 ad，拆成 setbit, tagbit
    unsigned long setbit,tagbit;
    setbit = (ad>>b) << (64 - s) >> (64 - s);
    // 用来找 set 组
    tagbit = ad >> (s + b);
    // 用来验证 tag

    char flag = 0; // 标记
    for (int i = setbit*e; i <= (setbit+1)*e-1 ; i++)
    {
        if(cache[i].val == 0) 
        // 这说明这块缓存还没用，而且由于我是顺序处理，后面的肯定也没用，所以直接用这块缓存
        {
            if(mode == 'M') hit++;
            miss++;

            cache[i].val = 1;
            cache[i].tag = tagbit;
            cache[i].L = ++cnt;
            flag = 1;
            break;
        }

        // 下面表示找到了 tag 符合的
        if(cache[i].tag == tagbit)
        {
            if(mode =='M') hit++;
            hit++;
            cache[i].L = ++cnt;
            flag = 1;
            break;
        }
    }
    // 下面表示，循环了一圈发现都没有符合的，LRU 驱逐
    if(flag == 0)
    {
        out++;
        miss++;
        if(mode == 'M') hit++;
        int minpos = setbit * e;
        int uppos = (setbit + 1) * e - 1;
        for (int i = setbit * e; i <= uppos; i++)
            if(cache[i].L < cache[minpos].L) 
                minpos = i;
        cache[minpos].tag = tagbit;
        cache[minpos].L = ++cnt;   
    }
}
```

6. 整理程序
```c
fclose(fp);
printSummary(hit, miss, out);
free(cache);
return 0;
// 这份程序里我忽略了 -v 详细模式的输出，见原代码里有这一部分
```