ページング
================================================================================

前書き
--------------------------------------------------------------------------------

`Linuxカーネルのブートプロセス`の[第5部](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)で、カーネルの初期段階において何をするかを学びました。次のステップでは、カーネルがinitプロセスをどのように実行するかを確認する前に、`initrd`のマウント、lockdepの初期化など、カーネルが様々な初期化を行います。
In the fifth [part](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html) of the series `Linux kernel booting process` we learned about what the kernel does in its earliest stage. In the next step the kernel will initialize different things like `initrd` mounting, lockdep initialization, and many many other things, before we can see how the kernel runs the first init process.

ええ、これらの多くの事柄がありますが、それらも**メモリ**で動作します。
Yeah, there will be many different things, but many many and once again many work with **memory**.

私の見解では、メモリ管理はLinuxカーネルの最も複雑な部分の1つであり、それは一般にシステムプログラミングで行われます。これが、カーネルの初期化に進む前に、ページングに精通する必要がある理由です。
In my view, memory management is one of the most complex parts of the Linux kernel and in system programming in general. This is why we need to get acquainted with paging, before we proceed with the kernel initialization stuff.

`ページング`はリニアアドレスから物理アドレスに変換するメカニズムです。本書の前のパートを読んでいるなら、セグメントレジスタを4シフトし、オフセットを追加して物理アドレスを計算すると、リアルモードでセグメンテーションが確認できたことを覚えているかもしれません。また、プロテクトモードでのセグメンテーションも確認しました。このモードでは、ディスクリプタテーブルとオフセット付きのディスクリプタからベースアドレスを使用して、物理アドレスを計算しました。これで、64ビットモードでのページングが表示されます。
`Paging` is a mechanism that translates a linear memory address to a physical address. If you have read the previous parts of this book, you may remember that we saw segmentation in real mode when physical addresses are calculated by shifting a segment register by four and adding an offset. 
We also saw segmentation in protected mode, where we used the descriptor tables and base addresses from descriptors with offsets to calculate the physical addresses. Now we will see paging in 64-bit mode.

Intelのマニュアルにあるように：
As the Intel manual says:

> ページングは、プログラムの実行環境のセクションが必要に応じて物理メモリにマップされる、従来のデマンドページな仮想メモリシステムを実装するためのメカニズムを提供します。
> Paging provides a mechanism for implementing a conventional demand-paged, virtual-memory system where sections of a program’s execution environment are mapped into physical memory as needed.

なので...この投稿では、ページングの背後にある理論を説明します。もちろん。それはLinuxカーネルの`x86_64`バージョンと密接に関連していますが、あまり詳細には触れません(少なくともこの投稿では)。
So... In this post I will try to explain the theory behind paging. Of course it will be closely related to the `x86_64` version of the Linux kernel, but we will not go into too much details (at least in this post).

ページングの有効化
Enabling paging
--------------------------------------------------------------------------------

3つのページングモードがあります。
There are three paging modes:

* 32ビットページング
* PAEページング
* IA-32eページング
* 32-bit paging;
* PAE paging;
* IA-32e paging.

ここでは、最後のモードのみを説明します。`IA-32eページング`モードを有効にするために、次のことを行う必要があります。
We will only explain the last mode here. To enable the `IA-32e paging` paging mode we need to do following things:

* `CR0.PG`ビットを設定します。
* `CR4.PAE`ビットを設定します。
* `IA32_EFER.LME`ビットを設定します。
* set the `CR0.PG` bit;
* set the `CR4.PAE` bit;
* set the `IA32_EFER.LME` bit.

これらのビットが[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)にて設定されていることは既に確認しました。
We already saw where those bits were set in [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S):

```assembly
movl	$(X86_CR0_PG | X86_CR0_PE), %eax
movl	%eax, %cr0
```

そして
and

```assembly
movl	$MSR_EFER, %ecx
rdmsr
btsl	$_EFER_LME, %eax
wrmsr
```

ページング機構
Paging structures
--------------------------------------------------------------------------------

