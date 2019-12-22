Kernel initialization. Part 1.
================================================================================

kernelコード内の最初のステップ
--------------------------------------------------------------------------------

[前回](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)は、Linux kernelの[ブート処理](https://0xax.gitbooks.io/linux-insides/content/Booting/index.html)の章の最後のパートでした。次はLinux kernelの初期化処理に潜り込みます。
まず、Linux Kernelのイメージが展開され、メモリの正しい位置に配置されると動作を開始します。
これまでの全てのパートは、Linux kernelのコードの最初のバイトが実行される前に準備を行っている、セットアップコードの動きについて説明しました。
これからkernelの中を見ていきますが、この章の全てのパートは、[pid](https://en.wikipedia.org/wiki/Process_identifier) `1`のプロセスを立ち上げる前のkernelの初期化処理に専念しています。
kernelが`init`を立ち上げる前にすべきことが多くあります。そのため、この大きな章でkernelをはじめる前に、全ての準備を確認しておくことが望ましい。?

[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)に位置するkernelのエントリポイントから開始し、さらに先へ進んでいきます。
[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)内から呼ばれる`start_kernel`関数を見る前に、早期のページテーブル初期化や、新しいkernel空間のディスクリプタへの切り替え等の、その他にもたくさんある最初の準備処理について見ていきます。
[前の章](https://0xax.gitbooks.io/linux-insides/content/Booting/index.html) の[最後のパート](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)では、[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/head_64.S)のアセンブラソースコードに記載されているjmp命令で止まっていました。


```assembly
jmp	*%rax
```
この時点では`rax`レジスタは、[arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c)に定義されている`decompress_kernel`関数により取得した、Linux kernelのエントリポイントのアドレスを保持しています。
そのため、kernelのセットアップコードの最後の命令はkernelのエントリポイントへのjumpとなります。Linux kernelのエントリポイントがどこに定義されているかは既に説明しているので、この後Linux kernelが何をするかを学び始めることができます。

kernelの最初のステップ
--------------------------------------------------------------------------------

さて、解凍済みのkernelイメージのアドレスを`decompress_kernel`関数により`rax`レジスタへ取得し、そのアドレスへジャンプしました。
解凍済みkernelイメージのエントリポイントは、[arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)のアセンブリソースコードファイルの先頭の、以下定義から開始されます。

```assembly
    .text
	__HEAD
	.code64
	.globl startup_64
startup_64:
	...
	...
	...
```

`__HEAD`セクションに`startup_64`ルーチンの定義があります。これは、実行可能な`.head.text`セクションに展開されるマクロです。

We can see definition of the `startup_64` routine that is defined in the `__HEAD` section, which is just a macro which expands to the definition of executable `.head.text` section:

```C
#define __HEAD		.section	".head.text","ax"
```

このセクションは[arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)のリンカスクリプトに定義があります。
We can see definition of this section in the [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) linker script:

```
.text : AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	...
	...
	...
} :text = 0x9090
```
`.text`セクションの定義に加えて、リンカスクリプトより、仮想、および物理アドレスのデフォルト値を知ることができます。
`_text`のアドレスは以下のようにロケーションカウンタで定義されることに注意してください。
Besides the definition of the `.text` section, we can understand default virtual and physical addresses from the linker script. Note that address of the `_text` is location counter which is defined as:

```
. = __START_KERNEL;
```

[x86_64](https://en.wikipedia.org/wiki/X86-64)の例です。`__START_KERNEL`マクロの定義は、[arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h)のヘッダファイルにあります。これは、kernelがマッピングされるベース仮想アドレスとベース物理アドレスの合計で表現されます。
for [x86_64](https://en.wikipedia.org/wiki/X86-64). The definition of the `__START_KERNEL` macro is located in the [arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h) header file and represented by the sum of the base virtual address of the kernel mapping and physical start:

```C
#define __START_KERNEL	(__START_KERNEL_map + __PHYSICAL_START)

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```
または言い換えると：

* Linux Kernelのベース物理アドレス - `0x1000000`;
* Linux Kernelのベース仮想アドレス(.textセクションの開始アドレス) - `0xffffffff81000000`

となります。

Or in other words:

* Base physical address of the Linux kernel - `0x1000000`;
* Base virtual address of the Linux kernel - `0xffffffff81000000`.

CPUコンフィグレーションをサニタイズした後、[arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)に定義されている`__startup_64`関数を呼び出します。
After we sanitized CPU configuration, we call `__startup_64` function which is defined in [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c):

```assembly
	leaq	_text(%rip), %rdi
	pushq	%rsi
	call	__startup_64
	popq	%rsi
```

```C
unsigned log __head __startup_64(unsigned long physaddr,
				 struct boot_params *bp)
{
	unsigned long load_delta, *p;
	unsigned long pgtable_flags;
	pgdval_t *pgd;
	p4dval_t *p4d;
	pudval_t *pud;
	pmdval_t *pmd, pmd_entry;
	pteval_t *mask_ptr;
	bool la57;
	int i;
	unsigned int *next_pgt_ptr;
	...
	...
	...
}
```
[kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux)が有効になっていると、`__startup_64`ルーチンのアドレスは、コンパイル時のアドレスと異なることがあります。そのため、以下のコードにより差分を計算する必要があります。

Since [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux) is enabled, the address `startup_64` routine was loaded may be different from the address compiled to run at, so we need to calculate the delta with the following code:

```C
	load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);
```

結果として、`__load_delta`は、コンパイル時のアドレスと実際にロードされたアドレスの差分を含んでいます。
As a result, `load_delta` contains the delta between the address compiled to run at and the address actually loaded.

この差分を得た後、`_text`のアドレスが正しく`2`メガバイトにアラインされているかどうかを、以下コードにより確認します。
After we got the delta, we check if `_text` address is correctly aligned for `2` megabytes. We will do it with the following code:

```C
	if (load_delta & ~PMD_PAGE_MASK)
		for (;;);
```
もし`_text`のアドレスが`2`メガバイトにアラインされていなければ、無限ループに陥ります。`PMD_PAGE_MASK`は`Page middle directory`([ページング](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)を参照) のためのマスクを表現しており、以下のように定義されます。
If `_text` address is not aligned for `2` megabytes, we enter infinite loop. The `PMD_PAGE_MASK` indicates the mask for `Page middle directory` (read [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html) about it) and is defined as:

```C
#define PMD_PAGE_MASK           (~(PMD_PAGE_SIZE-1))
```

`PMD_PAGE_SIZE` は以下のように定義されます。
where `PMD_PAGE_SIZE` macro is defined as:

```C
#define PMD_PAGE_SIZE           (_AC(1, UL) << PMD_SHIFT)
#define PMD_SHIFT		21
```

簡単に計算できるように、`PMD_PAGE_SIZE`は`2`メガバイトです。
As we can easily calculate, `PMD_PAGE_SIZE` is `2` megabytes.

もし[SME](https://en.wikipedia.org/wiki/Zen_%28microarchitecture%29#Enhanced_security_and_virtualization_support)がサポートされており、有効になっている場合は、SMEをアクティベートし、SME暗号化マスクを`load_delta`に含めます。
If [SME](https://en.wikipedia.org/wiki/Zen_%28microarchitecture%29#Enhanced_security_and_virtualization_support) is supported and enabled, we activate it and include the SME encryption mask in `load_delta`:

```C
	sme_enable(bp);
	load_delta += sme_get_me_mask();
```

さて、いくつかの初期チェックを行たったことで、次に進めます。
Okay, we did some early checks and now we can move on.

ページテーブルのベースアドレスの修正
--------------------------------------------------------------------------------

次のステップでは、ページテーブル内の物理アドレスを修正します。
In the next step we fixup the physical addresses in the page table:

```C
	pgd = fixup_pointer(&early_top_pgt, physaddr);
	pud = fixup_pointer(&level3_kernel_pgt, physaddr);
	pmd = fixup_pointer(level2_fixmap_pgt, physaddr);
```

それでは、渡された引数の物理アドレスを返す、`fixup_pointer`関数の定義を見てみましょう。
So, let's look at the definition of `fixup_pointer` function which returns physical address of the passed argument:

```C
static void __head *fixup_pointer(void *ptr, unsigned long physaddr)
{
	return ptr - (void *)_text + (void *)physaddr;
}
```

次に、`early_top_pgt`と、上で見たその他のページテーブルシンボルに注目します。それでは、これらのシンボルの意味を考えてみましょう。まず、その定義を見てみます。
Next we'll focus on `early_top_pgt` and the other page table symbols which we saw above. Let's try to understand what these symbols mean. First of all let's look at their definition:

```assembly
NEXT_PAGE(early_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0

NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE_NOENC
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC

NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

一見難しそうに見えますが、そうではありません。まず、`early_top_pgt`を見てみましょう。`4096`バイト(`CONFIG_PAGE_TABLE_ISOLATION`が有効になっている場合は`8192`バイト)のゼロで始まるため、最初の`512`エントリは利用しないことを意味します。
Looks hard, but it isn't. First of all let's look at the `early_top_pgt`. It starts with the `4096` bytes of zeros (or `8192` bytes if `CONFIG_PAGE_TABLE_ISOLATION` is enabled), it means that we don't use the first `512` entries.
そしてこの後、`level3_kernel_pgt`エントリがあります。この定義の最初も、``4080`バイトのゼロ(`L3_START_KERNEL`は`510`に等しい)で埋まっています。
And after this we can see `level3_kernel_pgt` entry. At the start of its definition, we can see that it is filled with the `4080` bytes of zeros (`L3_START_KERNEL` equals `510`). 
その後、kernel空間をマップする二つのエントリを保存します。`level2_kernel_pgt`と`level2_fixmap_pgt`から、`__START_KERNEL_map`を減算することに注意してください。
Subsequently, it stores two entries which map kernel space. Note that we subtract `__START_KERNEL_map` from `level2_kernel_pgt` and `level2_fixmap_pgt`.
`__START_KERNEL_map`はkernelのtextセクションのベース仮想アドレスであることはわかっているので、`__START_KERNEL_map`を減算してやれば、`level2_kernel_pgt`と`level2_fixmap_pgt`の物理アドレスが取得できます。
As we know `__START_KERNEL_map` is a base virtual address of the kernel text, so if we subtract `__START_KERNEL_map`, we will get physical addresses of the `level2_kernel_pgt` and `level2_fixmap_pgt`.

次に、`_KERNPG_TABLE_NOENC`と`_PAGE_TABLE_NOENC`を見ていきましょう。これらは、単にページエントリのアクセス権です。
Next let's look at `_KERNPG_TABLE_NOENC` and `_PAGE_TABLE_NOENC`, these are just page entry access rights:

```C
#define _KERNPG_TABLE_NOENC   (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | \
			       _PAGE_DIRTY)
#define _PAGE_TABLE_NOENC     (_PAGE_PRESENT | _PAGE_RW | _PAGE_USER | \
			       _PAGE_ACCESSED | _PAGE_DIRTY)
```
`level2_kernel_pgt`は、kernel空間にマップするページミドルディレクトリへのポインタを含むページテーブルエントリです。これは、kernelの`.text`セクションの`__START_KERNEL_map`から、512`メガバイトをつくりだす`PDMS`マクロを呼びます。(この`512`メガバイトは後でモジュールのためのメモリ空間となります)
The `level2_kernel_pgt` is page table entry which contains pointer to the page middle directory which maps kernel space. It calls the `PDMS` macro which creates `512` megabytes from the `__START_KERNEL_map` for kernel `.text` (after these `512` megabytes will be module memory space).

`level2_fixmap_pgt`は、任意の物理アドレス(kernel空間下でさえ)を参照できる仮想アドレスです。これらは、`4048`バイトのゼロ、`level1_fixmap_pgt`エントリ、[vsyscalls](https://lwn.net/Articles/446528/)のマッピングのために予約された`8`メガバイトの領域、`2`メガバイトの空き領域、で表されます。
The `level2_fixmap_pgt` is a virtual addresses which can refer to any physical addresses even under kernel space. They are represented by the `4048` bytes of zeros, the `level1_fixmap_pgt` entry, `8` megabytes reserved for [vsyscalls](https://lwn.net/Articles/446528/) mapping and `2` megabytes of hole.

詳細については、[ページング](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)を参照ください。
You can read more about it in the [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html) part.

さて、これらのシンボルの定義を見たので、コードに戻りましょう。次は、`pgd`の最後のエントリを`level3_kernel_pgt`で初期化します。
Now, after we saw the definitions of these symbols, let's get back to the code. Next we initialize last entry of `pgd` with `level3_kernel_pgt`:

```C
	pgd[pgd_index(__START_KERNEL_map)] = level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC;
```

すべての`P*d`アドレスは、`startup_64`が`0x1000000`のデフォルトアドレス値と異なれば、間違っているかもしれません。`load_delta`は、kernelの[リンク](https://en.wikipedia.org/wiki/Linker_%28computing%29)中に取得された`startup_64`シンボルと、実際のアドレス間の差分を含むことを思い出してください。ここで、その差分を特定の`p*d`エントリに対して加算します。
All of `p*d` addresses may be wrong if the `startup_64` is not equal to default `0x1000000` address. Remember that the `load_delta` contains delta between the address of the `startup_64` symbol which was got during kernel [linking](https://en.wikipedia.org/wiki/Linker_%28computing%29) and the actual address. So we add the delta to the certain entries of the `p*d`.

```C
	pgd[pgd_index(__START_KERNEL_map)] += load_delta;
	pud[510] += load_delta;
	pud[511] += load_delta;
	pmd[506] += load_delta;
```

これらの後、次のようになります。
After all of this we will have:

```
early_top_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[506] -> level1_fixmap_pgt
```

`early_top_pgt`といくつかのページテーブルディレクトリのベースアドレスは、ビルド時にページテーブルがfillされたときに決定するため、修正しなかったことに注意してください。ページテーブルのアドレスを修正したら、ビルドを開始できます。
Note that we didn't fixup base address of the `early_top_pgt` and some of other page table directories, because we will see this when building/filling structures of these page tables. As we corrected base addresses of the page tables, we can start to build it.

Identity mappingのセットアップ
--------------------------------------------------------------------------------

これで、初期ページテーブルのidentity mappingの設定を確認できます。Identity Mapped ページングでは、仮想アドレスは物理アドレスに同じようにマッピングされます。詳細を見ていきましょう。まず第一に、`pud`と`pmd`を、`early_dynamic_pgts`の最初と二番目のエントリを示すポインタに置き換えます。
Now we can see the set up of identity mapping of early page tables. In Identity Mapped Paging, virtual addresses are mapped to physical addresses identically. Let's look at it in detail. First of all we replace `pud` and `pmd` with the pointer to first and second entry of `early_dynamic_pgts`:

```C
	next_pgt_ptr = fixup_pointer(&next_early_pgt, physaddr);
	pud = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
	pmd = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
```

`early_dynamic_pgts`の定義を見ていきましょう。
Let's look at the `early_dynamic_pgts` definition:

```assembly
NEXT_PAGE(early_dynamic_pgts)
	.fill	512*EARLY_DYNAMIC_PAGE_TABLES,8,0
```

これは、初期カーネル用の一時ページテーブルを保存しています。
which will store temporary page tables for early kernel.

次に、後で`p*d`エントリを初期化する際に使う、`pagetable_flags`を初期化します。
Next we initialize `pgtable_flags` which will be used when initializing `p*d` entries later:

```C
	pgtable_flags = _KERNPG_TABLE_NOENC + sme_get_me_mask();
```
`sme_get_me_mask`関数は、`sme_enable`関数内で初期化された`sme_me_mask`を返します。
`sme_get_me_mask` function returns `sme_me_mask` which was initialized in `sme_enable` function.

次に、`pgd`の二つのエントリを、`bud`に上記で初期化した`pgtable_flags`を加算した値で埋めます。
Next we fill two entries of `pgd` with `pud` plus `pgtable_flags` which we initialized above:

```C
	i = (physaddr >> PGDIR_SHIFT) % PTRS_PER_PGD;
	pgd[i + 0] = (pgdval_t)pud + pgtable_flags;
	pgd[i + 1] = (pgdval_t)pud + pgtable_flags;
```

`PGDIR_SHFT`は、仮想アドレスのページグローバルディレクトリビットのマスクを表しています。また、`512`を超えるインデックスにアクセスしないように、`PTRS_PER_PGD`(`512`として展開される)とのmoduloを計算します。なお、すべてのタイプのページディレクトリ用のマクロがあります。
`PGDIR_SHFT` indicates the mask for page global directory bits in a virtual address. Here we calculate modulo with `PTRS_PER_PGD` (which expands to `512`) so as not to access the index greater than `512`. There are macro for all types of page directories:

```C
#define PGDIR_SHIFT     39
#define PTRS_PER_PGD	512
#define PUD_SHIFT       30
#define PTRS_PER_PUD	512
#define PMD_SHIFT       21
#define PTRS_PER_PMD	512
```
上記とほぼ同じことを行います。
We do the almost same thing above:

```C
	i = (physaddr >> PUD_SHIFT) % PTRS_PER_PUD;
	pud[i + 0] = (pudval_t)pmd + pgtable_flags;
	pud[i + 1] = (pudval_t)pmd + pgtable_flags;
```

次に、`pmd_entry`を初期化し、サポートされていない`__PAGE_KERNEL_*`ビットを除外します。
Next we initialize `pmd_entry` and filter out unsupported `__PAGE_KERNEL_*` bits:

```C
	pmd_entry = __PAGE_KERNEL_LARGE_EXEC & ~_PAGE_GLOBAL;
	mask_ptr = fixup_pointer(&__supported_pte_mask, physaddr);
	pmd_entry &= *mask_ptr;
	pmd_entry += sme_get_me_mask();
	pmd_entry += physaddr;
```

次に、kernelのフルサイズをカバーするために、すべての`pmd`エントリを埋めます。
Next we fill all `pmd` entries to cover full size of the kernel:

```C
	for (i = 0; i < DIV_ROUND_UP(_end - _text, PMD_SIZE); i++) {
		int idx = i + (physaddr >> PMD_SHIFT) % PTRS_PER_PMD;
		pmd[idx] = pmd_entry + i * PMD_SIZE;
	}
```

次に、kernelのtext+dataの仮想アドレスを修正します。kernelが再配置されると、無効なpmdを書き込む可能性があることに注意してください。(`cleanup_highmap`関数は、これを`_end`を超えたマッピングとともに修正します)
Next we fixup the kernel text+data virtual addresses. Note that we might write invalid pmds, when the kernel is relocated (`cleanup_highmap` function fixes this up along with the mappings beyond `_end`).

```C
	pmd = fixup_pointer(level2_kernel_pgt, physaddr);
	for (i = 0; i < PTRS_PER_PMD; i++) {
		if (pmd[i] & _PAGE_PRESENT)
			pmd[i] += load_delta;
	}
```
次に、真の物理アドレスを得るためにメモリ暗号化マスクをはずします。(`load_delta`はマスクを含むことに注意してください)
Next we remove the memory encryption mask to obtain the true physical address (remember that `load_delta` includes the mask):

```C
	*fixup_long(&phys_base, physaddr) += load_delta - sme_get_me_mask();
```

`phys_base`は`level2_kernel_pgt`の最初のエントリと一致している必要があります。
`phys_base` must match the first entry in `level2_kernel_pgt`.

`__startup_64`関数の最後のステップとして、kernelを暗号化し(SMEが有効な場合)、`cr3`レジスタにプログラムされた、初期ページディレクトリエントリの修飾子として利用されるSME暗号化マスクを返します。
As final step of `__startup_64` function, we encrypt the kernel (if SME is active) and return the SME encryption mask to be used as a modifier for the initial page directory entry programmed into `cr3` register:

```C
	sme_encrypt_kernel(bp);
	return sme_get_me_mask();
```

それでは、アセンブリコードに戻りましょう。次のパラグラフの準備を以下のコードで行います。
Now let's get back to assembly code. We prepare for next paragraph with following code:

```assembly
	addq	$(early_top_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

`early_top_pgt`の物理アドレスを`rax`レジスタにaddしています。その結果、`rax`レジスタはアドレスとSME暗号化マスクの合計値を含みます。
which adds physical address of `early_top_pgt` to `rax` register so that `rax` register contains sum of the address and the SME encryption mask.

以上です。初期のページングは準備が終わり、kernelエントリポイントにジャンプする前の最後の準備を完了する必要があります。
That's all for now. Our early paging is prepared and we just need to finish last preparation before we will jump into kernel entry point.

kernelエントリポイントにジャンプする前の最後の準備
--------------------------------------------------------------------------------

その後、ラベル`1`にジャンプし、`PAE`、`PGE`(Paging Global Extension)を有効にし、`phys_base`(上記を参照)の中身を`rax`レジスタに代入し、`rc3`レジスタにそれを入力します。
After that we jump to the label `1` we enable `PAE`, `PGE` (Paging Global Extension) and put the content of the `phys_base` (see above) to the `rax` register and fill `cr3` register with it:

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

次のステップでは、CPUが[NX](http://en.wikipedia.org/wiki/NX_bit)ビットをサポートしていることを確認します。
In the next step we check that CPU supports [NX](http://en.wikipedia.org/wiki/NX_bit) bit with:

```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

`0x80000001`を`eax`に入れ、拡張プロセッサ情報(extended processor info)と機能ビット(feature bits)をを得るために、`cpuid`命令を実行します。その結果は、`edx`レジスタに入れられ、`edi`に移されます。
We put `0x80000001` value to the `eax` and execute `cpuid` instruction for getting the extended processor info and feature bits. The result will be in the `edx` register which we put to the `edi`.

次に、`0xc0000080`、もしくは`MSR_EFER`を`ecx`に入れ、モデル固有のレジスタ読み出しのための命令である`rdmsr`(Reading Model Specific Register)を実行します。
Now we put `0xc0000080` or `MSR_EFER` to the `ecx` and execute `rdmsr` instruction for the reading model specific register.

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

その結果は、`edx:eax`に格納されます。一般的な`EFFR`の一覧は次の通りです。
The result will be in the `edx:eax`. General view of the `EFER` is following:

```
63                                                                              32
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved MBZ                                   |
|                                                                               |
 --------------------------------------------------------------------------------
31                            16  15      14      13   12  11   10  9  8 7  1   0
 --------------------------------------------------------------------------------
|                              | T |       |       |    |   |   |   |   |   |   |
| Reserved MBZ                 | C | FFXSR | LMSLE |SVME|NXE|LMA|MBZ|LME|RAZ|SCE|
|                              | E |       |       |    |   |   |   |   |   |   |
 --------------------------------------------------------------------------------
```

すべてのフィールドを詳細に示しているわけではありませんが、特別なパートでこれと他の`MSRs`については学びます。`edx:eax`の`EFFR`を読むとき、`btsl`命令で`_EFER_SCE`または`System Call Extensions`であるゼロビットをチェックし、1に設定します。`SCE`ビットをセットすることにより、`SYSCALL`と`SYSRET`命令が有効になります。次のステップでは、`edi`の20番目のビットを確認します。このレジスタには、`cpuid`(上記参照)の結果が格納されることに注意してください。`20`ビットがセット(`NX`ビット)されている場合、`EFER_SCE`をモデル固有レジスタ(Model Specific Register)に単に書き込みます。
We will not see all fields in details here, but we will learn about this and other `MSRs` in a special part about it. As we read `EFER` to the `edx:eax`, we check `_EFER_SCE` or zero bit which is `System Call Extensions` with `btsl` instruction and set it to one. 
By the setting `SCE` bit we enable `SYSCALL` and `SYSRET` instructions. In the next step we check 20th bit in the `edi`, remember that this register stores result of the `cpuid` (see above). If `20` bit is set (`NX` bit) we just write `EFER_SCE` to the model specific register.

```assembly
	btsl	$_EFER_SCE, %eax
	btl	$20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

[NX](https://en.wikipedia.org/wiki/NX_bit)ビットがサポートされている場合、`_EFER_NX`を有効にし、`wrmsr`命令で書き込みます。 [NX](https://en.wikipedia.org/wiki/NX_bi)ビットがセットされた後、`cr0`[control register](https://en.wikipedia.org/wiki/Control_register)のいくつかのビットを次のアセンブリコードによりセットします。
If the [NX](https://en.wikipedia.org/wiki/NX_bit) bit is supported we enable `_EFER_NX`  and write it too, with the `wrmsr` instruction. After the [NX](https://en.wikipedia.org/wiki/NX_bit) bit is set, we set some bits in the `cr0` [control register](https://en.wikipedia.org/wiki/Control_register) with following assembly code:

```assembly
	movl	$CR0_STATE, %eax
	movq	%rax, %cr0
```

特に次のビットです。
specifically the following bits:

* `X86_CR0_PE` - システムは保護モード(protected mode)です。
* `X86_CR0_MP` - WAIT/FWAIT命令とCR0のTSフラグとの相互作用を制御します。
* `X86_CR0_ET` - 386では、外部数値演算コプロセッサが80287であるか、80387であるかを指定できます。
* `X86_CR0_NE` - セット時、内部のx87浮動小数点エラーレポートを有効にします。それ以外の場合はPCスタイルのx87エラー検出を有効にします。
* `X86_CR0_WP` - セット時、特権レベルが0の場合、CPUは読み取り専用(read-only)のページに書き込めません。
* `X86_CR0_AM` - AMがセットされ、ACフラグ(EFLAGSレジスタ上に存在)がセットされ、かつ、特権レベルが3の場合、アライメントチェックが有効になります。
* `X86_CR0_PG` - ページングを有効にします。
* `X86_CR0_PE` - system is in protected mode;
* `X86_CR0_MP` - controls interaction of WAIT/FWAIT instructions with TS flag in CR0;
* `X86_CR0_ET` - on the 386, it allowed to specify whether the external math coprocessor was an 80287 or 80387;
* `X86_CR0_NE` - enable internal x87 floating point error reporting when set, else enables PC style x87 error detection;
* `X86_CR0_WP` - when set, the CPU can't write to read-only pages when privilege level is 0;
* `X86_CR0_AM` - alignment check enabled if AM set, AC flag (in EFLAGS register) set, and privilege level is 3;
* `X86_CR0_PG` - enable paging.

さて、アセンブリから[C](https://en.wikipedia.org/wiki/C_%28programming_language%29)のコードを実行するには、スタックをセットアップする必要があることを既に知っています。いつものように、メモリの正しい位置に[スタックポインタ](https://en.wikipedia.org/wiki/Stack_register)を設定し、その後に、[フラグ](https://en.wikipedia.org/wiki/FLAGS_register)レジスタをリセットすることでそれを行っています。
We already know that to run any code, and even more [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) code from assembly, we need to setup a stack. As always, we are doing it by the setting of [stack pointer](https://en.wikipedia.org/wiki/Stack_register) to a correct place in memory and resetting [flags](https://en.wikipedia.org/wiki/FLAGS_register) register after this:

```assembly
	movq initial_stack(%rip), %rsp
	pushq $0
	popfq
```

ここで最も興味深いのは、`initial_stack`です。このシンボルは、[ソースコード](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) ファイルで定義され、次のようになります。
The most interesting thing here is the `initial_stack`. This symbol is defined in the [source](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) code file and looks like:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`THREAD_SIZE`マクロは[arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) に定義され、`KASAN_STACK_ORDER`マクロの値に依存しています。
The `THREAD_SIZE` macro is defined in the [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h) header file and depends on value of the `KASAN_STACK_ORDER` macro:

```C
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

[kasan](https://github.com/torvalds/linux/blob/master/Documentation/dev-tools/kasan.rst)が無効化されており、`PAGE_SIZE`が`4096`バイトの場合で考えます。そのため、`THREAD_SIZE`は`16`キロバイトに拡張され、これはスレッドのスタックサイズを表します。なぜ`thread`なのでしょうか？各[プロセス](https://en.wikipedia.org/wiki/Process_%28computing%29)は、[親プロセス](https://en.wikipedia.org/wiki/Parent_process)と[子プロセス](https://en.wikipedia.org/wiki/Child_process)を持っていることを既に知っているかもしれません。実際には、親プロセスと子プロセスはスタックが異なります。新しいプロセス用に新しいkernelスタックが割り当てられます。Linux Kernelでは、このスタックは`thread_info`構造体を持った[共用体](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)として表現されます。
We consider when the [kasan](https://github.com/torvalds/linux/blob/master/Documentation/dev-tools/kasan.rst) is disabled and the `PAGE_SIZE` is `4096` bytes. So the `THREAD_SIZE` will expands to `16` kilobytes and represents size of the stack of a thread. 
Why is `thread`? You may already know that each [process](https://en.wikipedia.org/wiki/Process_%28computing%29) may have [parent processes](https://en.wikipedia.org/wiki/Parent_process) and [child processes](https://en.wikipedia.org/wiki/Child_process). Actually, a parent process and child process differ in stack. A new kernel stack is allocated for a new process. In the Linux kernel this stack is represented by the [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B) with the `thread_info` structure.

`init_thread_union`は`thread_union`によって表現されます。そして、`thraed_union`は[include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h)に次のように定義されています。
The `init_thread_union` is represented by the `thread_union`. And the `thread_union` is defined in the [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) file like the following:

```C
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

`CONFIG_ARCH_TASK_STRUCT_ON_STACK`kernelコンフィグオプションは`ia64`アーキテクチャのみ有効であり、`CONFIG_THREAD_INFO_IN_TASK`は`x86_64`アーキテクチャで有効になります。それゆえ、`thread_info`構造体は`thread_union`共用体の代わりに`task_struct`構造体に配置されます。
The `CONFIG_ARCH_TASK_STRUCT_ON_STACK` kernel configuration option is only enabled for `ia64` architecture, and the `CONFIG_THREAD_INFO_IN_TASK` kernel configuration option is enabled for `x86_64` architecture. Thus the `thread_info` structure will be placed in `task_struct` structure instead of the `thread_union` union.

`init_thread_union`は[include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/blob/master/include/asm-generic/vmlinux.lds.h)の`INIT_TASK_DATA`マクロの一部に位置し、次のようになります。
The `init_thread_union` is placed in the [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/blob/master/include/asm-generic/vmlinux.lds.h) file as part of the `INIT_TASK_DATA` macro like the following:

```C
#define INIT_TASK_DATA(align)  \
	. = ALIGN(align);      \
	...                    \
	init_thread_union = .; \
	...
```

このマクロは[arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)で使用されており、次のようになります。
This macro is used in the [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) file like the following:

```
.data : AT(ADDR(.data) - LOAD_OFFSET) {
	...
	INIT_TASK_DATA(THREAD_SIZE)
	...
} :data
```

つまり、`init_thread_union`は`16`キロバイトの`THREAD_SIZE`でアラインされた、アドレスで初期化されます。
That is, `init_thread_union` is initialized with the address which is aligned to `THREAD_SIZE` which is `16` kilobytes.

さて、これで次の式を理解できます。
Now we may understand this expression:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`initial_stack`シンボルは`thread_union.stack`配列の先頭 + `16`キロバイトの`THREAD_SIZE` - `SIZEOF_PTREGS`の先頭を指します。`SIZEOF_PTREGS`は、kernel内アンワインダーがスタックの末尾を確実に検出するのに役立つ規則です。
that `initial_stack` symbol points to the start of the `thread_union.stack` array + `THREAD_SIZE` which is 16 killobytes and - `SIZEOF_PTREGS` which is convention which helps the in-kernel unwinder reliably detect the end of the stack.

初期ブートスタックがセットされた後、[Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)を`lgdt`命令で更新します。
After the early boot stack is set, to update the [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) with the `lgdt` instruction:

```assembly
lgdt	early_gdt_descr(%rip)
```

`early_gdt_descr`は次のように定義されます。
where the `early_gdt_descr` is defined as:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

kernelは低いユーザー空間アドレスで動作するようになりましたが、すぐにkernelは独自のスペースで動作するため、`Global Descriptor Table`をリロードする必要があります。
We need to reload `Global Descriptor Table` because now kernel works in the low userspace addresses, but soon kernel will work in its own space.

さて、`early_gdt_descr`の定義を見てみましょう。`GOT_ENTRIES`は`32`に展開されるので、Global Descriptor Tableにはkernelコード、データ、スレッド、ローカルストレージセグメントなどの`32`エントリが含まれます。
Now let's look at the definition of `early_gdt_descr`. `GDT_ENTRIES` expands to `32` so that Global Descriptor Table contains `32` entries for kernel code, data, thread local storage segments and etc...

では、`early_gdt_descr_base`の定義を見てみましょう。`gdt_page`構造体は[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)に次のように定義されています。
Now let's look at the definition of `early_gdt_descr_base`. The `gdt_page` structure is defined in the [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h) as:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```

これは、次のように定義される`desc_struct`構造体の配列である`gdt`フィールドを一つ持ちます。
It contains one field `gdt` which is array of the `desc_struct` structure which is defined as:

```C
struct desc_struct {
         union {
                 struct {
                         unsigned int a;
                         unsigned int b;
                 };
                 struct {
                         u16 limit0;
                         u16 base0;
                         unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
                         unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
                 };
         };
 } __attribute__((packed));
```

おなじみの`GDT`ディスクリプタに見えます。`gdt_page`構造体は、`4096`バイトの`PAGE_SIZE`にアラインされていることに注意してください。つまり、`gdt`は1ページを占有します。
which looks familiar `GDT` descriptor. Note that `gdt_page` structure is aligned to `PAGE_SIZE` which is `4096` bytes. Which means that `gdt` will occupy one page.

それでは、`INIT_PER_CPU_VAR`が何かを見ていきましょう。`INIT_PER_CPU_VAR`は[arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h)に定義されているマクロで、単に`init_per_cpu__`と与えられたパラメータを連結するだけです。
Now let's try to understand what `INIT_PER_CPU_VAR` is. `INIT_PER_CPU_VAR` is a macro which is defined in the [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h) and just concatenates `init_per_cpu__` with the given parameter:

```C
#define INIT_PER_CPU_VAR(var) init_per_cpu__##var
```

`INIT_PER_CPU_VAR`マクロが展開された後、`init_per_cpu__gpt_page`が得られます。[リンカスクリプト](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)での`init_per_cpu__gpt_page`の初期化は次のようになります。
After the `INIT_PER_CPU_VAR` macro will be expanded, we will have `init_per_cpu__gdt_page`. We can see the initialization of `init_per_cpu__gdt_page` in the [linker script](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S):

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

`INIT_PER_CPU_VER`で`init_per_cpu__gpt_page`を取得し、リンカスクリプトから`INIT_PER_CPU`マクロが展開されれば、`__per_cpu_load`からの？オフセットを得ることができます。この計算の後、新しいGDTの正しいベースアドレスを得られます。
As we got `init_per_cpu__gdt_page` in `INIT_PER_CPU_VAR` and `INIT_PER_CPU` macro from linker script will be expanded we will get offset from the `__per_cpu_load`. After this calculations, we will have correct base address of the new GDT.

通常、CPUごとの変数は2.6 kernelの機能です。名前からそれが何であるかを理解できます。`CPU毎の(per-CPU)`変数を作成する際、各CPUはこの変数の独自のコピーを持ちます。ここでは、CPUごとの変数として`gpt_page`を作成しています。このタイプの変数には、各CPUが独自のコピーした変数を利用して動作するため、ロックがいらないなどの多くの利点があります。そのため、マルチプロセッサのすべてのコアは独自の`GDT`テーブルを持っており、テーブル内のすべてのエントリは、コア上で実行されたスレッドからアクセスできるメモリセグメントを表現しています。`per-CPU`の変数に関する詳細は、[Concepts/per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)の投稿で読むことができます。
Generally per-CPU variables is a 2.6 kernel feature. You can understand what it is from its name. When we create `per-CPU` variable, each CPU will have its own copy of this variable. Here we are creating `gdt_page` per-CPU variable. 
There are many advantages for variables of this type, like there are no locks, because each CPU works with its own copy of variable and etc... 
So every core on multiprocessor will have its own `GDT` table and every entry in the table will represent a memory segment which can be accessed from the thread which ran on the core. You can read in details about `per-CPU` variables in the [Concepts/per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) post.

新しいGlobal Descriptor Tableをロードした際、毎回行っていたようにセグメントをリロードします。
As we loaded new Global Descriptor Table, we reload segments as we did it every time:

```assembly
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
```

これらのすべてのステップの後、[割り込み](https://en.wikipedia.org/wiki/Interrupt)が処理される特別なスタックを表す`irqstack`に配置する`gs`レジスタを設定します。？
After all of these steps we set up `gs` register that it post to the `irqstack` which represents special stack where [interrupts](https://en.wikipedia.org/wiki/Interrupt) will be handled on:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

`MSR_GS_BASE`の場所は以下になります。
where `MSR_GS_BASE` is:

```C
#define MSR_GS_BASE             0xc0000101
```

`MSG_GS_BASE`を`ecx`レジスタへ入れ、`wrmsr`命令により`eax`と`edx`(`inital_gs`を指す)からデータをロードする必要があります。64bitモードでのアドレッシングでは、`cs`、`fs`、`ds`、そして`ss`セグメントレジスタは利用しませんが、`fs`と`gs`レジスタは使用できます。`fs`と`gs`は隠された部分(`cs`のリアルモードで見たように)を持っており、この部分は[Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register)にマッピングされるディスクリプタを含んでいます。したがって、上記の`0xc0000101`は`gs.base`のMSRのアドレスです。[システムコール](https://en.wikipedia.org/wiki/System_call)や[割り込み](https://en.wikipedia.org/wiki/Interrupt)が発生した場合は、エントリポイント時点ではkernelスタックがないため、`MSG_GS_BASE`の値は割り込みスタックのアドレスを保持しています。
We need to put `MSR_GS_BASE` to the `ecx` register and load data from the `eax` and `edx` (which point to the `initial_gs`) with `wrmsr` instruction. 
We don't use `cs`, `fs`, `ds` and `ss` segment registers for addressing in the 64-bit mode, but `fs` and `gs` registers can be used. 
`fs` and `gs` have a hidden part (as we saw it in the real mode for `cs`) and this part contains a descriptor which is mapped to [Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register).
So we can see above `0xc0000101` is a `gs.base` MSR address. When a [system call](https://en.wikipedia.org/wiki/System_call) or [interrupt](https://en.wikipedia.org/wiki/Interrupt) occurs, there is no kernel stack at the entry point, so the value of the `MSR_GS_BASE` will store address of the interrupt stack.

次のステップでは、リアルモードのbootparam構造体のアドレスを`dri`に入れ(`rsi`はこの構造体のポインタを最初から保持していることを思い出してください)、次のようにCコードにジャンプします。
In the next step we put the address of the real mode bootparam structure to the `rdi` (remember `rsi` holds pointer to this structure from the start) and jump to the C code with:

```assembly
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorq	%rbp, %rbp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
.Lafter_lret:
```

`initial_code`のアドレスを`rax`へ配置し、リターンアドレスと、`__KERNEL_CS`と`initial_code`のアドレスをスタックへプッシュします。この後、`lretq`命令が配置されていますが、これはリターンアドレスをスタック(スタックには現在`initial_code`のアドレスがある)から抽出し、そこにジャンプすることを意味します。`initial_code`は同じソースコードファイルに定義されており、次のようになります。
Here we put the address of the `initial_code` to the `rax` and push the return address, `__KERNEL_CS` and the address of the `initial_code` to the stack. After this we can see `lretq` instruction which means that after it return address will be extracted from stack (now there is address of the `initial_code`) and jump there. `initial_code` is defined in the same source code file and looks:

```assembly
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	...
	...
	...
```

ごらんの通り`initial_code`は、[arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)に定義されている`x86_64_start_kernel`のアドレスを含んでおり、次のようになります。
As we can see `initial_code` contains address of the `x86_64_start_kernel`, which is defined in the [arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c) and looks like this:

```C
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
	...
	...
	...
}
```

引数は`real_mode_delta`一つです。(リアルモードデータのアドレスを`rdi`レジスタに以前渡したことを思い出してください)
It has one argument is a `real_mode_data` (remember that we passed address of the real mode data to the `rdi` register previously).

start_kernel
Next to start_kernel
--------------------------------------------------------------------------------

"kernelのエントリポイント"、つまり[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)に定義されるstart_kernel関数を見る前に、最後の準備を確認する必要があります。
We need to see last preparations before we can see "kernel entry point" - start_kernel function from the [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c).

まず、`x86_64_start_kernel`関数内にていくつかチェックがあります。
First of all we can see some checks in the `x86_64_start_kernel` function:

```C
BUILD_BUG_ON(MODULES_VADDR < __START_KERNEL_map);
BUILD_BUG_ON(MODULES_VADDR - __START_KERNEL_map < KERNEL_IMAGE_SIZE);
BUILD_BUG_ON(MODULES_LEN + KERNEL_IMAGE_SIZE > 2*PUD_SIZE);
BUILD_BUG_ON((__START_KERNEL_map & ~PMD_MASK) != 0);
BUILD_BUG_ON((MODULES_VADDR & ~PMD_MASK) != 0);
BUILD_BUG_ON(!(MODULES_VADDR > __START_KERNEL));
MAYBE_BUILD_BUG_ON(!(((MODULES_END - 1) & PGDIR_MASK) == (__START_KERNEL & PGDIR_MASK)));
BUILD_BUG_ON(__fix_to_virt(__end_of_fixed_addresses) <= MODULES_END);
```

モジュール空間の仮想アドレスがカーネルテキストのベースアドレス以上である(`__STAT_KERNEL_map`に対しても同様)、モジュールを含むカーネルテキストは少なくともカーネルのイメージである、など、様々なものに対するチェックがあります。`BUILD_BUG_ON`はマクロで、次のようになります。
There are checks for different things like virtual address of module space is not fewer than base address of the kernel text - `__STAT_KERNEL_map`, that kernel text with modules is not less than image of the kernel and etc... `BUILD_BUG_ON` is a macro which looks as:

```C
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

このトリックの仕組みを理解していきましょう。例えば、最初の条件である`MODULES_VADDR < __START_KERNEL_map`に関して考えてみましょう。`!!conditions`は、`condition != 0`と同じです。そのため、`MODULES_VADDR < __START_KERNEL_map`が真でであれば、`!!(condition)`にて`1`を取得し、そうでなければゼロを取得します。つまり、`2*!!(condition)` の後で、`2`か`0`を取得できます。そして、計算の最後にて以下の二つの異なる動作が取得できます。
Let's try to understand how this trick works. Let's take for example first condition: `MODULES_VADDR < __START_KERNEL_map`. `!!conditions` is the same that `condition != 0`. So it means if `MODULES_VADDR < __START_KERNEL_map` is true, we will get `1` in the `!!(condition)` or zero if not. After `2*!!(condition)` we will get or `2` or `0`. In the end of calculations we can get two different behaviors:

* 負のインデックスを持つchar配列のサイズを取得しようとするため、コンパイルエラーが発生します。(`MODULES_VADDR`が`__START_KERNEL_map`を下回ることがないため、このケースも同様です)
* コンパイルエラーは発生しません。
* We will have compilation error, because try to get size of the char array with negative index (as can be in our case, because `MODULES_VADDR` can't be less than `__START_KERNEL_map` will be in our case);
* No compilation errors.

それですべてです。いくつかの定数に依存するコンパイルエラーを得るためのとても興味深いCのトリックです。
That's all. So interesting C trick for getting compile error which depends on some constants.

次のステップでは、各CPUの`cr4`のシャドウコピーを保存する`cr4_init_shadow`関数の呼び出しを見ます。コンテキストスイッチは`cr4`のビットを変更することがあるので、CPUごとの`cr4`を保存する必要があります。そしてこの後、すべてのページグローバルディレクトリエントリをリセットし、`cr3`のPGTへの新しいポインタを書き込む`reset_early_page_tables`関数の呼び出しがあります。
In the next step we can see call of the `cr4_init_shadow` function which stores shadow copy of the `cr4` per cpu. Context switches can change bits in the `cr4` so we need to store `cr4` for each CPU. And after this we can see call of the `reset_early_page_tables` function where we resets all page global directory entries and write new pointer to the PGT in `cr3`:

```C
	memset(early_top_pgt, 0, sizeof(pgd_t)*(PTRS_PER_PGD-1));
	next_early_pgt = 0;
	write_cr3(__sme_pa_nodebug(early_top_pgt));
```

次に、すぐに新しいページテーブルを作成します。ここでは、すべてのページグローバルディレクトリエントリがゼロであることがわかります。この後、`next_early_pgt`を0にセットし(詳細については次の投稿で説明します)、`early_top_pgt`の物理アドレスを`cr3`に書き込みます。
Soon we will build new page tables. Here we can see that we zero all Page Global Directory entries. After this we set `next_early_pgt` to zero (we will see details about it in the next post) and write physical address of the `early_top_pgt` to the `cr3`.

この後、`_bss`を、`__bss_stop`から`__bss_start`までクリアし、`init_top_pgt`もまたクリアします。`init_top_pgt`は[arch/x86/kerne/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)に以下のように定義されています。
After this we clear `_bss` from the `__bss_stop` to `__bss_start` and also clear `init_top_pgt`. `init_top_pgt` is defined in the [arch/x86/kerne/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) like the following:

```assembly
NEXT_PGD_PAGE(init_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0
``` 

これは`early_top_pgt`と全く同じ定義です。
This is exactly the same definition as `early_top_pgt`.

次のステップは、初期`IDT`ハンドラのセットアップですが、それは大きなコンセプトなので次の投稿で見ていくことにします。
The next step will be setup of the early `IDT` handlers, but it's big concept so we will see it in the next post.

結論
--------------------------------------------------------------------------------

これでLinuxカーネルの初期化に関する最初のパートは終わりです。
This is the end of the first part about linux kernel initialization.

XXX
If you have questions or suggestions, feel free to ping me in twitter [0xAX](https://twitter.com/0xAX), drop me [email](anotherworldofworld@gmail.com) or just create [issue](https://github.com/0xAX/linux-insides/issues/new).

次のパートでは、早期割り込みハンドラの初期化、カーネル空間のメモリマッピングについて説明します。
In the next part we will see initialization of the early interruption handlers, kernel space memory mapping and a lot more.

XXX
**Please note that English is not my first language and I am really sorry for any inconvenience. If you found any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

リンク集
--------------------------------------------------------------------------------

* [Model Specific Register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)
* [Previous part - kernel load address randomization](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization)
