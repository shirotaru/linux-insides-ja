Kernel booting process. Part 2.
================================================================================

カーネルセットアップの最初のステップ
--------------------------------------------------------------------------------

<<<<<<< HEAD
前回の[パート](linux-bootstrap-1.md)では、Linuxカーネルの中へ潜り始め、カーネルをセットアップするコードの初期の部分を見ていきました。
私たちは、[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)内の `main`関数（C言語で書かれた最初の関数）を初めて呼び出すところで止まっていました。

このパートでは、引き続きカーネルのセットアップコードについての調査と以下の内容をやります。

* `プロテクトモード` が何なのか、
* `プロテクトモード` に入るための準備、
* ヒープとコンソールの初期化、
* メモリの検出、cpu validation、キーボードの初期化
* その他いろいろ
=======
We started to dive into the linux kernel's insides in the previous [part](linux-bootstrap-1.md) and saw the initial part of the kernel setup code. We stopped at the first call to the `main` function (which is the first function written in C) from [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c).

In this part, we will continue to research the kernel setup code and go over
* what `protected mode` is,
* the transition into it,
* the initialization of the heap and the console,
* memory detection, CPU validation and keyboard initialization
* and much much more.

So, let's go ahead.
>>>>>>> upstream/master

では、やっていきましょう。

プロテクトモード
--------------------------------------------------------------------------------

