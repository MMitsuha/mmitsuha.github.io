---
title: d3kernel-wp
date: 2025-06-01 17:59:19
categories:
    - "CTF"
tags: 
    - "d3ctf"
---

## 中文

### 绕过R3反调试

挂上WinDbg双机调试加载驱动之后，发现WinDbg不再响应了（如图），应该是通过`KdDisableDebugger`禁用了调试器。
![no response in WinDbg](image.png)

打开YDArk，看到驱动注册了两个回调和一个线程。
![YDArk callback](image-1.png)
![YDArk thread](image-2.png)

<!-- more -->

直接调试client，发现创建了一个shellcode，引发除零异常以隐藏程序控制流。
![divide zero](image-3.png)

继续往下，又是一个故意的空指针异常。
![null pointer](image-4.png)

题目有反调试，先对几个检测调试器的函数下断点。
![detect debugger](image-5.png)

都被断下了，说明程序使用了这两个函数检测调试器。
![detect function](image-6.png)

手动patch，使这些函数直接`ret`返回。
![patch function](image-7.png)

单步回到检测调试器的代码中，可以看到隐藏了导入表，函数都是动态获取的。
![dynamic function](image-8.png)

在下一个`call rax`下断点，发现确实调用了`CheckRemoteDebuggerPresent`。
![CheckRemoteDebuggerPresent](image-9.png)

直接patch返回值检测（两个检测都要patch，否则将会进入`fake_main`里）
![patch return value](image-10.png)

慢慢跟到`CreateFileW`。
![CreateFileW](image-11.png)

下一个`call rax`为`DeviceIoControl`，发现进程是通过`DeviceIoControl`与驱动通信的。由此我们可以得知程序与驱动交互所使用的结构体等信息。
![DeviceIoControl](image-12.png)

### 绕过R0反调试

当我们配置好双机调试，加载驱动后，我们发现windbg不再响应我们的命令，这是由于`KdDisableDebugger`关闭了双机调试，所以我们要在加载驱动之前patch一下`KdDisableDebugger`，使它不发挥任何作用。
![before](image-13.png)

直接在`KdDisableDebugger`的开头`ret`
![after](image-14.png)

现在加载驱动就不会掉调试器了。当然你也可以以`nop`填充驱动本身创建线程处的代码，条条大路通罗马。
![debugger alive after loading driver](image-15.png)

### 预期解：虚拟机的逆向

既然已经知道是`DeviceIoControl`，那就看看`DRIVER_OBJECT`里的`MajorFunction`。
![driver object](image-16.png)

第十五个函数就是`DeviceIoControl`的处理函数了，下断点，再去用户层发请求看看。
![DeviceIoControl](image-17.png)

成功被断下。
![break on DeviceIoControl](image-18.png)

在IDA里可以看到这个函数超级大，inline了一堆东西。
![big function](image-19.png)

另外还发现它调用了一个看起来像虚拟机的函数。
![virtual machine](image-20.png)

把伪代码扔给ChatGPT看。
![chatgpt](image-21.png)

基本上都是正确的，接下来逆向虚拟机实际执行的代码，对`vm_init`下断点。
![vm_init](image-22.png)

其中参数分别为虚拟机结构体，虚拟机的代码指针，虚拟机代码长度。
![vm_init parameter](image-23.png)

断下后寻找一个`code_size`最大的，查看虚拟机代码。
![vm code](image-24.png)

根据虚拟机代码中的定义可以得出这段虚拟机代码是将两个输入的数字异或，第一个数字为buffer的长度。

通过IDA查看到编码后的username和password为：
![encoded value](image-25.png)

编写一个python脚本解密：

```python
def decrypt(buffer):
    decrypted = [len(buffer)]
    for i in range(len(buffer)):
        decrypted.append(buffer[i] ^ decrypted[i])
    return "".join([chr(x) for x in decrypted[1:]])
```

![decoded](image-26.png)

### 非预期解：爆破

在比赛过程中，我注意到有一些选手使用了爆破的方法，这是由于这段编码较为简单，而且有很强的特征，因此我们可以在驱动验证编码后的内容是否相等的时候下断点，当我们的输入字符正确的话，我们应该可以观测到`memcmp`访问了下一个字节的数据。下面我们将会验证这个想法。

这是驱动验证编码后的内容是否相等处的代码。
![validate input](image-27.png)

我们只要在`if ( v30 != v31 )`处下断点，即可知道这一位是否就是正确的。
![break on validate](image-28.png)

当我们输入用户名为111111时，跳转成立，则1不是用户名的第一位。
![input](image-30.png)
![zf = 0](image-29.png)

我们可以这样依次尝试，当我们输入m111111时，跳转不成立，则m是用户名的第一位。
![input](image-31.png)
![zf = 1](image-32.png)

这样你只需要爆破7 + 36次即可得出答案！

