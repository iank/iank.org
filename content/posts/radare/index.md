---
template: post
title: 'Using radare2 to make a simple binary patch'
slug: using-radare2-to-make-a-simple-binary-patch
featured_image: radare2_sq.png
draft: false
date: 2019-12-27T14:30:00.000Z
description: My partner found a game on her computer that she probably downloaded a decade ago, and we wanted to tweak the behaviour.
category: security
tags:
  - reverse engineering
  - assembly
---
My partner found a game on her computer that she probably downloaded a decade ago, and we wanted to tweak the behaviour.

In case it's not clear, the following is outrageous and the right way to do this would have been to edit the freely-available C++ source code and rebuild the game, either on my partner's Macbook or cross-compiled from my Linux laptop. But I didn't want to interrupt her. And while the build system for this game is actually fairly straightforward, I have 36 megabytes free on my aging Chromebook. No room for an OS X toolchain. In the end, I think this saved everyone precious block-clicking time and it was certainly more fun.

The first step was to identify the game, which seemed like the product of an SDK tutorial or a tech demo. In fact it was the "blocks" test app for [FLTK](https://www.fltk.org/doc-1.3/examples.html#examples_blocks). The [source code](https://github.com/fltk/fltk/blob/release-1.1.10/test/blocks.cxx) is straightforward and short, and the change we wanted to make was in the level up progression. Each level adds a new level of complexity and also speeds the game up; we wanted to keep adding complexity without speeding up the game.

Here is the relevant function:

```c++
void BlockWindow::up_level() {
  interval_ *= 0.95;
  opened_columns_ = 0;
  if (num_colors_ < 7) num_colors_ ++;
  level_ ++;
  sprintf(title_, "Level: %d", level_);
  title_y_ = h();
  Fl::repeat_timeout(interval_, (Fl_Timeout_Handler)timeout_cb, (void *)this);
}
```

All we need to do is delete the code that changes `interval_` or change the `0.95` constant to `1`. It's been a while since I've had access to IDA Pro, so it's time for me to learn a new tool, [radare2](https://rada.re/).

So, we open the binary, analyze it, and list functions:

```txt
ian@chrx:~/Downloads/blocks.app/Contents/MacOS$ radare2 blocks 
[0x100004af0]> aaa
[0x100004af0]> afl

...
0x100005df6 627   8  fcn.100005df6
0x100009a6c 19   1  fcn.100009a6c
0x10000606a 139   3  sym.__ZN11BlockWindow8up_levelEv
0x100005750 337   3  sym.__ZN11BlockWindow5clickEii
0x100005796 14  15  fcn.10000579a
0x1000057a4 253  12  fcn.1000057a4
...
```

Great. It's not stripped. With some searching, we find the symbol we're looking for. `BlockWindow::up_level` has become `__ZN11BlockWindow8up_levelEv` due to [C++ name mangling](https://en.wikipedia.org/wiki/Name_mangling#C++). We can seek to it and print the disassembly:

```txt
[0x100004af0]> s sym.__ZN11BlockWindow8up_levelEv
[0x10000606a]> pdf
|           ; CODE (CALL) XREF from 0x100005f50 (fcn.1000058c0)
|           ; CODE (CALL) XREF from 0x1000054e6 (fcn.1000050dc)
   ;      BlockWindow::up_level()
/ (fcn) sym.__ZN11BlockWindow8up_levelEv 139
|           0x10000606a    55           push rbp
|           0x10000606b    4889e5       mov rbp, rsp
|           0x10000606e    53           push rbx
|           0x10000606f    50           push rax
|           0x100006070    4889fb       mov rbx, rdi
|           0x100006073    f30f5a83100. cvtss2sd xmm0, [rbx+0xb10]
|           0x10000607b    f20f59050d1. mulsd xmm0, [rip+0x4180d]
|           0x100006083    f20f5ac0     cvtsd2ss xmm0, xmm0
|           0x100006087    f30f1183100. movss [rbx+0xb10], xmm0
|           0x10000608f    c7831c0b000. mov dword [rbx+0xb1c], 0x0
|           0x100006099    8b83180b0000 mov eax, [rbx+0xb18]
|           0x10000609f    83f806       cmp eax, 0x6
|       ,=< 0x1000060a2    7f08         jg 0x1000060ac
|       |   0x1000060a4    ffc0         inc eax
|       |   0x1000060a6    8983180b0000 mov [rbx+0xb18], eax
|       `-> 0x1000060ac    8b93140b0000 mov edx, [rbx+0xb14]
|           0x1000060b2    ffc2         inc edx
|           0x1000060b4    8993140b0000 mov [rbx+0xb14], edx
|           0x1000060ba    488dbb300b0. lea rdi, [rbx+0xb30]
|           ; CODE (CALL) XREF from 0x100009a7d (fcn.100009a7d)
|           ; DATA XREF from 0x100009a7d (fcn.100009a7d)
|           0x1000060c1    488d35b5390. lea rsi, [rip+0x439b5] ; 0x100009a7d 
|           0x1000060c8    31c0         xor eax, eax
|           0x1000060ca    e87f0b0400   call sym.imp.sprintf
|              sym.imp.sprintf(unk, unk, unk)
|           0x1000060cf    8b432c       mov eax, [rbx+0x2c]
|           0x1000060d2    8983300c0000 mov [rbx+0xc30], eax
|           0x1000060d8    f30f5a83100. cvtss2sd xmm0, [rbx+0xb10]
|           ; CODE (CALL) XREF from 0x1000150dc (fcn.1000150dc)
|           ; CODE (CALL) XREF from 0x100015159 (fcn.100015159)
|           ; CODE (CALL) XREF from 0x100015117 (fcn.100015117)
|           ; DATA XREF from 0x1000150dc (fcn.1000150dc)
|           0x1000060e0    488d3df5eff. lea rdi, [rip-0x100b] ; 0x1000150dc 
|           0x1000060e7    4889de       mov rsi, rbx
|           0x1000060ea    4883c408     add rsp, 0x8
|           0x1000060ee    5b           pop rbx
|           0x1000060ef    5d           pop rbp
\           0x1000060f0    e94f1e0000   jmp sym.__ZN2Fl14repeat_timeoutEdPFvPvES0_
``` 

These are the lines in question. They're relatively easy to pick out since we're interested in a floating point operation:

```txt
|           0x100006073    f30f5a83100. cvtss2sd xmm0, [rbx+0xb10]
|           0x10000607b    f20f59050d1. mulsd xmm0, [rip+0x4180d]
|           0x100006083    f20f5ac0     cvtsd2ss xmm0, xmm0
|           0x100006087    f30f1183100. movss [rbx+0xb10], xmm0
```

Remember, we're looking for assembly code corresponding to `interval_ *= 0.95`. This fits. We:

* load `interval_` (from the address `rbx+0xb10`)
* multiply by a constant (at `rip+0x4180d`)
* and store the result back in `rbx+0xb10`

To make sure, we can check the value of the constant in `[rip+0x4180d]`. It should be `0.95`. We know it's an argument to mulsd, so we're looking for a [double](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64). Recall `rip` points to the next instruction to be executed, so:

```txt
[0x10000606a]> pf q @ 0x100006083+0x4180d
0x100047890 = (qword) 0x3fee666666666666 
```

And indeed, if we [convert it to decimal](https://binaryconvert.com/result_double.html?hexadecimal=3FEE666666666666), we get `0.9499999...`.

Now all we need to do is find this code in the binary and [NOP](https://www.felixcloutier.com/x86/nop) it out. It's easy enough to search for the corresponding machine code in a hex editor since radare's disassembler has provided it alongside the assembly, and replace that section with `0x90`s.

Radare2 is a cool tool, though, and there's ways to do the patching entirely within the tool. [This tutorial](https://monosource.gitbooks.io/radare2-explorations/content/tut1/tut1_-_simple_patch.html) has an example.
