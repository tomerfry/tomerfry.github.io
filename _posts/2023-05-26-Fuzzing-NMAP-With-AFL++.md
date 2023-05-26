---
layout: post
title: "Fuzzing NMAP with AFL++"
date: 2023-05-26 03:00:00 +02:00
author: "Tomer Goldschmidt"
tags: fuzzing afl++
---

I've been wanting to research a common utility software for a long time. I figured I might as well try researching something I had somewhat of an experience working with.
After some thought, I decided looking into `nmap`.
At first I wanted to open the binary in `ghidra` and start analyzing, but before delving into binary reverse engineering I googled if anyone did something like this before.
I found an [article](https://katsuragicsl.github.io/posts/2022/08/fuzzing-nmap1-a-dumb-approach/) explaining about fuzzing the utility `nmap`. It sounded like a great opportunity to give fuzzing with `AFL++` a go.

---
## Getting `AFL++`
```bash
user@ubuntu:~$ sudo docker pull aflplusplus/aflplusplus
```

---
## Reproducing Article Results
First thing I thought might be beneficial for me was to reproduce the results from the Article, and to see for myself what happens.

The following describes the steps I took to reproduce the results:
1. Downloaded sources for `nmap` from [https://nmap.org/download.html](https://nmap.org/download.html)
<img src="/assets/2023052614749.png" />

2. Started the `AFL++` docker and started prepping for fuzzing session
```bash
user@ubuntu:~/fuzzin-nmap/nmap-src/$ docker run -ti -v `pwd`:/src aflplusplus/aflplusplus
```
 3. Configured the `nmap` project as described in the article using `AFL++ clang` compilers
```bash
[AFL++ 1ab0a53dd9fb] /src # AFL_USE_ASAN=1 CC=`which afl-clang-fast` CXX=`which afl-clang-fast++` ./configure --prefix="$HOME/nmap_fuzzing/install/"
```
 5. Used the `make` utility to build the project with my configuration.
```bash
[AFL++ 403c784b2306] ~ # AFL_USE_ASAN=1 make && AFL_USE_ASAN=1 make install
```
5. Started A fuzzing session.
```bash
[AFL++ 403c784b2306] ~ # afl-fuzz -m none -i ~/corpus/ -o ~/output -s 999 -- ~/nmap_fuzzing/install/bin/nmap 127.0.0.1 -p 80 --script @@
```

<img src="/assets/20230526021804.png" />
After some time letting the session go, I stopped it with the understanding that its not so optimal. The author of the article even presented his conclusions for his demonstration. One of which, to make the fuzzing more focused.

---
## My Conclusion
Making a fuzzing session on plainly running `nmap` was a bit naïve. `nmap` has the `lua` scripting engine built-in to evaluate `nse` scripts. This makes it unreasonable for me to test this particular area of the software. With that I concluded that i need pivot my fuzzing effort in to a different area of the software.

---
## Testing Packet Parsing Capability
Another option I had for testing `nmap` was testing its packet parsing capability. Apparently `nmap` has a module in the tests folder that is used for testing the tool. So I decided to try fuzzing this target and see what might pop up while fuzzing.
But before starting to fuzz this sample, I needed to tweak the test module a little bit. Specifically removing its network querying functionality. 
I did this because I only wanted to test the parsing of response packets.

My edit results for `src/tests/nmap_dns_test.cc`:
```c
#include "../nmap_dns.h"

#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>


int main(int argc, char **argv)
{
  int fd = open(argv[1], O_RDONLY);
  struct stat st;
  fstat(fd, &st);
  long int sz = st.st_size;
  u8 *answer_buf = (u8 *)malloc(st.st_size);
  DNS::Packet p;
  if (sz != read(fd, answer_buf, sz))
  {
    free(answer_buf);
    return -1;
  }

  p.parseFromBuffer(answer_buf, sz);
  free((void *)answer_buf);
  return 0; // 0 means ok
}
```

```bash
[AFL++ 403c784b2306] /src # apt install libssh-dev
[AFL++ 403c784b2306] /src # AFL_USE_ASAN=1 make
[AFL++ 403c784b2306] /src # AFL_USE_ASAN=1 make tests/check_dns
[AFL++ 403c784b2306] /src # afl-fuzz -m none -i ~/corpus/ -o ~/output -s 999 -- ./tests/check_dns @@
```

Seemingly I get some crashes, but I am doubtful this means a bug/vulnerability.
<img src="/assets/20230526113258.png" />

---
## Triaging
Looking at the crash results with my fuzzing setup I can see that some kind of `Heap-Overflow` has occurred.

Below is the prompt when running the instrumented `check_dns` binary with the crash input:
<img src="/assets/20230526113912.png" />

When testing this crash sample on a clean variation of `check_dns` on my host machine, it seems that the program doesn't crash.
* I did notice that I get a traceback with the function that causes the crash. `DNS::Query::parseFromBuffer(...)` is the place where the overflow allegedly occurs.
  Specifically in `/src/nmap_dns.cc:1527:27`.

---
## Root Cause Analysis
After some reading I figured out that the issue in this case is an issue of reading beyond the bounds of the allocated buffer on the heap.
This might cause issues in the program. In my case it did not cause any significant effect on the program I used for fuzzing.

---
## Summary
I wanted to finish this post about now, with the intention of investigating and researching more. I think that writing this blog post helped me a lot in gathering more experience with fuzzing. Specifically with `AFL++`. Thank you for reading this post, I hope that it was informative and  fun for you readers.


