---
layout: post
title: "Using Radare2"
date: 2023-05-24 06:35:00 +02:00
author: "Tomer Goldschmidt"
tags: tools radare2
---


I wanted for a long time to document my workflow with `radare2`. And decided that now might be the best time.
The truth is that I am adjusting into it and have a lot more to learn about this gigantic tool.

---
## Goal 🎯
Be able to reverse engineer a sample - starting from a simple binary sample we already know. -> Our candidate will be `level01` from the `fusion` exploit exercises sequence.

---
## Using The Cheatsheet 📄
* There is a [Cheatsheet](https://github.com/radareorg/radare2/blob/master/doc/intro.md) online that I will try to utilze while practicing with radare2.
* There is also this online book that I used [Radare2-Book](https://book.rada.re/)

---
## r2ghidra Plugin
* Install the ghidra compile into `radare2`.
```bash
user@ubuntu:~/fusion/level01$ sudo r2pm -ci r2ghidra
```

---
## Command Line Workflow
1. Started by opening the tool with `level01` sample:
```bash
user@ubuntu:~/fusion/level01$ r2 ./level01
 -- Control the height of the terminal on serial consoles with e scr.height
[0x08048b70]> 
[0x08048b70]> 
```

2. Analyzed the sample.
```bash
[0x08048b70]> aaa
```

3. Usually I would look at what function exist in samples
```
0x08048b73]> afl
0x08048b70    1     33 entry0
0x08048a60    1      6 sym.imp.__libc_start_main
...
0x08048a50    1      6 sym.imp.strchr
0x0804997a    1     72 main
...
```

4. I found that the `main()` was named and so I repositioned to this function to decompile it.
```bash
[0x08048b73]> s main
[0x0804997a]> pdg

void main(void)
{
    uint fildes;   
    sym.background_process("level01", 0x4e21, 0x4e21);
    fildes = sym.serve_forever(0x4e21);
    sym.set_io(fildes);
    sym.parse_http_request();
    return;
}
```
* That way I could start analyzing the binary.

---
## Binary Pattern Searching 🔍💪
* The most beneficial feature I used in `radare2` was the binary pattern matching and searching feature.
* It supports searching for binary patterns with wild cards, creaing structural patterns and also for assembly directives.
* For example, here I show how I look for specific `ROP Gadgets` in the binary code.
```bash
0x08048b70]> /R
...
[0x08048b70]> /x 31..5e
Searching 3 bytes in [0x804b470-0x804b4ac]
hits: 0
Searching 3 bytes in [0x804b2bc-0x804b470]
hits: 0
Searching 3 bytes in [0x8048000-0x804a2bc]
hits: 1
0x08048b70 hit1_0 31ed5e
```

---
## GUI ⁉
* Well apperently this tool as a pluging called `iaito` that implements a Graphical User Interface. 🤔
* Install `iaito` and have a graphic interface with decompiler.
```bash
user@ubuntu:~/fusion/level01$ sudo r2pm -ci iaito
...
user@ubuntu:~/fusion/level01$ iaito ./level01
```
* And there you go...

<img src="radare2-iaiot.png" />


---
## Summery
This was a rather short post and yet I feel it could be beneficial for someone. 
I feel that writing this post was very refreshing. It made me think about maintained projects I am using today or have used in the past. 
The kind of projects you move on after using them once.
Well, they tend to keep evolving through time, and maybe I should from time to time check on them. Because who knows, maybe they will come handy on a second iteration.
Thanks for reading this post, I really hope it was informative for you.
