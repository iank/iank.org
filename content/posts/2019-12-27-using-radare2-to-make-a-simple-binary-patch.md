---
template: post
title: 'Using radare2 to make a simple binary patch'
slug: using-radare2-to-make-a-simple-binary-patch
socialImage: /media/radare2_sq.png
draft: false
date: 2019-12-27T14:30:00.000Z
description: My partner found a game on her computer that she probably downloaded a decade ago, and we wanted to tweak the behaviour.
category: security
tags:
  - reverse engineering
  - assembly
---
My partner found a game on her computer that she probably downloaded a decade ago, and we wanted to tweak the behaviour.

In case it's not clear, everything that follows is insane and the right way to do this would have been to edit the freely-available C++ source code and rebuild the game, either on my partner's Macbook or cross-compiled from my Linux laptop. But I didn't want to interrupt her. And while the build system for this game is actually fairly straightforward, I have 36 megabytes free on my aging Chromebook. No room for an OS X toolchain. In the end, I think this saved everyone precious block-clicking time and it was certainly more fun.

The first step was to identify the game, which seemed like the product of an SDK tutorial or a tech demo. In fact it was the "blocks" test app for [FLTK](https://www.fltk.org/doc-1.3/examples.html#examples_blocks). The [source code](https://github.com/fltk/fltk/blob/release-1.1.10/test/blocks.cxx) is straightforward and short, and the change we wanted to make was in the level up progression. Each level adds a new level of complexity and also speeds the game up; we wanted to keep adding complexity without speeding up the game.

Here is the relevant function:

    void BlockWindow::up_level() {
      interval_ *= 0.95;
      opened_columns_ = 0;
      if (num_colors_ < 7) num_colors_ ++;
      level_ ++;
      sprintf(title_, "Level: %d", level_);
      title_y_ = h();
      Fl::repeat_timeout(interval_, (Fl_Timeout_Handler)timeout_cb, (void *)this);
    }

All we need to do is delete the code that changes `interval_` or change the `0.95` constant to `1`. It's been a while since I've had access to IDA Pro, so it's time for me to learn a new tool, [radare2](https://rada.re/).

So, we open the binary, analyze it, and list functions:

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

Great. It's not stripped. With some searching, we find the symbol we're looking for. `BlockWindow::up_level` has become `__ZN11BlockWindow8up_levelEv` due to [C++ name mangling](https://en.wikipedia.org/wiki/Name_mangling#C++). We can seek to it and print the disassembly:

<pre>
<span style="color:olive;">[0x100004af0]&gt;</span> s sym.__ZN11BlockWindow8up_levelEv
<span style="color:olive;">[0x10000606a]&gt;</span> pdf
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x100005f50 (fcn.1000058c0)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x1000054e6 (fcn.1000050dc)</span>
<span style="color:blue;">   ;      BlockWindow::up_level()
</span><span style="color:teal;">/ </span><span style="color:red;">(fcn) sym.__ZN11BlockWindow8up_levelEv</span> 139
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000606a</span>    <span style="color:gray;">55</span>           <span style="color:purple;">push</span><span style="color:teal;"> rbp</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000606b</span>    <span style="color:gray;">4889e5</span>       mov<span style="color:teal;"> rbp</span>,<span style="color:teal;"> rsp</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000606e</span>    <span style="color:gray;">53</span>           <span style="color:purple;">push</span><span style="color:teal;"> rbx</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000606f</span>    <span style="color:gray;">50</span>           <span style="color:purple;">push</span><span style="color:teal;"> rax</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x100006070</span>    <span style="color:gray;">4889fb</span>       mov<span style="color:teal;"> rbx</span>,<span style="color:teal;"> rdi</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x100006073</span>    <span style="color:gray;">f30f5a83100.</span> <span style="color:gray;">cvtss2sd</span><span style="color:teal;"> xmm0</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb10</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000607b</span>    <span style="color:gray;">f20f59050d1.</span> <span style="color:gray;">mulsd</span><span style="color:teal;"> xmm0</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rip</span>+<span style="color:teal;"></span><span style="color:olive;">0x4180d</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x100006083</span>    <span style="color:gray;">f20f5ac0</span>     <span style="color:gray;">cvtsd2ss</span><span style="color:teal;"> xmm0</span>,<span style="color:teal;"> xmm0</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x100006087</span>    <span style="color:gray;">f30f1183100.</span> <span style="color:gray;">movss</span><span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb10</span>]<span style="color:teal;"></span>,<span style="color:teal;"> xmm0</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000608f</span>    <span style="color:gray;">c7831c0b</span><span style="color:green;">00</span><span style="color:gray;">0.</span> mov dword<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb1c</span>]<span style="color:teal;"></span>,<span style="color:teal;"> </span><span style="color:olive;">0x0</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x100006099</span>    <span style="color:gray;">8b83180b</span><span style="color:green;">0000</span> mov<span style="color:teal;"> eax</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb18</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x10000609f</span>    <span style="color:gray;">83f806</span>       <span style="color:teal;">cmp eax</span>,<span style="color:teal;"> </span><span style="color:olive;">0x6</span>
<span style="color:teal;">| </span><span style="color:teal;">      ,=&lt; </span><span style="color:green;">0x1000060a2</span>    <span style="color:teal;">7f</span><span style="color:gray;">08</span>         <span style="color:green;">jg 0x1000060ac</span>
<span style="color:teal;">| </span><span style="color:teal;">      |   </span><span style="color:green;">0x1000060a4</span>    <span style="color:red;">ff</span><span style="color:gray;">c0</span>         <span style="color:olive;">inc</span><span style="color:teal;"> eax</span>
<span style="color:teal;">| </span><span style="color:teal;">      |   </span><span style="color:green;">0x1000060a6</span>    <span style="color:gray;">8983180b</span><span style="color:green;">0000</span> mov<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb18</span>]<span style="color:teal;"></span>,<span style="color:teal;"> eax</span>
<span style="color:teal;">| </span><span style="color:teal;">      `-&gt; </span><span style="color:green;">0x1000060ac</span>    <span style="color:gray;">8b93140b</span><span style="color:green;">0000</span> mov<span style="color:teal;"> edx</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb14</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060b2</span>    <span style="color:red;">ff</span><span style="color:gray;">c2</span>         <span style="color:olive;">inc</span><span style="color:teal;"> edx</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060b4</span>    <span style="color:gray;">8993140b</span><span style="color:green;">0000</span> mov<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb14</span>]<span style="color:teal;"></span>,<span style="color:teal;"> edx</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060ba</span>    <span style="color:gray;">488dbb300b0.</span> lea<span style="color:teal;"> rdi</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb30</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x100009a7d (fcn.100009a7d)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; DATA XREF from 0x100009a7d (fcn.100009a7d)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060c1</span>    <span style="color:gray;">488d35b5390.</span> lea<span style="color:teal;"> rsi</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rip</span>+<span style="color:teal;"></span><span style="color:olive;">0x439b5</span>]<span style="color:teal;"></span> ; 0x100009a7d 
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060c8</span>    <span style="color:gray;">31c0</span>         <span style="color:olive;">xor</span><span style="color:teal;"> eax</span>,<span style="color:teal;"> eax</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060ca</span>    <span style="color:gray;">e8</span><span style="color:teal;">7f</span><span style="color:gray;">0b04</span><span style="color:green;">00</span>   <span style="color:green;font-weight:bold;">call sym.imp.sprintf</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span>   sym.imp.sprintf(unk, unk, unk)
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060cf</span>    <span style="color:gray;">8b432c</span>       mov<span style="color:teal;"> eax</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0x2c</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060d2</span>    <span style="color:gray;">8983300c</span><span style="color:green;">0000</span> mov<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xc30</span>]<span style="color:teal;"></span>,<span style="color:teal;"> eax</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060d8</span>    <span style="color:gray;">f30f5a83100.</span> <span style="color:gray;">cvtss2sd</span><span style="color:teal;"> xmm0</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rbx</span>+<span style="color:teal;"></span><span style="color:olive;">0xb10</span>]<span style="color:teal;"></span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x1000150dc (fcn.1000150dc)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x100015159 (fcn.100015159)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; CODE (CALL) XREF from 0x100015117 (fcn.100015117)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:blue;">; DATA XREF from 0x1000150dc (fcn.1000150dc)</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060e0</span>    <span style="color:gray;">488d3df5eff.</span> lea<span style="color:teal;"> rdi</span>,<span style="color:teal;"> </span>[<span style="color:teal;">rip</span>-<span style="color:teal;"></span><span style="color:olive;">0x100b</span>]<span style="color:teal;"></span> ; 0x1000150dc 
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060e7</span>    <span style="color:gray;">4889de</span>       mov<span style="color:teal;"> rsi</span>,<span style="color:teal;"> rbx</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060ea</span>    <span style="color:gray;">4883c408</span>     <span style="color:olive;">add</span><span style="color:teal;"> rsp</span>,<span style="color:teal;"> </span><span style="color:olive;">0x8</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060ee</span>    <span style="color:gray;">5b</span>           <span style="color:purple;font-weight:bold;">pop</span><span style="color:teal;font-weight:bold;"> rbx</span>
<span style="color:teal;">| </span><span style="color:teal;">          </span><span style="color:green;">0x1000060ef</span>    <span style="color:gray;">5d</span>           <span style="color:purple;font-weight:bold;">pop</span><span style="color:teal;font-weight:bold;"> rbp</span>
<span style="color:teal;">\ </span><span style="color:teal;">          </span><span style="color:green;">0x1000060f0</span>    <span style="color:gray;">e94f1e</span><span style="color:green;">0000</span>   <span style="color:green;">jmp sym.__ZN2Fl14repeat_timeoutEdPFvPvES0_</span>


<span style="color:olive;">[0x10000606a]&gt;</span> 
</pre>

These are the lines in question. They're relatively easy to pick out since we're interested in a floating point operation:

    |           0x100006073    f30f5a83100. cvtss2sd xmm0, [rbx+0xb10]
    |           0x10000607b    f20f59050d1. mulsd xmm0, [rip+0x4180d]
    |           0x100006083    f20f5ac0     cvtsd2ss xmm0, xmm0
    |           0x100006087    f30f1183100. movss [rbx+0xb10], xmm0

Remember, we're looking for assembly code corresponding to `interval_ *= 0.95`. This fits. We:

* load `interval_` (from the address `rbx+0xb10`)
* multiply by a constant (at `rip+0x4180d`)
* and store the result back in `rbx+0xb10`

To make sure, we can check the value of the constant in `[rip+0x4180d]`. It should be `0.95`. We know it's an argument to mulsd, so we're looking for a [double](https://en.wikipedia.org/wiki/Double-precision_floating-point_format#IEEE_754_double-precision_binary_floating-point_format:_binary64). Recall `rip` points to the next instruction to be executed, so:

    [0x10000606a]> pf q @ 0x100006083+0x4180d
    0x100047890 = (qword) 0x3fee666666666666 

And indeed, if we [convert it to decimal](https://binaryconvert.com/result_double.html?hexadecimal=3FEE666666666666), we get `0.9499999...`.

Now all we need to do is find this code in the binary and [NOP](https://www.felixcloutier.com/x86/nop) it out. It's easy enough to search for the corresponding machine code in a hex editor since radare's disassembler has provided it alongside the assembly, and replace that section with `0x90`s.

Radare2 is a cool tool, though, and there's ways to do the patching entirely within the tool. [This tutorial](https://monosource.gitbooks.io/radare2-explorations/content/tut1/tut1_-_simple_patch.html) has an example.
