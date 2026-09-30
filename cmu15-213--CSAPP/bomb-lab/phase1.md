# Bomb Lab 
## Phase 1 记录

**我的解决步骤：**<br>
1. 学习 [GDB调试指南](https://www.yanbinghu.com/2019/04/20/41283.html) 第一章，会启动调试即可
2. 学习gdb内的 `(gdb) disassemble phase_1`

<br>

**以上即可解决 phase1,**<br>
***具体内容如下，结合注释一起看即得***<br>
```x86asm
(gdb) disassemble phase_1
Dump of assembler code for function phase_1:
   // 16字节对齐(上文的call会多出8字节)
   0x0000000000400ee0 <+0>:     sub    $0x8,%rsp 
   // bomb.c里告诉了我们，传入的参数是input。在%rdi里，这里补上了%rsi，显然是为了下面的函数提供参数做准备
   0x0000000000400ee4 <+4>:     mov    $0x402400,%esi
   // 调用函数，参数是%rdi和%rsi，猜测是比较函数
   0x0000000000400ee9 <+9>:     call   0x401338 <strings_not_equal>
   // 显然这里返回的是0或者1，要不然不能用test
   0x0000000000400eee <+14>:    test   %eax,%eax
   // 返回值 %rax 是0就过，不是就爆炸
   0x0000000000400ef0 <+16>:    je     0x400ef7 <phase_1+23>
   0x0000000000400ef2 <+18>:    call   0x40143a <explode_bomb>
   // 还原栈指针
   0x0000000000400ef7 <+23>:    add    $0x8,%rsp
   0x0000000000400efb <+27>:    ret
End of assembler dump.
```
有兴趣的还可以读一下下面的汇编代码，挺无聊的，猜也知道答案是去找上一段的%rsi里地址对应的字符串
```x86asm
(gdb) disassemble strings_not_equal
Dump of assembler code for function strings_not_equal:
   0x0000000000401338 <+0>:     push   %r12
   0x000000000040133a <+2>:     push   %rbp
   0x000000000040133b <+3>:     push   %rbx
   0x000000000040133c <+4>:     mov    %rdi,%rbx
   0x000000000040133f <+7>:     mov    %rsi,%rbp
   0x0000000000401342 <+10>:    call   0x40131b <string_length>
   0x0000000000401347 <+15>:    mov    %eax,%r12d
   0x000000000040134a <+18>:    mov    %rbp,%rdi
   0x000000000040134d <+21>:    call   0x40131b <string_length>
   0x0000000000401352 <+26>:    mov    $0x1,%edx
   0x0000000000401357 <+31>:    cmp    %eax,%r12d
   0x000000000040135a <+34>:    jne    0x40139b <strings_not_equal+99>
   0x000000000040135c <+36>:    movzbl (%rbx),%eax
   0x000000000040135f <+39>:    test   %al,%al
   0x0000000000401361 <+41>:    je     0x401388 <strings_not_equal+80>
   0x0000000000401363 <+43>:    cmp    0x0(%rbp),%al
   0x0000000000401366 <+46>:    je     0x401372 <strings_not_equal+58>
   0x0000000000401368 <+48>:    jmp    0x40138f <strings_not_equal+87>
   0x000000000040136a <+50>:    cmp    0x0(%rbp),%al
   0x000000000040136d <+53>:    nopl   (%rax)
   0x0000000000401370 <+56>:    jne    0x401396 <strings_not_equal+94>
   0x0000000000401372 <+58>:    add    $0x1,%rbx
   0x0000000000401376 <+62>:    add    $0x1,%rbp
   0x000000000040137a <+66>:    movzbl (%rbx),%eax
   0x000000000040137d <+69>:    test   %al,%al
   0x000000000040137f <+71>:    jne    0x40136a <strings_not_equal+50>
   0x0000000000401381 <+73>:    mov    $0x0,%edx
   0x0000000000401386 <+78>:    jmp    0x40139b <strings_not_equal+99>
   0x0000000000401388 <+80>:    mov    $0x0,%edx
   0x000000000040138d <+85>:    jmp    0x40139b <strings_not_equal+99>
   0x000000000040138f <+87>:    mov    $0x1,%edx
   0x0000000000401394 <+92>:    jmp    0x40139b <strings_not_equal+99>
   0x0000000000401396 <+94>:    mov    $0x1,%edx
   0x000000000040139b <+99>:    mov    %edx,%eax
   0x000000000040139d <+101>:   pop    %rbx
   0x000000000040139e <+102>:   pop    %rbp
   0x000000000040139f <+103>:   pop    %r12
   0x00000000004013a1 <+105>:   ret
   ```