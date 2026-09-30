# Bomb Lab 
## Phase 4 记录

**我的解决步骤：**<br>
1. GDB 调试 ：`gdb bomb`
2. 反汇编 : `(gdb) disassemble phase_3` 
3. 把部分代码翻译成高级语言 (cpp) 搞清楚流程即可

<br>

***代码如下，结合注释阅读***
```x86asm
(gdb) disassemble phase_4
Dump of assembler code for function phase_4:
   // 前面的和 phase_3 完全如出一辙，设输入的两个整数是 a1(%rdx) 和 a2(%rcx)
   0x000000000040100c <+0>:     sub    $0x18,%rsp
   0x0000000000401010 <+4>:     lea    0xc(%rsp),%rcx
   0x0000000000401015 <+9>:     lea    0x8(%rsp),%rdx
   0x000000000040101a <+14>:    mov    $0x4025cf,%esi
   0x000000000040101f <+19>:    mov    $0x0,%eax
   0x0000000000401024 <+24>:    call   0x400bf0 <__isoc99_sscanf@plt>
   0x0000000000401029 <+29>:    cmp    $0x2,%eax
   0x000000000040102c <+32>:    jne    0x401035 <phase_4+41>
   // 这里是说第一个参数要 <=15
   0x000000000040102e <+34>:    cmpl   $0xe,0x8(%rsp)
   0x0000000000401033 <+39>:    jbe    0x40103a <phase_4+46>
   0x0000000000401035 <+41>:    call   0x40143a <explode_bomb>
   // 下面三行表示初始传参应当为 a1,0,15,a2
   0x000000000040103a <+46>:    mov    $0xe,%edx
   0x000000000040103f <+51>:    mov    $0x0,%esi
   0x0000000000401044 <+56>:    mov    0x8(%rsp),%edi
   0x0000000000401048 <+60>:    call   0x400fce <func4>
   // 以上为函数调用阶段，看下面。
   // 返回值应当为0，否则爆炸
   0x000000000040104d <+65>:    test   %eax,%eax
   0x000000000040104f <+67>:    jne    0x401058 <phase_4+76>
   // r2应当为0，否则爆炸。这就是全部条件。
   0x0000000000401051 <+69>:    cmpl   $0x0,0xc(%rsp)
   0x0000000000401056 <+74>:    je     0x40105d <phase_4+81>
   0x0000000000401058 <+76>:    call   0x40143a <explode_bomb>
   0x000000000040105d <+81>:    add    $0x18,%rsp
   0x0000000000401061 <+85>:    ret
End of assembler dump.
```
```x86asm
// 说实话这段代码我没有完全分析，感觉汇编的顺序很蛋疼，我翻译成了c语言，在下方，建议直接看。
(gdb) disassemble func4
Dump of assembler code for function func4:
   0x0000000000400fce <+0>:     sub    $0x8,%rsp
   0x0000000000400fd2 <+4>:     mov    %edx,%eax
   0x0000000000400fd4 <+6>:     sub    %esi,%eax
   0x0000000000400fd6 <+8>:     mov    %eax,%ecx
   0x0000000000400fd8 <+10>:    shr    $0x1f,%ecx
   0x0000000000400fdb <+13>:    add    %ecx,%eax
   0x0000000000400fdd <+15>:    sar    $1,%eax
   0x0000000000400fdf <+17>:    lea    (%rax,%rsi,1),%ecx
   0x0000000000400fe2 <+20>:    cmp    %edi,%ecx
   0x0000000000400fe4 <+22>:    jle    0x400ff2 <func4+36>
   0x0000000000400fe6 <+24>:    lea    -0x1(%rcx),%edx
   0x0000000000400fe9 <+27>:    call   0x400fce <func4>
   0x0000000000400fee <+32>:    add    %eax,%eax
   0x0000000000400ff0 <+34>:    jmp    0x401007 <func4+57>
   0x0000000000400ff2 <+36>:    mov    $0x0,%eax
   0x0000000000400ff7 <+41>:    cmp    %edi,%ecx
   0x0000000000400ff9 <+43>:    jge    0x401007 <func4+57>
   0x0000000000400ffb <+45>:    lea    0x1(%rcx),%esi
   0x0000000000400ffe <+48>:    call   0x400fce <func4>
   0x0000000000401003 <+53>:    lea    0x1(%rax,%rax,1),%eax
   0x0000000000401007 <+57>:    add    $0x8,%rsp
   0x000000000040100b <+61>:    ret
End of assembler dump.
```
```cpp
// func4 翻译成 cpp 
int func(int rdi,int rsi,int rdx,int rcx){
    /* 
    这里涉及递归和一些四则运算，应该很容易可以画成流程图。
    先缩减变量，尽可能变成 1 / 2 个参数，然后画流程图即可。
    但是我没这么做
    */
    int rax = rdx;
    rax -= rsi;
    rcx = rax;
    rcx = (rcx >=0 ? 0 : 1);
    rax += rcx;
    rax >>= 1;
    rcx = rax + rsi;
    if(rcx-rdi>0){
        rdx = rcx - 1;
        func(rdi,rsi,rdx,rcx);
        rax *= 2;
        return rax;
    }
    if(rcx-rdi<=0){
        rax = 0;
        if(rcx-rdi>=0){
            return rax;
        }
        else{
            rsi = rcx + 1;
            func(rdi,rsi,rdx,rcx);
            rax = 1 + rax + rax;
            return rax;
        }
    }
}
// 最后需要让rax=0，rcx初始值要为0
// func初始传参: rdi = a1, rsi = 0, rdx = 15, rcx = 0 = a2
// 我们发现当 rdi=rcx 的时候就可以了，所以直接 a1 = 0, a2 = 0结束
```