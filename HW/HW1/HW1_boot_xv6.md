# MIT6.828 HW1: boot xv6

HW1主要是了解xv6的启动过程，以及qemu的基本使用。

系统启动时，首先由BIOS完成硬件自检等一系列操作，然后从硬盘第一个扇区载入Boot Loader，并转交控制权。

## Boot Loader

xv6的Boot Loader由`bootasm.S`、`bootmain.c`组成，先来看`bootasm.S`。

此时系统尚处于16位实模式下，在切换至32位保护模式之前，需要做一些准备工作：关中断，寄存器清零，打开A20地址线（历史包袱）。

```assembly
# Start the first CPU: switch to 32-bit protected mode, jump into C.
# The BIOS loads this code from the first sector of the hard disk into
# memory at physical address 0x7c00 and starts executing in real mode
# with %cs=0 %ip=7c00.

.code16                       # Assemble for 16-bit mode
.globl start
start:
  cli                         # BIOS enabled interrupts; disable

  # Zero data segment registers DS, ES, and SS.
  xorw    %ax,%ax             # Set %ax to zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment
    # Physical address line A20 is tied to zero so that the first PCs 
  # with 2 MB would run software that assumed 1 MB.  Undo that.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```

下面代码中的`lgdt gdtdesc`指令将GDT表的首地址加载到GDTR寄存器。GDT表其实就是一个数组，数组元素为8个字节，而段选择子则是数组索引。这里的`gdt`定义了三个表项。

之后修改`cr0`，打开保护模式。

```assembly
# Switch from real to protected mode.  Use a bootstrap GDT that makes
  # virtual addresses map directly to physical addresses so that the
  # effective memory map doesn't change during the transition.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE, %eax
  movl    %eax, %cr0
  
  # Bootstrap GDT
.p2align 2                                # force 4 byte alignment
gdt:
  SEG_NULLASM                             # null seg
  SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)   # code seg
  SEG_ASM(STA_W, 0x0, 0xffffffff)         # data seg

gdtdesc:
  .word   (gdtdesc - gdt - 1)             # sizeof(gdt) - 1
  .long   gdt                             # address gdt
```

至此，保护模式已经打开。我们还需要用长跳转指令让系统开始使用保护模式。代码段选择子`SEG_KCODE<<3`，也就是8，指向GDT表的第二个表项。

```assembly
//PAGEBREAK!
  # Complete the transition to 32-bit protected mode by using a long jmp
  # to reload %cs and %eip.  The segment descriptors are set up with no
  # translation, so that the mapping is still the identity mapping.
  ljmp    $(SEG_KCODE<<3), $start32
```

接下来初始化段寄存器，设置栈指针，调用`bootmain`函数。

```assembly
.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain

  # If bootmain returns (it shouldn't), trigger a Bochs
  # breakpoint if running under Bochs, then loop.
  movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
  movw    %ax, %dx
  outw    %ax, %dx
  movw    $0x8ae0, %ax            # 0x8ae0 -> port 0x8a00
  outw    %ax, %dx
spin:
  jmp     spin
```
再来看`bootmain.c`是如何加载内核的。

这里的内核其实是一个ELF文件。ph是指向程序头表的指针，pa是各文件段的地址，依次调用`readseg`读入各文件段。

```c
void
bootmain(void)
{
  struct elfhdr *elf;
  struct proghdr *ph, *eph;
  void (*entry)(void);
  uchar* pa;

  elf = (struct elfhdr*)0x10000;  // scratch space

  // Read 1st page off disk
  readseg((uchar*)elf, 4096, 0);

  // Is this an ELF executable?
  if(elf->magic != ELF_MAGIC)
    return;  // let bootasm.S handle error

  // Load each program segment (ignores ph flags).
  ph = (struct proghdr*)((uchar*)elf + elf->phoff);
  eph = ph + elf->phnum;
  for(; ph < eph; ph++){
    pa = (uchar*)ph->paddr;
    readseg(pa, ph->filesz, ph->off);
    if(ph->memsz > ph->filesz)
      stosb(pa + ph->filesz, 0, ph->memsz - ph->filesz);
  }

  // Call the entry point from the ELF header.
  // Does not return!
  entry = (void(*)(void))(elf->entry);
  entry();
}
```

值得注意的是.bss段（未初始化的全局变量），`objdump -h kernel`可以看到.bss与.debug_line的File pff相同，意味着.bss的filesz为0，但我们扔需要在内存中为它分配空间。

![bss](/home/chaomin/6.828/HW/HW1/bss.png)

## Exercise: What is on the Stack?

![stack](/home/chaomin/6.828/HW/HW1/stack.png)

栈从高地址往下增长，有效的数值依次是0x7c4d，0x0000，0x7d8d。从`bootblock.asm`中我们可以找到答案：0x7c4d是`call  bootmain`的返回地址，0x0000是当时保存的ebp，0x7d8d则是调用entry()的返回地址。