ネイティブの Intel64 [ロングモード](http://en.wikipedia.org/wiki/Long_mode) に切り替える前に、 カーネルはCPUをプロテクトモードに切り替える必要があります。

[プロテクトモード](https://en.wikipedia.org/wiki/Protected_mode)とは何でしょう?
プロテクトモードが最初に x86アーキテクチャ に追加されたのは1982年で、
このモードは[80286](http://en.wikipedia.org/wiki/Intel_80286)プロセッサが出てから、Intel 64とロングモードが登場するまでは、主要のモードでした。

<<<<<<< HEAD
[リアルモード](http://wiki.osdev.org/Real_Mode)から移行した主な理由は、RAMへのアクセスが非常に制限されていたからです。
前回のパートの内容を覚えているかもしれませんが、リアルモードで利用可能なRAMはせいぜい2<sup>20</sup>byteか1MBで、中には640KBしかないものもあります。

プロテクトモードになって、多くの点が変わりましたが、メモリ管理で最も大きな変更がありました。
20bitアドレスバスが32bitアドレスバスに置き換えられたことで、リアルモードでは1MBのメモリにしかアクセスできなかったのが、4GBのメモリにアクセスが可能になりました。
さらにプロテクトモードは、[ページング](http://en.wikipedia.org/wiki/Paging)にもまた対応しています。これについては、次のセクションで紹介します。
=======
The main reason to move away from [Real mode](http://wiki.osdev.org/Real_Mode) is that there is very limited access to the RAM. As you may remember from the previous part, there are only 2<sup>20</sup> bytes or 1 Megabyte, sometimes even only 640 Kilobytes of RAM available in Real mode.

Protected mode brought many changes, but the main one is the difference in memory management. The 20-bit address bus was replaced with a 32-bit address bus. It allowed access to 4 Gigabytes of memory vs the 1 Megabyte in Real mode. Also, [paging](http://en.wikipedia.org/wiki/Paging) support was added, which you can read about in the next sections.
>>>>>>> upstream/master

プロテクトモードにおけるメモリ管理は、ほぼ独立した次の2つの方式に分かれます。:

* セグメンテーション
* ページング

<<<<<<< HEAD
セグメンテーションにのみここでは見ていきます。ページングに関しては次のセクションで見ていきましょう。

あなたは前のパートでリアルモードのアドレスは２つのパートで構成されると学びました:
=======
Here we will only talk about segmentation. Paging will be discussed in the next sections.

As you can read in the previous part, addresses consist of two parts in Real mode:
>>>>>>> upstream/master

* セグメントのベースアドレス
* セグメントベースからのオフセット

そして、これらの2つの情報が分かれば、物理アドレスを取得することができます:

```
PhysicalAddress = Segment Base * 16 + Offset
```

<<<<<<< HEAD
プロテクトモードになって、メモリセグメンテーションが一新され、64KBの固定サイズのセグメントがなくなりました。
その代わりに、各セグメントのサイズと位置は、セグメントディスクリプタと呼ばれる一連のデータ構造体で表現されます。
このセグメントディスクリプタが格納されているのが、`Global Descriptor Table`（GDT）というデータ構造体です。

GDTはメモリ内にある構造体です。GDTの場所はメモリ内で固定されているわけではなく、専用の `GDTR`レジスタにアドレスが格納されています。
LinuxカーネルのコードでGDTを読み込む方法については、後ほど見ていきましょう。以下のようにGDTは、メモリに読み込まれます:
=======
Memory segmentation was completely redone in protected mode. There are no 64 Kilobyte fixed-size segments. Instead, the size and location of each segment is described by an associated data structure called the _Segment Descriptor_. These segment descriptors are stored in a data structure called the `Global Descriptor Table` (GDT).

The GDT is a structure which resides in memory. It has no fixed place in the memory, so its address is stored in the special `GDTR` register. Later we will see how the GDT is loaded in the Linux kernel code. There will be an operation for loading it from memory, something like:
>>>>>>> upstream/master

```assembly
lgdt gdt
```

<<<<<<< HEAD
この`lgdt`命令で、`GDTR`レジスタ にベースアドレスとグローバルディスクリプタテーブルの制限（サイズ）を読み込みます。
`GDTR` は48bitのレジスタで、以下の2つの部分から構成されています:

 * グローバルディスクリプタテーブルのサイズ（16bit）
 * グローバルディスクリプタテーブルのアドレス（32bit）

先ほど説明したように、GDTにはメモリセグメントを表す `segment descriptors` が含まれています。
各ディスクリプタのサイズは64bitで、ディスクリプタの一般的な配置は次のようになっています:
=======
where the `lgdt` instruction loads the base address and limit(size) of the global descriptor table to the `GDTR` register. `GDTR` is a 48-bit register and consists of two parts:

 * the size(16-bit) of the global descriptor table;
 * the address(32-bit) of the global descriptor table.

As mentioned above, the GDT contains `segment descriptors` which describe memory segments.  Each descriptor is 64-bits in size. The general scheme of a descriptor is:
>>>>>>> upstream/master

```
 63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

<<<<<<< HEAD
リアルモードの後でこれを見ると、少し怖いかもしれませんが、これは簡単です。
例えば、LIMIT 15:0というのは、ディスクリプタのbit 0–15に制限の値が含まれていることを意味します。
そして残りはLIMIT 19:16の中にあります。よってリミットのサイズは0–19なので、20bitです。詳しく見てみましょう。:

1. Limit[20bit]は0–15と16–19のbitにあります。これは`length_of_segment – 1`を定義し、`G`（粒度）ビットに依存します。

  * `G`（bit 55）とセグメントリミットが0の場合、セグメントのサイズは1byteです。
  * `G` が1でセグメントリミットが0の場合、セグメントのサイズは4096byteです。
  * `G` が0でセグメントリミットが0xfffffの場合、セグメントのサイズは1MBです。
  * `G` が1でセグメントリミットが0xfffffの場合、セグメントのサイズは4GBです。

  つまり以下のようになります。

  * `G` が0なら、Limitは1バイト単位と見なされ、セグメントの最大サイズは1MBになります。
  * `G` が1なら、Limitは4096バイト ＝ 4KB ＝ 1ページ単位と見なされ、セグメントの最大サイズは4GBになります。実際、`G`が1なら、LIMITの値は1bit分左にずれます。つまり20bit ＋ 12bitで32bit、すなわち2<sup>32</sup> ＝ 4GBになります。

2. Base[32bit]は（0–15、32–39、56–63bit）にあり、これはセグメントの開始位置の物理アドレスを定義します。

3. タイプ/属性（40–47bit）はセグメントのタイプとセグメントに対する種々のアクセスについて定義します
  * bit 44の `S`フラグはディスクリプタのタイプを指定します。`S`が0ならこのセグメントはシステムセグメントで、`S`が1ならコードまたはデータのセグメントになります（スタックセグメントはデータセグメントで、これは読み書き可能なセグメントである必要があります）。

このセグメントがコードとデータ、どちらのセグメントなのかを判別するには、以下の図で0と表記されたEx(bit 43)属性を確認します。
これが0ならセグメントはデータセグメントで、1ならコードセグメントになります。
=======
Don't worry, I know it looks a little scary after Real mode, but it's easy. For example LIMIT 15:0 means that bits 0-15 of the segment limit are located at the beginning of the Descriptor. The rest of it is in LIMIT 19:16, which is located at bits 48-51 of the Descriptor. So, the size of Limit is 0-19 i.e 20-bits. Let's take a closer look at it:

1. Limit[20-bits] is split between bits 0-15 and 48-51. It defines the `length_of_segment - 1`. It depends on the `G`(Granularity) bit.

  * if `G` (bit 55) is 0 and the segment limit is 0, the size of the segment is 1 Byte
  * if `G` is 1 and the segment limit is 0, the size of the segment is 4096 Bytes
  * if `G` is 0 and the segment limit is 0xfffff, the size of the segment is 1 Megabyte
  * if `G` is 1 and the segment limit is 0xfffff, the size of the segment is 4 Gigabytes

  So, what this means is
  * if G is 0, Limit is interpreted in terms of 1 Byte and the maximum size of the segment can be 1 Megabyte.
  * if G is 1, Limit is interpreted in terms of 4096 Bytes = 4 KBytes = 1 Page and the maximum size of the segment can be 4 Gigabytes. Actually, when G is 1, the value of Limit is shifted to the left by 12 bits. So, 20 bits + 12 bits = 32 bits and 2<sup>32</sup> = 4 Gigabytes.

2. Base[32-bits] is split between bits 16-31, 32-39 and 56-63. It defines the physical address of the segment's starting location.

3. Type/Attribute[5-bits] is represented by bits 40-44. It defines the type of segment and how it can be accessed.
  * The `S` flag at bit 44 specifies the descriptor type. If `S` is 0 then this segment is a system segment, whereas if `S` is 1 then this is a code or data segment (Stack segments are data segments which must be read/write segments).

To determine if the segment is a code or data segment, we can check its Ex(bit 43) Attribute (marked as 0 in the above diagram). If it is 0, then the segment is a Data segment, otherwise, it is a code segment.
>>>>>>> upstream/master

セグメントは以下のいずれかのタイプになります:

```
--------------------------------------------------------------------------------------
|           Type Field        | Descriptor Type | Description                        |
|-----------------------------|-----------------|------------------------------------|
| Decimal                     |                 |                                    |
|             0    E    W   A |                 |                                    |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
|                  C    R   A |                 |                                    |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
--------------------------------------------------------------------------------------
```

<<<<<<< HEAD
見て取れるように最初のビット（bit 43）は、_data_ セグメントの場合は`0`で、_code_ セグメントの場合は`1`です。
続く3つのbit(40, 41,42)は`EWA`(*E*xpansion *W*ritable *A*ccessible)またはCRA(*C*onforming *R*eadable *A*ccessible)のどちらかになります。

  *E（bit 42）が0なら上に拡張し、1なら下に拡張します。詳細は[こちら](http://www.sudleyplace.com/dpmione/expanddown.html)を参照してください。
  *W（bit 41）（データセグメントの場合）が1なら書き込みアクセスが可能、0なら不可です。データセグメントでは、読み取りアクセスが常に許可されている点に注目してください。
  *A（bit 40）はプロセッサからセグメントへのアクセス可能か否かを示します。
  *C（bit 43）（コードセレクタの場合）はコンフォーミングビットです。Cが1なら、ユーザレベルなどの下位レベルの権限から、セグメントコードを実行することが可能です。Cが0なら、同じ権限レベルからのみ実行可能です。
  *R（bit 41）（コードセグメントの場合）が1なら、セグメントへの読み取りアクセスが可能、0なら不可です。コードセグメントに対して、書き込みアクセスは一切できません。

4. DPL[2-bits]はbits 45 – 46にあります。これはセグメントの特権レベルを定義し、値は0-3で、0が最も権限があります。

5. Pフラグ（bit 47）は、セグメントがメモリ内にあるか否かを示します。Pが0なら、セグメントは無効であることを意味し、プロセッサはこのセグメントの読み取りを拒否します。
=======
As we can see the first bit(bit 43) is `0` for a _data_ segment and `1` for a _code_ segment. The next three bits (40, 41, 42) are either `EWA`(*E*xpansion *W*ritable *A*ccessible) or CRA(*C*onforming *R*eadable *A*ccessible).
  * if E(bit 42) is 0, expand up, otherwise, expand down. Read more [here](http://www.sudleyplace.com/dpmione/expanddown.html).
  * if W(bit 41)(for Data Segments) is 1, write access is allowed, and if it is 0, the segment is read-only. Note that read access is always allowed on data segments.
  * A(bit 40) controls whether the segment can be accessed by the processor or not.
  * C(bit 43) is the conforming bit(for code selectors). If C is 1, the segment code can be executed from a lower level privilege (e.g. user) level. If C is 0, it can only be executed from the same privilege level.
  * R(bit 41) controls read access to code segments; when it is 1, the segment can be read from. Write access is never granted for code segments.

4. DPL[2-bits] (Descriptor Privilege Level) comprises the bits 45-46. It defines the privilege level of the segment. It can be 0-3 where 0 is the most privileged level.

5. The P flag(bit 47) indicates if the segment is present in memory or not. If P is 0, the segment will be presented as _invalid_ and the processor will refuse to read from this segment.
>>>>>>> upstream/master

6. AVLフラグ（bit 52）は利用可能な予約ビットで、Linuxにおいては無視されます。

<<<<<<< HEAD
7. Lフラグ（ビット53）は、コードセグメントがネイティブ64ビットコードを含んでいるかを示します。1ならコードセグメントは64ビットモードで実行されます。

8. D/Bフラグ（ビット54）は、デフォルト/ビッグフラグで、例えば16/32ビットのようなオペランドのサイズを表します。フラグがセットされていれば32ビット、そうでなければ16ビットです。
=======
7. The L flag(bit 53) indicates whether a code segment contains native 64-bit code. If it is set, then the code segment executes in 64-bit mode.

8. The D/B flag(bit 54)  (Default/Big flag) represents the operand size i.e 16/32 bits. If set, operand size is 32 bits. Otherwise, it is 16 bits.
>>>>>>> upstream/master

セグメントレジスタには、リアルモードのようにセグメントセレクタが含まれていません。
しかし、プロテクトモードでは、セグメントレジスタの扱いは異なります。各セグメントディスクリプタは
関連する16ビットの構造体であるSegment Selectorを持ちます。:

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

<<<<<<< HEAD
* **Index** がGDTにおけるディスクリプタのインデックス番号を示します。
* **TI**(Table Indicator) はディスクリプタを探す場所を示します。0ならば、Global Descriptor Table（GDT）内を検索し、そうでない場合は、Local Descriptor Table（LDT）内を調べます。
* **RPL** は、Requester’s Privilege Levelのことです。

すべてのセグメントレジスタは見える部分と隠れた部分を持っています。

* Visible – セグメントセレクタはここに保存されています。
* Hidden – セグメントディスクリプタ（ベース、制限、属性、フラグ）

以下のステップは、プロテクトモードで物理アドレスを取得するのに要する手順です。:
=======
Where,
* **Index** stores the index number of the descriptor in the GDT.
* **TI**(Table Indicator) indicates where to search for the descriptor. If it is 0 then the descriptor is searched for in the Global Descriptor Table(GDT). Otherwise, it will be searched for in the Local Descriptor Table(LDT).
* And **RPL** contains the Requester's Privilege Level.

Every segment register has a visible and a hidden part.
* Visible - The Segment Selector is stored here.
* Hidden -  The Segment Descriptor (which contains the base, limit, attributes & flags) is stored here.

The following steps are needed to get a physical address in protected mode:

* The segment selector must be loaded in one of the segment registers.
* The CPU tries to find a segment descriptor at the offset `GDT address + Index` from the selector and then loads the descriptor into the *hidden* part of the segment register.
* If paging is disabled, the linear address of the segment, or its physical address, is given by the formula: Base address (found in the descriptor obtained in the previous step) + Offset.
>>>>>>> upstream/master

* セグメントセレクタはセグメントレジスタの1つにロードしなければなりません。
* CPUは、GDTアドレスとセレクタのIndexによってセグメントディスクリプタを特定し、ディスクリプタをセグメントレジスタの *隠れた* 部分にロードしようとします。
* ベースアドレス（セグメントディスクリプタから）+オフセットは、物理アドレスであるセグメントのリニアアドレスになります。（ページングが無効の場合）

図で表すとこうなります:

![linear address](images/linear_address.png)

リアルモードからプロテクトモードへ移行するためのアルゴリズムは、

<<<<<<< HEAD
* 割り込みを無効にします。
* `lgdt`命令でGDTを記述、ロードします。
* CR0（コントロールレジスタ0）におけるPE（Protection Enable、プロテクト有効化）ビットを設定します。
* プロテクトモードのコードにジャンプします。
=======
* Disable interrupts
* Describe and load the GDT with the `lgdt` instruction
* Set the PE (Protection Enable) bit in CR0 (Control Register 0)
* Jump to protected mode code
>>>>>>> upstream/master

次の章でlinuxカーネル内で完璧にプロテクトモードへの移行をします。ただ、プロテクトモードへ移る前にもう少し準備が必要です。

<<<<<<< HEAD
[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c)を見てみましょう。キーボード初期化、ヒープ初期化などを実行する部分があるのがわかります。よく見てみましょう。
=======
Let's look at [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). We can see some routines there which perform keyboard initialization, heap initialization, etc... Let's take a look.
>>>>>>> upstream/master

ブートパラメータを ”zeropage” にコピー
--------------------------------------------------------------------------------

<<<<<<< HEAD
“main.c”の`main`ルーティンから始めましょう。`main`の中で最初に呼び出される関数は[`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L30)です。
これは、カーネル設定ヘッダを、[arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L113)にて定義された`boot_params`構造体のフィールドにコピーします。

`boot_params`構造体は、`struct setup_header hdr`フィールドを持っています。この構造体は[linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)で定義されているのと同じフィールドを持ち、
ブートローダによって、カーネルのコンパイル/ビルド時に書き込まれます。`copy_boot_params`は2つのことを実行します:

1. `hdr`を[header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L281)から`setup_header`フィールド内の`boot_params`構造体へコピー

2. カーネルが古いコマンドラインプロトコルでロードされた場合に、ポインタをカーネルのコマンドラインに更新

`hdr`を、[copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S)で定義されている`memcpy`関数と一緒にコピーしていることに注目してください。中身を見てみましょう:
=======
We will start from the `main` routine in "main.c". The first function which is called in `main` is [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). It copies the kernel setup header into the corresponding field of the `boot_params` structure which is defined in the [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) header file.

The `boot_params` structure contains the `struct setup_header hdr` field. This structure contains the same fields as defined in the [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) and is filled by the boot loader and also at kernel compile/build time. `copy_boot_params` does two things:

1. It copies `hdr` from [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280) to the `setup_header` field in `boot_params` structure.

2. It updates the pointer to the kernel command line if the kernel was loaded with the old command line protocol.

Note that it copies `hdr` with the `memcpy` function, defined in the [copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S) source file. Let's have a look inside:
>>>>>>> upstream/master

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

<<<<<<< HEAD
わたしたちは、Cのコードに移ってきたばかりですが、またアセンブリをみます。:) まず初めに、`memcpy`とここで定義されている他のルーティンが2つのマクロで挟まれており、`GLOBAL`で始まって`ENDPROC`で終わっているのに気づきます。`GLOBAL`は、`globl`のディレクティブとそのためのラベルを定義する[arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/linkage.h)に記述されています。
`ENDPROC`は、`name`シンボルを関数名としてマークアップし`name`シンボルのサイズで終わる[include/linux/linkage.h](https://github.com/torvalds/linux/blob/master/include/linux/linkage.h)に記述されています。

`memcpy`の実装は簡単です。まず、`si`と`di`レジスタの値をスタックにプッシュします。これらの値は`memcpy`の実行中に変化するので、スタックにプッシュして値を保存すします。`memcpy`（に加え、[copy.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S)内の他の関数）は`fastcall`呼び出しのため、パラメータを`ax`、`dx`そして`cx`レジスタから取得します。
`memcpy`の呼び出しは次のように表示されます:
=======
Yeah, we just moved to C code and now assembly again :) First of all, we can see that `memcpy` and other routines which are defined here, start and end with the two macros: `GLOBAL` and `ENDPROC`. `GLOBAL` is described in [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h) which defines the `globl` directive and its label. `ENDPROC` is described in [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h) and marks the `name` symbol as a function name and ends with the size of the `name` symbol.

The implementation of `memcpy` is simple. At first, it pushes values from the `si` and `di` registers to the stack to preserve their values because they will change during the `memcpy`. As we can see in the `REALMODE_CFLAGS` in `arch/x86/Makefile`, the kernel build system uses the `-mregparm=3` option of GCC, so functions get the first three parameters from `ax`, `dx` and `cx` registers.  Calling `memcpy` looks like this:
>>>>>>> upstream/master

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

<<<<<<< HEAD
したがって、
* `ax`は、`boot_params.hdr`のアドレスを持ち、
* `dx`は、`hdr`のアドレスを持ち、
* `cx`は、`hdr`のサイズの数値を持ちます。

`memcpy`は`boot_params.hdr`のアドレスを`di`に入れ、スタックにサイズを保存します。
この後、2サイズ右にシフト（あるいは4で除算）し、`si`から`di`に4バイト毎にコピーします。
この後さらに、`hdr`のサイズをリストアし、4バイトでアドレスをそろえ、残りのバイト（もしあれば）を`si`から`di`に1バイト毎にコピーします。
最後に`si`と`di`の値をスタックからリストアすると、コピーは完了です。
=======
So,
* `ax` will contain the address of `boot_params.hdr`
* `dx` will contain the address of `hdr`
* `cx` will contain the size of `hdr` in bytes.

`memcpy` puts the address of `boot_params.hdr` into `di` and saves `cx` on the stack. After this it shifts the value right 2 times (or divides it by 4) and copies four bytes from the address at `si` to the address at `di`. After this, we restore the size of `hdr` again, align it by 4 bytes and copy the rest of the bytes from the address at `si` to the address at `di` byte by byte (if there is more). Now the values of `si` and `di` are restored from the stack and the copying operation is finished.
>>>>>>> upstream/master

コンソールの初期化
--------------------------------------------------------------------------------

<<<<<<< HEAD
`hdr`を`boot_params.hdr`にコピーしたら、次のステップは、
[arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/early_serial_console.c)に定義されている`console_init`関数を呼び、コンソールを初期化することです。

その関数は、`earlyprintk`オプションをコマンドラインから探し、もしあれば、ポートアドレスとシリアルポートのボーレートを解析し、シリアルポートを初期化します。`earlyprintk`コマンドラインオプションの値は、次のうちのいずれかです:
=======
After `hdr` is copied into `boot_params.hdr`, the next step is to initialize the console by calling the `console_init` function,  defined in [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c).

It tries to find the `earlyprintk` option in the command line and if the search was successful, it parses the port address and baud rate of the serial port and initializes the serial port. The value of the `earlyprintk` command line option can be one of these:
>>>>>>> upstream/master

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

シリアルポート初期化の後、最初の出力を見れます:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

<<<<<<< HEAD
`puts`は[tty.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c)で定義されています。
見てのとおり、それはputchar関数を呼び出すことで、1文字1文字をループで表示します。putcharの実装を見てみましょう:
=======
The definition of `puts` is in [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c). As we can see it prints character by character in a loop by calling the `putchar` function. Let's look into the `putchar` implementation:
>>>>>>> upstream/master

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

<<<<<<< HEAD
`__attribute__((section(".inittext")))`は、このコードが`.inittext`セクションにあることを意味しています。
このセクションは、リンカファイル[setup.ld](https://github.com/torvalds/linux/blob/master/arch/x86/boot/setup.ld#L19)内にあります。

まず最初に、`putchar`は`\n`シンボルをチェックし、それが見つかれば`\r`を先に表示します。
その後、`0x10`の割り込みでBIOSを呼び出し、VGAスクリーンに文字を表示させます:
=======
`__attribute__((section(".inittext")))` means that this code will be in the `.inittext` section. We can find it in the linker file [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld).

First of all, `putchar` checks for the `\n` symbol and if it is found, prints `\r` before. After that it prints the character on the VGA screen by calling the BIOS with the `0x10` interrupt call:
>>>>>>> upstream/master

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

ここで、`initregs`は`biosregs`構造体を扱います。最初に`memset`関数を使って`biosregs`をゼロで埋め、それからレジスタの値を入力します。

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

<<<<<<< HEAD
[memset](https://github.com/torvalds/linux/blob/master/arch/x86/boot/copy.S#L36)の実装を見ていきましょう:
=======
Let's look at the implementation of [memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36):
>>>>>>> upstream/master

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

<<<<<<< HEAD
上から読み取れる通り、`memcpy`関数のように、関数呼び出しに`fastcall`呼び出し規約を使っています。
つまり、関数は`ax`、`dx`そして`cx`レジスタから引数を取得しています。

概して`memset`は、`memcpy`の実装に似ています。`di`レジスタの値をスタックに保存し、`ax`レジスタの値をbiosregs構造体のアドレスであるdiに置きます。
次に`movzbl`命令が、`dl`レジスタの値を`eax`レジスタの下位2バイトにコピーします。残りの`eax`の上位2バイトにはゼロが入力されます。

次の命令は`eax`に`0x01010101`をかけます。`memset`が4バイトを同時にコピーできるようにするためです。
例えば、`memset`を使って構造体を`0x7`で埋めたいとします。その場合、`eax`は`0x00000007`を持ちます。
ここでeaxに`0x01010101`をかけると`0x07070707`になり、4バイトを構造体にコピーできるのです。
`memset`は、eaxを`es:di`にコピーする際、`rep; stosl`命令を使います。

`memset`関数が他に行うことは、`memcpy`とほぼ同じです。

`biosregs`構造体が`memset`により埋められると、`bios_putchar`は、[0x10](http://www.ctyme.com/intr/rb-0106.htm)割り込みを呼び出し、文字を表示します。
次に、シリアルポートが初期化されたかどうかをチェックし、初期化済みであれば、[serial_putchar](https://github.com/torvalds/linux/blob/master/arch/x86/boot/tty.c#L30)と、`inb/outb`命令で文字を書き出します。
=======
As you can read above, it uses the same calling conventions as the `memcpy` function, which means that the function gets its parameters from the `ax`, `dx` and `cx` registers.

The implementation of `memset` is similar to that of memcpy. It saves the value of the `di` register on the stack and puts the value of`ax`, which stores the address of the `biosregs` structure, into `di` . Next is the `movzbl` instruction, which copies the value of `dl` to the lowermost byte of the `eax` register. The remaining 3 high bytes of `eax` will be filled with zeros.

The next instruction multiplies `eax` with `0x01010101`. It needs to because `memset` will copy 4 bytes at the same time. For example, if we need to fill a structure whose size is 4 bytes with the value `0x7` with memset, `eax` will contain the `0x00000007`. So if we multiply `eax` with `0x01010101`, we will get `0x07070707` and now we can copy these 4 bytes into the structure. `memset` uses the `rep; stosl` instruction to copy `eax` into `es:di`.

The rest of the `memset` function does almost the same thing as `memcpy`.

After the `biosregs` structure is filled with `memset`, `bios_putchar` calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt which prints a character. Afterwards it checks if the serial port was initialized or not and writes a character there with [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c) and `inb/outb` instructions if it was set.
>>>>>>> upstream/master

ヒープの初期化
--------------------------------------------------------------------------------

<<<<<<< HEAD
スタックとbssセクションが[header.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S)（前の[章](linux-bootstrap-1.md)参照）に準備できたら、カーネルは[`init_heap`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116)関数を使って[ヒープ](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L116)を初期化する必要があります。

まず`init_heap`はカーネルセットアップヘッダにある[`loadflags`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/header.S#L321) から[`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/master/arch/x86/include/uapi/asm/bootparam.h#L21)フラグをチェックし、フラグがセットされている場合はスタックの終わりを計算します:
=======
After the stack and bss section have been prepared in [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) (see previous [part](linux-bootstrap-1.md)), the kernel needs to initialize the [heap](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) with the [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function.

First of all `init_heap` checks the [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) flag from the [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) structure in the kernel setup header and calculates the end of the stack if this flag was set:
>>>>>>> upstream/master

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

言い換えると、`stack_end = esp - STACK_SIZE` という計算を行います。

`heap_end`の計算は以下のようになります:

<<<<<<< HEAD
```c
    heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

これは、`heap_end_ptr`または、`_end + 512`(`0x200h`)を意味します。最後に`heap_end`は`stack_end`より大きいかどうかがチェックされます。それが正の場合は、それらをイコールにするため、`stack_end`が`heap_end`に適用されます。

これでヒープは初期化され、`GET_HEAP`メソッドを用いてこれを使うことができるようになりました。実際の使われ方、使い方、実装は次の章で見ていきましょう。
=======
Then there is the `heap_end` calculation:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

which means `heap_end_ptr` or `_end` + `512` (`0x200h`). The last check is whether `heap_end` is greater than `stack_end`. If it is then `stack_end` is assigned to `heap_end` to make them equal.

Now the heap is initialized and we can use it using the `GET_HEAP` method. We will see what it is used for, how to use it and how it is implemented in the next posts.
>>>>>>> upstream/master

CPUのバリデーション
--------------------------------------------------------------------------------

<<<<<<< HEAD
次のステップは、[arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpu.c)に書かれた`validate_cpu`によるCPUのバリデーションです。

これは[`check_cpu`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/cpucheck.c#L102)関数を呼び出し、現在のCPUレベルと必須とされているCPUレベルを渡してカーネルが正しいCPUレベルで起動していることをチェ確認します。

```c
=======
The next step as we can see is cpu validation through the `validate_cpu` function from [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c) source code file.

It calls the [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) function and passes cpu level and required cpu level to it and checks that the kernel launches on the right cpu level.

```C
>>>>>>> upstream/master
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```
<<<<<<< HEAD
=======

The `check_cpu` function checks the CPU's flags, the presence of [long mode](http://en.wikipedia.org/wiki/Long_mode) in the case of x86_64(64-bit) CPU, checks the processor's vendor and makes preparations for certain vendors like turning off SSE+SSE2 for AMD if they are missing, etc.

at the next step, we may see a call to the `set_bios_mode` function after setup code found that a CPU is suitable. As we may see, this function is implemented only for the `x86_64` mode:

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

The `set_bios_mode` function executes the `0x15` BIOS interrupt to tell the BIOS that [long mode](https://en.wikipedia.org/wiki/Long_mode) (if `bx == 2`) will be used.
>>>>>>> upstream/master

`check_cpu`はCPUのフラグを確認します。x86_64（64-bit）CPUの場合は[long mode](http://en.wikipedia.org/wiki/Long_mode)の存在を確認します。また、CPUベンダーを確認し、AMDのようなSSE+SSE2がないCPUの場合、その機能をオフにします。

メモリ検知
--------------------------------------------------------------------------------

<<<<<<< HEAD
次のステップは`detect_memory`関数によるメモリ検知です。`detect_memory`は使用可能なRAMのマップをCPUに提供します。メモリ検知には`0xe820`、`0xe801`そして`0x8`8などのいくつかの異なるプログラミングインターフェースを使います。ここでは **0xE820** の実装だけを見ていきましょう。

[arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/memory.c) ソースファイルから`detect_memory_e820`の実装を見ましょう。まず先述のように`detect_memory_e820`関数は`biosregs`構造体を初期化し、レジスタに`0xe820`呼び出しのための特別な値を入力します:
=======
The next step is memory detection through the `detect_memory` function. `detect_memory` basically provides a map of available RAM to the CPU. It uses different programming interfaces for memory detection like `0xe820`, `0xe801` and `0x88`. We will see only the implementation of the **0xE820** interface here.

Let's look at the implementation of the `detect_memory_e820` function from the [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c) source file. First of all, the `detect_memory_e820` function initializes the `biosregs` structure as we saw above and fills registers with special values for the `0xe820` call:
>>>>>>> upstream/master

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

<<<<<<< HEAD
* `ax` は関数のアドレスを内包します (ここでは0xe820)
* `cx` レジスタは、メモリに関するデータを格納するバッファのサイズを持ちます。
* `edx` は `SMAP` マジックナンバーを持っている必要があります。
* `es:di` はメモリデータを含むバッファのアドレスを内包する必要があります。
* `ebx` は0を持っている必要がある.

次に、メモリに関するデータを収集するループです。`0x15` BIOS割り込みの呼び出しで始まり、address allocation tableから1行を書き出します。次の行を取得するには、この割り込みを再度呼び出す必要があります（ループ内で行います）。次の呼び出しの前に`ebx`は前に返り値を持たねばなりません:
=======
* `ax` contains the number of the function (0xe820 in our case)
* `cx` contains the size of the buffer which will contain data about the memory
* `edx` must contain the `SMAP` magic number
* `es:di` must contain the address of the buffer which will contain memory data
* `ebx` has to be zero.

Next is a loop where data about the memory will be collected. It starts with a call to the `0x15` BIOS interrupt, which writes one line from the address allocation table. For getting the next line we need to call this interrupt again (which we do in the loop). Before the next call `ebx` must contain the value returned previously:
>>>>>>> upstream/master

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

<<<<<<< HEAD
最終的ににebxはループ内で反復し、address allocation table からデータを集め、データを以下のように`e820entry`配列に書き込みます:

* メモリセグメントの開始アドレス
* メモリセグメントのサイズ
* メモリセグメントのタイプ (予約済み、使用不可能 etc...).
=======
Ultimately, this function collects data from the address allocation table and writes this data into the `e820_entry` array:

* start of memory segment
* size  of memory segment
* type of memory segment (whether the particular segment is usable or reserved)
>>>>>>> upstream/master

この結果は以下のような`dmesg`の出力によって確認できます:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

キーボードの初期化
--------------------------------------------------------------------------------

<<<<<<< HEAD
次のステップは[`keyboard_init()`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/main.c#L65)関数の呼び出しによるキーボードの初期化です。
`keyboard_init`は、まず`initregs`関数を使いレジスタを初期化し、[0x16](http://www.ctyme.com/intr/rb-1756.htm)割り込みを呼び出して、キーボードのステータスを取得します。。
=======
The next step is the initialization of the keyboard with a call to the [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) function. At first `keyboard_init` initializes registers using the `initregs` function. It then calls the [0x16](http://www.ctyme.com/intr/rb-1756.htm) interrupt to query the status of the keyboard.
>>>>>>> upstream/master

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

<<<<<<< HEAD
この処理が終わった後、[0x16](http://www.ctyme.com/intr/rb-1757.htm)を再度呼び出しリピート率と遅延時間を設定します。
=======
After this it calls [0x16](http://www.ctyme.com/intr/rb-1757.htm) again to set the repeat rate and delay.
>>>>>>> upstream/master

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

クエリ
--------------------------------------------------------------------------------

<<<<<<< HEAD
次の2ステップはいくつかのパラメータのためのクエリです。これらクエリについて今は詳細に追いませんが、また後のパートで見ましょう。簡単にこれらの関数を見ていきましょう:

[query_mca](https://github.com/torvalds/linux/blob/master/arch/x86/boot/mca.c#L18)ルーティンは、
[0x15](http://www.ctyme.com/intr/rb-1594.htm)BIOS割り込みを呼び出し、マシンモデルナンバー、サブモデルナンバー、BIOSアップデートレベル、その他のハードウェアの属性を取得します:

```c
int query_mca(void)
{
    struct biosregs ireg, oreg;
    u16 len;

    initregs(&ireg);
    ireg.ah = 0xc0;
    intcall(0x15, &ireg, &oreg);

    if (oreg.eflags & X86_EFLAGS_CF)
        return -1;  /* No MCA present */

    set_fs(oreg.es);
    len = rdfs16(oreg.bx);

    if (len > sizeof(boot_params.sys_desc_table))
        len = sizeof(boot_params.sys_desc_table);

    copy_from_fs(&boot_params.sys_desc_table, oreg.bx, len);
    return 0;
}
```

これは`ah`レジスタに`0xc0`を入れ、`0x15`BIOS割り込みを呼び出します。割り込みが実行された後、[carry flag](http://en.wikipedia.org/wiki/Carry_flag)をチェックし、1のときはBIOSは[**MCA**](https://en.wikipedia.org/wiki/Micro_Channel_architecture)をサポートしません。キャリーフラグが0の場合は、`ES:BX`はシステム情報テーブルへのポインタを指します。詳細は次のとおりです:

```
Offset  Size    Description
 00h    WORD    number of bytes following
 02h    BYTE    model (see #00515)
 03h    BYTE    submodel (see #00515)
 04h    BYTE    BIOS revision: 0 for first release, 1 for 2nd, etc.
 05h    BYTE    feature byte 1 (see #00510)
 06h    BYTE    feature byte 2 (see #00511)
 07h    BYTE    feature byte 3 (see #00512)
 08h    BYTE    feature byte 4 (see #00513)
 09h    BYTE    feature byte 5 (see #00514)
---AWARD BIOS---
 0Ah  N BYTEs   AWARD copyright notice
---Phoenix BIOS---
 0Ah    BYTE    ??? (00h)
 0Bh    BYTE    major version
 0Ch    BYTE    minor version (BCD)
 0Dh  4 BYTEs   ASCIZ string "PTL" (Phoenix Technologies Ltd)
---Quadram Quad386---
 0Ah 17 BYTEs   ASCII signature string "Quadram Quad386XT"
---Toshiba (Satellite Pro 435CDS at least)---
 0Ah  7 BYTEs   signature "TOSHIBA"
 11h    BYTE    ??? (8h)
 12h    BYTE    ??? (E7h) product ID??? (guess)
 13h  3 BYTEs   "JPN"
 ```

次に、`set_fs`ルーティンを呼び出し、`es`レジスタの値をそこに渡します。`set_fs`の実装は非常にシンプルです:

```c
static inline void set_fs(u16 seg)
{
    asm volatile("movw %0,%%fs" : : "rm" (seg));
}
```

この関数はインラインアセンブリを内包しており、`seg`パラメータを取得してそれを`fs`レジスタに置きます。
[boot.h](https://github.com/torvalds/linux/blob/master/arch/x86/boot/boot.h) には、`set_gs`、`fs`、`gs`といったたくさんの関数があります。

`query_mca`の終わりでは、`es:bx`によって指されてたテーブルを`boot_params.sys_desc_table`にコピーするだけです。

次のステップは、`query_ist`関数を呼び出し、[Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)の情報を取得することです。まずCPUレベルをチェックし、それが正しければ`0x15`を呼び出して情報を取得し、その結果を`boot_params`に保存します。

下記の[query_apm_bios](https://github.com/torvalds/linux/blob/master/arch/x86/boot/apm.c#L21)関数は[Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management)情報をBIOSから取得します。`query_apm_bios`は`0x15`のBIOS割り込みも呼び出しますが、`APM`のインストールをチェックするため`ah`=`0x53`を使います。`0x15`実行後、`query_apm_bios`関数はPMの署名（`0x504d`であること）、キャリーフラグ（`APM`がサポートしている場合は0）と`cx`レジスタ（0x02の場合、プロテクトモードがサポートされていること）をチェックします。

次に再度`0x15`を呼び出しますが、`APM`インターフェースとの接続を切り、32bit プロテクトモードに接続するため、`ax=0x5304`を使います。最後にBIOSから得た値を`boot_params.apm_bios_info`に入力します。

`query_apm_bios`は、`CONFIG_APM`か`CONFIG_APM_MODULE`が設定ファイルで有効になっている場合のみ実行されることに注意してください:
=======
The next couple of steps are queries for different parameters. We will not dive into details about these queries but we will get back to them in later parts. Let's take a short look at these functions:

The first step is getting [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) information by calling the `query_ist` function. It checks the CPU level and if it is correct, calls `0x15` to get the info and saves the result to `boot_params`.

Next, the [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) function gets [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) information from the BIOS. `query_apm_bios` calls the `0x15` BIOS interruption too, but with `ah` = `0x53` to check `APM` installation. After `0x15` finishes executing, the `query_apm_bios` functions check the `PM` signature (it must be `0x504d`), the carry flag (it must be 0 if `APM` supported) and the value of the `cx` register (if it's 0x02, the protected mode interface is supported).

Next, it calls `0x15` again, but with `ax = 0x5304` to disconnect the `APM` interface and connect the 32-bit protected mode interface. In the end, it fills `boot_params.apm_bios_info` with values obtained from the BIOS.

Note that `query_apm_bios` will be executed only if the `CONFIG_APM` or `CONFIG_APM_MODULE` compile time flag was set in the configuration file:
>>>>>>> upstream/master

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

<<<<<<< HEAD
最後の[`query_edd`](https://github.com/torvalds/linux/blob/master/arch/x86/boot/edd.c#L122)関数は、`Enhanced Disk Drive`情報をBIOSから問い合わせます。`query_edd`の実装を見てみましょう。

まず[edd](https://github.com/torvalds/linux/blob/master/Documentation/kernel-parameters.txt#L1023)オプションをカーネルコマンドラインから読み取り、`off`に設定されている場合は`query_edd`の値をそのまま返します。
=======
The last is the [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122) function, which queries `Enhanced Disk Drive` information from the BIOS. Let's look at how `query_edd` is implemented.

First of all, it reads the [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) option from the kernel's command line and if it was set to `off` then `query_edd` just returns.
>>>>>>> upstream/master

EDDが有効になっている場合は、`query_edd`はBIOSがサポートしているハードディスクに行き、次のようなループでEDD情報を問い合わせます:

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    }
```

<<<<<<< HEAD
`0x80`があるのは最初のハードドライブで、`EDD_MBR_SIG_MAX`マクロの値は16です。
これはデータを[edd_info](https://github.com/torvalds/linux/blob/master/include/uapi/linux/edd.h#L172)構造体の配列に集めます。
`get_edd_info`はEDDが存在しているかどうかを、`ah`に`0x41`を入れ、`0x13`割り込みを呼び出し確かめ、
もし存在していれば`get_edd_info`が再び`0x13`割り込みを呼び出します。
=======
where `0x80` is the first hard drive and the value of the `EDD_MBR_SIG_MAX` macro is 16. It collects data into an array of [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h) structures. `get_edd_info` checks that EDD is present by invoking the `0x13` interrupt with `ah` as `0x41` and if EDD is present, `get_edd_info` again calls the `0x13` interrupt, but with `ah` as `0x48` and `si` containing the address of the buffer where EDD information will be stored.
>>>>>>> upstream/master

まとめ
--------------------------------------------------------------------------------

<<<<<<< HEAD
これでLinuxカーネルインターナルに関する記事のパート2は終わりです。次のパートではビデオモード設定とプロテクトモード移行の前に必要な残りの準備、そしてそのまま移行について見ていきましょう。
もし質問や提案があれば Twitter [0xAX](https://twitter.com/0xAX) や [email](anotherworldofworld@gmail.com) で連絡していただくか、Issueを作成してください。
=======
This is the end of the second part about the insides of the Linux kernel. In the next part, we will see video mode setting and the rest of the preparations before the transition to protected mode and directly transitioning into it.
>>>>>>> upstream/master

**ご注意：英語は私の第一言語ではないことをご承知おきください。誤りを見つけた方は[linux-insides](https://github.com/0xAX/linux-internals)に、プルリクエストを送ってください。**


--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
* [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [1つ前のパート](linux-bootstrap-1.md)