## English (Translated by ChatGPT)

### Bypassing R3 Anti-Debugging

After attaching WinDbg for kernel-mode debugging and loading the driver, WinDbg stopped responding (as shown in the picture), which is likely due to `KdDisableDebugger` disabling the debugger.
![no response in WinDbg](image.png)

Opening YDArk, I found that the driver registered two callbacks and a thread.
![YDArk callback](image-1.png)
![YDArk thread](image-2.png)

Directly debugging the client, I noticed a shellcode was created that triggers a divide-by-zero exception to hide the program's control flow.
![divide zero](image-3.png)

Continuing further, there was another deliberate null pointer exception.
![null pointer](image-4.png)

The challenge includes anti-debugging mechanisms, so I set breakpoints on several debugger detection functions.
![detect debugger](image-5.png)

All breakpoints were hit, indicating that the program uses these two functions to detect the debugger.
![detect function](image-6.png)

I manually patched these functions to immediately `ret` and return.
![patch function](image-7.png)

Stepping back into the debugger detection code, I saw that the import table was hidden and all functions were dynamically retrieved.
![dynamic function](image-8.png)

I set a breakpoint on the next `call rax` and confirmed that `CheckRemoteDebuggerPresent` was indeed called.
![CheckRemoteDebuggerPresent](image-9.png)

I directly patched the return value check (both checks need to be patched, otherwise the program will enter `fake_main`).
![patch return value](image-10.png)

I slowly traced the flow to `CreateFileW`.
![CreateFileW](image-11.png)

The next `call rax` was for `DeviceIoControl`, and I discovered that the process communicates with the driver via `DeviceIoControl`. This revealed the structure used by the program to interact with the driver.
![DeviceIoControl](image-12.png)

### Bypassing R0 Anti-Debugging

After setting up dual-machine debugging and loading the driver, we found that Windbg stopped responding to our commands. This was because `KdDisableDebugger` disabled the dual-machine debugger. Therefore, we need to patch `KdDisableDebugger` before loading the driver to prevent it from taking effect.
![before](image-13.png)

Simply add a `ret` at the beginning of `KdDisableDebugger`.
![after](image-14.png)

Now, loading the driver won't drop the debugger. Alternatively, you can also `nop` out the code where the driver itself creates a thread, as there are many ways to achieve this.
![debugger alive after loading driver](image-15.png)

### Expected Solution: Virtual Machine Reversal

Since we already know it's using `DeviceIoControl`, let's examine the `MajorFunction` in the `DRIVER_OBJECT`.
![driver object](image-16.png)

The fifteenth function is the handler for `DeviceIoControl`. Set a breakpoint and then send a request from the user space.
![DeviceIoControl](image-17.png)

The breakpoint is successfully hit.
![break on DeviceIoControl](image-18.png)

In IDA, you can see that this function is huge and inlined with a bunch of other code.
![big function](image-19.png)

Additionally, I noticed it calls a function that looks like it belongs to a virtual machine.
![virtual machine](image-20.png)

I threw the pseudocode into ChatGPT for analysis.
![chatgpt](image-21.png)

It was mostly correct. Next, I reversed the actual code executed by the virtual machine and set a breakpoint at `vm_init`.
![vm_init](image-22.png)

The parameters passed to `vm_init` are the virtual machine structure, the VM code pointer, and the VM code length.
![vm_init parameter](image-23.png)

After hitting the breakpoint, I searched for the largest `code_size` and inspected the virtual machine code.
![vm code](image-24.png)

From the definitions in the VM code, it was clear that this VM code XORs two input numbers, where the first number is the length of the buffer.

By examining the encoded username and password in IDA, I found the following:
![encoded value](image-25.png)

I wrote a Python script to decrypt it:

```python
def decrypt(buffer):
    decrypted = [len(buffer)]
    for i in range(len(buffer)):
        decrypted.append(buffer[i] ^ decrypted[i])
    return "".join([chr(x) for x in decrypted[1:]])
```

### Unexpected Solution: Brute Force

During the competition, I noticed that some participants used a brute-force approach. This was due to the simplicity of the encoding and its strong characteristics. Therefore, we can set a breakpoint where the driver validates whether the encoded content is equal. When our input characters are correct, we should be able to observe `memcmp` accessing the next byte of data. Let's verify this idea.

This is the code where the driver validates whether the encoded content matches.
![validate input](image-27.png)

By setting a breakpoint at `if (v30 != v31)`, we can determine whether the current byte is correct.
![break on validate](image-28.png)

When we input the username 111111, the jump occurs, meaning that 1 is not the first character of the username.
![input](image-30.png)
![zf = 0](image-29.png)

We can try this step by step. When we input m111111, the jump does not occur, so m is the first character of the username.
![input](image-31.png)
![zf = 1](image-32.png)

With this method, you only need to brute force 7 + 36 times to find the answer!