ページングはリニアアドレス空間を固定サイズのページに分割します。ページは、物理アドレス空間、または外部ストレージにマッピングできます。この固定サイズは`x86_64`のLinuxカーネルでは`4096`バイトになります。リニアアドレスから物理アドレスへ変換するために、特別なデータ構造が使用されます。すべてのデータ構造は`4096`バイトで、`512`エントリ(これは`PAE`モードと`IA32_EFER.LME`モードのみ)あります。ページングのデータ構造(paging structure)は階層的であり、Linuxカーネルは、`x86_64`アーキテクチャにおいては4レベルのページング階層を利用します。CPUはリニアアドレスの一部を利用して、より低いレベルの他のページング構造体のエントリ、物理メモリ領域(`ページフレーム`)、またはこの領域の物理アドレス(`ページオフセット`)を識別します。トップレベルのページング構造体のアドレスは`cr3`に置きます。これはすでに[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)で確認済みです。
Paging divides the linear address space into fixed-size pages. Pages can be mapped into the physical address space or external storage. 
This fixed size is `4096` bytes for the `x86_64` Linux kernel. To perform the translation from linear address to physical address, special structures are used. 
Every structure is `4096` bytes and contains `512` entries (this only for `PAE` and `IA32_EFER.LME` modes).
Paging structures are hierarchical and the Linux kernel uses 4 level of paging in the `x86_64` architecture. 
The CPU uses a part of linear addresses to identify the entry in another paging structure which is at the lower level, physical memory region (`page frame`) or physical address in this region (`page offset`). 
The address of the top level paging structure located in the `cr3` register. 
We have already seen this in [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S):

```assembly
leal	pgtable(%ebx), %eax
movl	%eax, %cr3
```

ページテーブル構造体を作成し、トップレベルのページング構造体のアドレスを`cr3`レジスタに配置しています。`cr3`は、トップレベルのページング構造体のアドレス、`PML4`や`Page Global Directory`を格納するために使用されます。`cr3`は64ビットレジスタで、以下のような構造を持ちます。
We build the page table structures and put the address of the top-level structure in the `cr3` register. Here `cr3` is used to store the address of the top-level structure, the `PML4` or `Page Global Directory` as it is called in the Linux kernel. `cr3` is 64-bit register and has the following structure:

```
63                  52 51                                                        32
 --------------------------------------------------------------------------------
|                     |                                                          |
|    Reserved MBZ     |            Address of the top level structure            |
|                     |                                                          |
 --------------------------------------------------------------------------------
31                                  12 11            5     4     3 2             0
 --------------------------------------------------------------------------------
|                                     |               |  P  |  P  |              |
|  Address of the top level structure |   Reserved    |  C  |  W  |    Reserved  |
|                                     |               |  D  |  T  |              |
 --------------------------------------------------------------------------------
```

これらのフィールドは、以下の意味を持ちます。
These fields have the following meanings:

* ビット 63:52 - 予約済みで、0でなければなりません。
* ビット 51:12 - トップレベルのページング構造体のアドレスを保持します。
* ビット 11: 5 - 予約済みで、0でなければなりません。
* ビット 4 : 3 - PWT、またはページと、PCD、またはを意味します。これらのビットは、ハードウェアキャッシュによるページまたはページテーブルの処理方法を制御します。
* Bits 2 : 0 - 予約済みです。
* Bits 63:52 - reserved must be 0.
* Bits 51:12 - stores the address of the top level paging structure;
* Bits 11: 5 - reserved must be 0;
* Bits 4 : 3 - PWT or Page-Level Writethrough and PCD or Page-level cache disable indicate. These bits control the way the page or Page Table is handled by the hardware cache;
* Bits 2 : 0 - ignored;

リニアアドレス変換は以下の通りです。
The linear address translation is following:

