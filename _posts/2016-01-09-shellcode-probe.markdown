---
layout: post
title:  "Stack Over Flow 初探"
date:   2016-01-09 19:53:44
author: sunrainchy
categories: ShellCode
image: http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2015.12.12/nginx.jpg
---
从一次实践来了解什么是StackOverFlow，从一段简单的C语言代码入手。
cat exploit.c 
void fun_1(void) 
{ 
    printf("This is fun 1\n"); 
    char buffer[10]; 
    gets(buffer); 
} 
int main() 
{ 
    fun_1(); 
}
保存为：exploit.c
cc -mpreferred-stack-boundary=2 -ggdb exploit.c -o exploit -fno-stack-protector 
-fno-stack-protector 告诉gcc我们不想一栈溢出保护机制进行编译（如果不加上实验会失败）

编译好了先用GDB看一下：
<div class="post-img">
<img class="img-responsive img-post" src="http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2015.12.12/ngx_top_20151214.png"/>
</div>

执行disassemble 命令，对main函数进行反汇编
<div class="post-img">
<img class="img-responsive img-post" src="http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2015.12.12/ngx_top_20151214.png"/>
</div>

可以看到在调用完成fun_1 函数后下一条指令地址是0x0804482b

再看看fun_1 的反汇编结果
disassemble fun_1

<div class="post-img">
<img class="img-responsive img-post" src="http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2015.12.12/ngx_top_20151214.png"/>
</div>

分析下这段汇编代码：
Dump of assembler code for function fun_1: 
   0x08048404 <+0>: push   %ebp                                    // 基栈寄存器入站，保存
   0x08048405 <+1>: mov    %esp,%ebp                          // 用当前栈顶作为基栈地址
   0x08048407 <+3>: sub    $0x10,%esp                           // 给char buffer[10]; 在栈上分配地址，注意这里分配了16个字节，注意Linux平台栈是从高地址向低地址延伸的
   0x0804840a <+6>: movl   $0x8048500,(%esp)              
   0x08048411 <+13>:    call   0x8048320 <puts@plt>   // 调用printf 输出
   0x08048416 <+18>:    lea    -0xa(%ebp),%eax            // 找到正确的位置进行串的输入
   0x08048419 <+21>:    mov    %eax,(%esp) 
   0x0804841c <+24>:    call   0x8048310 <gets@plt>    //调用gets 输入
   0x08048421 <+29>:    leave  
   0x08048422 <+30>:    ret    


在这里我们可以设置两个断点，看看此函数返回main 函数下一条指令地址是不是0x0804482b：
(gdb) break *0x08048422 
Breakpoint 1 at 0x8048422: file exploit.c, line 7. 

在最后ret 指令出停止

<div class="post-img">
<img class="img-responsive img-post" src="http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2015.12.12/ngx_top_20151214.png"/>
</div>

看第一行，找到0x0804482b了吧

下面通过栈溢出篡改函数返回执行地址，我们可以看到此函数返回执行地址是 0x0804482b
让我们让程序再调用一次fun_1 函数：
从 disassemble main 可以看到调用fun_1 的时候是这个地址 0x08048426
那我们就将函数返回执行地址篡改成这个：

printf "AAABBBCCCDDDDD\x26\x84\x04\x08" | ./exploit
This is fun 1 
This is fun 1 

输出两个 ”This is fun 1 ” 演示成功

为啥是14个字符后接地址而不是10个呢，原因是基栈寄存器入栈了0x08048404 <+0>:  push   %ebp 
，要将这四个也算上。



