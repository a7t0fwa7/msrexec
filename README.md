<img src="https://imgur.com/nNnOCPK.png"/>

# MsrExec - Elevate Arbitrary WRMSR To Kernel Execution

msrexec is a small project that can be used to elevate arbitrary MSR writes to kernel execution on 64 bit Windows-10 systems. This project is part of the VDM (vulnerable driver manipulation) namespace
and can be integrated into any prior VDM projects. Although this project falls under the VDM namespace, Voyager and bluepill can be used to provide arbitrary wrmsr writes.

#### Features

* integration with VDM
* integration with Voyager and bluepill
* WARNING: does not work under most anti virus hypervisors or HVCI systems...
* Use any vulnerable driver which exposes arbitrary WRMSR to obtain kernel exeuction
* Works under KVA shadowing (you will still need to run as admin however to load the driver, LSTAR points to KiSystemCall64Shadow though and that is taken into consideration...)

# Syscall - Fast System Call

SYSCALL invokes an OS system-call handler at privilege level 0. It does so by ***loading RIP from the IA32_LSTAR MSR*** (after saving the address of the instruction following SYSCALL into RCX). (The WRMSR instruction ensures that the IA32_LSTAR MSR always contain a canonical address.)

SYSCALL also saves RFLAGS into R11 and then masks RFLAGS using the IA32_FMASK MSR (MSR address C0000084H); specifically, the processor clears in RFLAGS every bit corresponding to a bit that is set in the IA32_FMASK MSR.

SYSCALL loads the CS and SS selectors with values derived from bits 47:32 of the IA32_STAR MSR. However, the CS and SS descriptor caches are not loaded from the descriptors (in GDT or LDT) referenced by those selectors. Instead, the descriptor caches are loaded with fixed values. See the Operation section for details. It is the responsibility of OS software to ensure that the descriptors (in GDT or LDT) referenced by those selector values correspond to the fixed values loaded into the descriptor caches; the SYSCALL instruction does not ensure this correspondence.

***The SYSCALL instruction does not save the stack pointer (RSP).*** If the OS system-call handler will change the stack pointer, it is the responsibility of software to save the previous value of the stack pointer. This might be done prior to executing SYSCALL, with software restoring the stack pointer with the instruction following SYSCALL (which will be executed after SYSRET). Alternatively, the OS system-call handler may save the stack pointer and restore it before executing SYSRET.

# ROP - Return-Oriented Programming

ROP or return-oriented programming, is a technique where an attacker gains control of the call stack to hijack program control flow and then executes carefully chosen machine instruction sequences that are already present in the machine's memory, called "gadgets". Note: ***"The SYSCALL instruction does not save the stack pointer (RSP)"***. This allows for an attacker to setup the stack with addresses of ROP gadgets are specific values. In this situation SMEP and SMAP are two cpu protections which prevent an attacker from setting IA32_LSTAR to a user controlled page.

### SMEP - Supervisor Mode Execution Protection

SMEP or Supervisor Mode Execution Protection, prevents a logical processor with a lower CPL from executing code mapped into virtual memory with super supervisor bit set. This is relevant to this project as one could not simply set LSTAR to a user controlled page. However, with ROP one could disable SMEP by executing the following gadgets:

```nasm
pop rcx
ret
```

```nasm
mov cr4, rcx
ret
```

However, when the syscall instruction is executed, the address of the next instruction (the one after the syscall instruction) is placed into RCX. In order to preserve RIP, it should be placed onto the stack before any addresses of gadgets are placed onto the stack.

```nasm
lea rax, finish
push rax
```

changing IA32_LSTAR to a ROP chain as described above will work just fine on CPU's that done support SMAP. Windows 10 will use SMAP if your CPU supports it. This means RSP is unaccessable since it is a user controlled page. 

### SMAP - Supervisor Mode Access Prevention

SMAP or Supervisor Mode Access Prevention is a CPU protection which prevents accessing data controlled by a higher CPL. In other words, if SMAP is set in CR4, a logical
processor executing kernel code cannot access usermode controlled pages (user supervisor).

This is an issue with ROP as RSP after a syscall contains a usermode address. Interfacing with this usermode stack in any way will cause a fault. However, you can essentially disable SMAP from usermode. There is a bit in the RFLAGS register which can be set to nullify SMAP. The instruction to set this bit is called `STAC` (Set AC Flag in EFLAGS Register). However this instruction is privilaged and will throw a #UD. However as [@drew](https://twitter.com/drewbervisor) pointed out, you can `POPFQ` an RFLAGS value with that bit set and the CPU will not throw any exceptions. I assumed that since `STAC` cannot be used in usermode, that `POPFQ` would also throw an exception, however this is not the case... Again thank you [@drew](https://twitter.com/drewbervisor), without this key information the project would have been a complete mess as there are no useable `mov cr4, [non rax registers] ; ret` gadgets which exist across windows versions.

```nasm
pushfq                          ; thank you drew :)
pop rax                         ; this will set the AC flag in RFLAGS which "disables SMAP"...
or rax, 040000h                 ;
push rax                        ;
popfq                           ;
```

RFLAGS is restored after the syscall instruction. The original RFLAGS value is pushed onto the stack prior to all of the gadgets and other values.

```nasm
syscall                         ; LSTAR points at a pop rcx gadget... 
                                ; it will put m_smep_off into rcx...
finish:
popfq                           ; restore EFLAGS...
pop r10                         ; restore r10...
ret
```

<img src="https://imgur.com/OVC3LGH.png"/>

# Credit - Special Thanks

* [@drew](https://twitter.com/drewbervisor) - pointing out AC bit in RFLAGS can be set in usermode. I originally assumed since the `STAC` instruction could not be executed in usermode that `POPFQ` would throw an exception if AC bit was high and CPL was greater then zero. Without this key information the project would have been a complete mess. Thank you!
* [@0xnemi](https://twitter.com/0xnemi) / [@everdox](https://twitter.com/nickeverdox) - [mov ss/pop ss exploit](https://www.youtube.com/watch?v=iU_No7gdcwc) 0xnemi's use of syscall and the fact that RSP is not changed + use of ROP made me think about how there are alot of vulnerable drivers that expose arbitrary wrmsr which could be used to change LSTAR and effectivlly replicate his solution...
* [@Ch3rn0byl](https://twitter.com/notCh3rn0byl) - donation of a few vulnerable drivers which exposed arbitrary WRMSR/helped test with KVA shadowing enabled/disabled. 
* [@namazso](https://twitter.com/namazso) - originally hinting at this project many months ago. its finally done :)

# Lisence

TL;DR: if you use this project, rehost it, put it on github, include `_xeroxz` in your release.

```
BSD 3-Clause License

Copyright (c) 2021, _xeroxz
All rights reserved.

Redistribution and use in source and binary forms, with or without
modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this
  list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice,
  this list of conditions and the following disclaimer in the documentation
  and/or other materials provided with the distribution.

* Neither the name of the copyright holder nor the names of its
  contributors may be used to endorse or promote products derived from
  this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE
DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE
FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY,
OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
```