* 与えられたリニアアドレスは、メモリバスに代わって[MMU](http://en.wikipedia.org/wiki/Memory_management_unit)に到着します。
* 64ビットリニアアドレスはいくつかに分割されます。下位48ビットのみが重要で、`2^48`、つまり256テラバイトのリニアアドレス空間にいつでもアクセスできることを意味します。
* `cr3`レジスタは4つのトップレベルのページング構造体のアドレスを保持します。
* 与えられたリニアアドレスの`47:39`ビットは、レベル4のページング構造体のインデックスを、`38:30`ビットはレベル3を、`29;21`ビットはレベル2を、`20:12`はレベル1をそれぞれ格納しており、`11:0`は物理ページへのオフセットをバイト単位で提供します。
`47:39` bits of the given linear address store an index into the paging structure level-4, `38:30` bits store index into the paging structure level-3, `29:21` bits store an index into the paging structure level-2, `20:12` bits store an index into the paging structure level-1 and `11:0` bits provide the offset into the physical page in byte.
* A given linear address arrives to the [MMU](http://en.wikipedia.org/wiki/Memory_management_unit) instead of memory bus.
* 64-bit linear address is split into some parts. Only low 48 bits are significant, it means that `2^48` or 256 TBytes of linear-address space may be accessed at any given time.
* `cr3` register stores the address of the 4 top-level paging structure.
* `47:39` bits of the given linear address store an index into the paging structure level-4, `38:30` bits store index into the paging structure level-3, `29:21` bits store an index into the paging structure level-2, `20:12` bits store an index into the paging structure level-1 and `11:0` bits provide the offset into the physical page in byte.

概略的には、次のようになります。
schematically, we can imagine it like this:

![4-level paging](images/4_level_paging.png)

リニアアドレスへのアクセスはすべて、スーパーバイザモード、もしくはユーザーモードアクセスのいずれかです。このアクセス種別は、`CPL`(Current Privilege Level)によって決定されます。`CPL < 3`の場合、スーパーバイザモードでのアクセスで、それ以外ならばユーザーモードのアクセスです。例えば、トップレベルのページテーブルエントリにはアクセスビットが含まれ、以下のような構造を持ちます。(ビットオフセットの定義は[arch/x86/include/asm/pgtable_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/pgtable_types.h)を参照してください。)
Every access to a linear address is either a supervisor-mode access or a user-mode access. This access is determined by the `CPL` (current privilege level). If `CPL < 3` it is a supervisor mode access level, otherwise it is a user mode access level. For example, the top level page table entry contains access bits and has the following structure (See [arch/x86/include/asm/pgtable_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/pgtable_types.h) for the bit offset definitions):

```
63  62                  52 51                                                    32
 --------------------------------------------------------------------------------
| N |                     |                                                     |
|   |     Available       |     Address of the paging structure on lower level  |
| X |                     |                                                     |
 --------------------------------------------------------------------------------
31                                              12 11  9 8 7 6 5   4   3 2 1     0
 --------------------------------------------------------------------------------
|                                                |     | M |I| | P | P |U|W|    |
| Address of the paging structure on lower level | AVL | B |G|A| C | W | | |  P |
|                                                |     | Z |N| | D | T |S|R|    |
 --------------------------------------------------------------------------------
```

Where:

* 63ビット - N/X ビット(No Execute:実行禁止ビット)テーブルエントリによってマッピングされた物理アドレスから、コードを実行する機能を示します。
* 62:52ビット - CPUによって無視され、システムソフトウェアによって使用されます。
* 51:12ビット - 下位レベルのページング構造体の物理アドレスを格納します。
* 11: 9ビット - CPUによって無視されます。
* MBZ - Must Be Zero. ゼロビットでなければなりません。
* IGN - 無視されます。
* A - アクセスビット。物理ページまたはページ構造体へアクセスされたことを示します。
* PWT and PCD - キャッシュのために使用されます。
* U/S - ユーザー/スーパーバイザビット。このテーブルエントリによってマップされた、すべての物理ページへのユーザーアクセスを制御します。
* R/W - 読み取り/書き込みビット。このテーブルエントリによってマッピングされたすべての物理ページへの読み取り/書き込みアクセスを制御します。
* P - プレゼント(Present)ビット。ページテーブルか物理ページが、メモリにロードされているかどうかを示します。
* 63 bit - N/X bit (No Execute Bit) which presents ability to execute the code from physical pages mapped by the table entry;
* 62:52 bits - ignored by CPU, used by system software;
* 51:12 bits - stores physical address of the lower level paging structure;
* 11: 9 bits - ignored by CPU;
* MBZ - must be zero bits;
* Ignored bits;
* A - accessed bit indicates was physical page or page structure accessed;
* PWT and PCD used for cache;
* U/S - user/supervisor bit controls user access to all the physical pages mapped by this table entry;
* R/W - read/write bit controls read/write access to all the physical pages mapped by this table entry;
* P - present bit. Current bit indicates was page table or physical page loaded into primary memory or not.

さて、ページング構造体とそのエントリについて学びました。それでは、Linuxカーネルの4レベルページングに関する詳細を見ていきましょう。
Ok, we know about the paging structures and their entries. Now let's see some details about 4-level paging in the Linux kernel.

Linuxカーネルのページング構造
Paging structures in the Linux kernel
--------------------------------------------------------------------------------

これまで見てきたように、`x86_64`のLinuxカーネルは4レベルページテーブルを利用しています。これらの名前は以下です。
As we've seen, the Linux kernel in `x86_64` uses 4-level page tables. Their names are:

* ページグローバルディレクトリ(Page Global Directory)
* ページアッパーディレクトリ(Page Upper  Directory)
* ページミドルディレクトリ(Page Middle Directory)
* ページテーブルエントリ(Page Table Entry)
* Page Global Directory
* Page Upper  Directory
* Page Middle Directory
* Page Table Entry

Linuxカーネルをコンパイルしてインストールした後、カーネルで使用される関数の仮想ドレスを`System.map`ファイルにて確認できます。例えば、
After you've compiled and installed the Linux kernel, you can see the `System.map` file which stores the virtual addresses of the functions that are used by the kernel. For example:

```
$ grep "start_kernel" System.map
ffffffff81efe497 T x86_64_start_kernel
ffffffff81efeaa2 T start_kernel
```

ここでは、`0xffffffff81efe497`とアドレスが記載されています。そんなに大きなRAMが本当に取り付けられているとは思えません。しかし、`start_kernel`と`x86_64_start_kernel`は実行されるでしょう。`x86_64`のアドレス空間のサイズは`2^64`ですが、大きすぎるため、48ビット幅の小さなアドレス空間が使用されます。そのため、物理アドレス空間としては48ビットに制限していますが、アドレッシングは64ビットポインタで行います。どうやってこの問題を解決しているのでしょうか？以下の図を見てください。
We can see `0xffffffff81efe497` here. I doubt you really have that much RAM installed. But anyway, `start_kernel` and `x86_64_start_kernel` will be executed. The address space in `x86_64` is `2^64` wide, but it's too large, that's why a smaller address space is used, only 48-bits wide. So we have a situation where the physical address space is limited to 48 bits, but addressing still performs with 64 bit pointers. How is this problem solved? Look at this diagram:

```
0xffffffffffffffff  +-----------+
                    |           |
                    |           | Kernelspace
                    |           |
0xffff800000000000  +-----------+
                    |           |
                    |           |
                    |   hole    |
                    |           |
                    |           |
0x00007fffffffffff  +-----------+
                    |           |
                    |           |  Userspace
                    |           |
0x0000000000000000  +-----------+
```

この解決方法は、`符号拡張(sign extension)`です。仮想アドレスの下位48ビットがアドレッシングに利用されます。`63:48`ビットはすべて0またはすべて1のみです。仮想アドレス空間は2つの部分に分かれることに注意してください。
This solution is `sign extension`. Here we can see that the lower 48 bits of a virtual address can be used for addressing. Bits `63:48` can be either only zeroes or only ones. Note that the virtual address space is split into 2 parts:

* カーネル空間(Kernel space)
* ユーザー空間(Userspace)
* Kernel space
* Userspace

ユーザー空間は仮想アドレス空間の下位部分、具体的には`0x000000000000000`から`0x00007fffffffffff`、を占有し、カーネル空間は上位部分、具体的には`0xffff8000000000`から`0xffffffffffffffff`、を占有しています。`63:48`ビットは、ユーザー空間では0、カーネル空間では1であることに注意してください。カーネル空間とユーザー空間のすべてのアドレス、言い換えると上位`63:48`ビットがすべて0か1であるアドレスを、`カノニカル(canonical)`アドレスと呼びます。なお、これらのメモリ領域の間には、`非カノニカル(non-cannonical)`領域が存在します。これら2つのメモリ領域(カーネル空間とユーザー空間)を合わせたビット幅は、確かに`2^48`となります。なお、[Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)にて、4レベルページテーブル仮想メモリマップの説明があります。
Userspace occupies the lower part of the virtual address space, from `0x000000000000000` to `0x00007fffffffffff` and kernel space occupies the highest part from `0xffff8000000000` to `0xffffffffffffffff`. Note that bits `63:48` is 0 for userspace and 1 for kernel space. All addresses which are in kernel space and in userspace or in other words which higher `63:48` bits are zeroes or ones are called `canonical` addresses. There is a `non-canonical` area between these memory regions. Together these two memory regions (kernel space and user space) are exactly `2^48` bits wide. We can find the virtual memory map with 4 level page tables in the [Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt):

```
0000000000000000 - 00007fffffffffff (=47 bits) user space, different per mm
hole caused by [48:63] sign extension
ffff800000000000 - ffff87ffffffffff (=43 bits) guard hole, reserved for hypervisor
ffff880000000000 - ffffc7ffffffffff (=64 TB) direct mapping of all phys. memory
ffffc80000000000 - ffffc8ffffffffff (=40 bits) hole
ffffc90000000000 - ffffe8ffffffffff (=45 bits) vmalloc/ioremap space
ffffe90000000000 - ffffe9ffffffffff (=40 bits) hole
ffffea0000000000 - ffffeaffffffffff (=40 bits) virtual memory map (1TB)
... unused hole ...
ffffec0000000000 - fffffc0000000000 (=44 bits) kasan shadow memory (16TB)
... unused hole ...
ffffff0000000000 - ffffff7fffffffff (=39 bits) %esp fixup stacks
... unused hole ...
ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
ffffffffa0000000 - ffffffffff5fffff (=1525 MB) module mapping space
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
ffffffffffe00000 - ffffffffffffffff (=2 MB) unused hole
```

ここには、ユーザー空間、カーネル空間、非カノニカル領域のメモリマップが記載されています。ユーザー空間のマッピングはシンプルなので、カーネル空間を詳しく見てみましょう。ハイパーバイザのために予約されている、ガードホール(guard hole)から始まっていることがわかります。ガードホールは、[arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/page_64_types.h)にて以下のように定義されています。
We can see here the memory map for user space, kernel space and the non-canonical area in-between them. The user space memory map is simple. Let's take a closer look at the kernel space. We can see that it starts from the guard hole which is reserved for the hypervisor. We can find the definition of this guard hole in [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/page_64_types.h):

```C
#define __PAGE_OFFSET _AC(0xffff880000000000, UL)
```

以前は、このガードホールと`__PAGE_OFFSET`は、非カノニカル領域へアクセスさせないために、`0xffff800000000000`から`0xffff87ffffffffff`のアドレスとなっていましたが、のちにハイパーバイザのために3ビット拡張されました。
Previously this guard hole and `__PAGE_OFFSET` was from `0xffff800000000000` to `0xffff87ffffffffff` to prevent access to non-canonical area, but was later extended by 3 bits for the hypervisor.

次は、カーネル空間で使用可能な最低アドレス、つまり`ffff880000000000`となります。この仮想メモリ領域はすべての物理メモリのダイレクトマッピング用です。すべての物理アドレスをマッピングするメモリ空間の後に、ガードホールがあります。これは、すべての物理メモリのダイレクトマッピングとvmalloc領域の間にある必要があります。そして、64テラバイトの仮想メモリマップと未使用のホールの後に、`kasan`シャドウメモリがあります。これは、[commit](https://github.com/torvalds/linux/commit/ef7f0d6a6ca8c9e4b27d78895af86c2fbfaeedb2)にて追加され、カーネルアドレスサニタイザ(Kernel Address SANitizer)を提供します。そして、未使用のホールの後、`esp`修正スタック(本書の他の部分で説明します)と、物理アドレスからのカーネルテキストマッピングの開始アドレス(つまり物理アドレス上では0)があります。このアドレス定義は、`__PAGE_OFFSET`と同じファイルにあります。
Next is the lowest usable address in kernel space - `ffff880000000000`. This virtual memory region is for direct mapping of all the physical memory. 
After the memory space which maps all the physical addresses, the guard hole. It needs to be between the direct mapping of all the physical memory and the vmalloc area. 
After the virtual memory map for the first terabyte and the unused hole after it, we can see the `kasan` shadow memory. It was added by [commit](https://github.com/torvalds/linux/commit/ef7f0d6a6ca8c9e4b27d78895af86c2fbfaeedb2) and provides the kernel address sanitizer. After the next unused hole we can see the `esp` fixup stacks (we will talk about it in other parts of this book) and the start of the kernel text mapping from the physical address - `0`. We can find the definition of this address in the same file as the `__PAGE_OFFSET`:

```C
#define __START_KERNEL_map      _AC(0xffffffff80000000, UL)
```

通常、カーネルの`.text`は、このアドレスに`CONFIG_PHYSICAL_START`オフセットを加味して開始します。 [ELF64](https://github.com/0xAX/linux-insides/blob/master/Theory/ELF.md)に関する説明でそれを見ました。
Usually kernel's `.text` starts here with the `CONFIG_PHYSICAL_START` offset. We have seen it in the post about [ELF64](https://github.com/0xAX/linux-insides/blob/master/Theory/ELF.md):

```
readelf -s vmlinux | grep ffffffff81000000
     1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1 
 65099: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 _text
 90766: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 startup_64
```

上記結果で、`vmlinux`の`CONFIG_PHYSICAL_START`が`0x1000000`であることを確認します。カーネルの`.text`の開始アドレスが`0xffffffff80000000`であるので、オフセットが`0x1000000`であれば、結果的に仮想アドレスは`0xffffffff80000000 + 1000000 = 0xffffffff81000000`となります。
Here I check `vmlinux` with `CONFIG_PHYSICAL_START` is `0x1000000`. So we have the start point of the kernel `.text` - `0xffffffff80000000` and offset - `0x1000000`, the resulted virtual address will be `0xffffffff80000000 + 1000000 = 0xffffffff81000000`.

カーネルの`.text`領域の後には、カーネルモジュール用の仮想メモリ領域、`vsyscalls`、2メガバイトの未使用のホールがあります。
After the kernel `.text` region there is the virtual memory region for kernel module, `vsyscalls` and an unused hole of 2 megabytes.

カーネルの仮想メモリマップがどのようにレイアウトされ、仮想アドレスがどのように物理アドレスに変換されるかを見てきました。例として次のアドレスを見てみましょう。
We've seen how virtual memory map in the kernel is laid out and how a virtual address is translated into a physical one. Let's take the following address as example:

```
0xffffffff81000000
```

二進数表記では：
In binary it will be:

```
1111111111111111 111111111 111111110 000001000 000000000 000000000000
      63:48        47:39     38:30     29:21     20:12      11:0
```

この仮想アドレスは、上記のように部分的に分割されています。
This virtual address is split in parts as described above:

* `63:48` - 未使用ビットです。
* `47:39` - レベル4のページテーブルのインデックスです。
* `38:30` - レベル3のページテーブルのインデックスです。
* `29:21` - レベル2のページテーブルのインデックスです。
* `20:12` - レベル1のページテーブルのインデックスです。
* `11:0`  - 物理ページへのオフセットをバイト単位で表しています。
* `63:48` - bits not used;
* `47:39` - bits store an index into the paging structure level-4;
* `38:30` - bits store index into the paging structure level-3;
* `29:21` - bits store an index into the paging structure level-2;
* `20:12` - bits store an index into the paging structure level-1;
* `11:0`  - bits provide the offset into the physical page in byte.

以上です。これで`ページング`の理論について少し理解できたので、実際にカーネルのソースコードをみて、最初の初期化ステップを見ていきましょう。
That is all. Now you know a little about theory of `paging` and we can go ahead in the kernel source code and see the first initialization steps.

結論
--------------------------------------------------------------------------------

ページング理論に関する短い説明は終わりです。もちろん、この投稿ではページングのすべての詳細をカバーしているわけではありませんが、Linuxカーネルがページング機構をどのように構築し、どのように機能するかを実際に見ていきます。
It's the end of this short part about paging theory. Of course this post doesn't cover every detail of paging, but soon we'll see in practice how the Linux kernel builds paging structures and works with them.

英語は私の第一言語ではありません。ご不便をおかけして申し訳ありません。間違いを見つけた場合は、[linux-insides](https://github.com/0xAX/linux-insides)に PRを送ってください。
**Please note that English is not my first language and I am really sorry for any inconvenience. If you've found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

リンク集
--------------------------------------------------------------------------------

* [Paging on Wikipedia](http://en.wikipedia.org/wiki/Paging)
* [Intel 64 and IA-32 architectures software developer's manual volume 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [MMU](http://en.wikipedia.org/wiki/Memory_management_unit)
* [ELF64](https://github.com/0xAX/linux-insides/blob/master/Theory/ELF.md)
* [Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)
* [Last part - Kernel booting process](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html)
