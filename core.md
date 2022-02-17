# BitVisor コードリーディングメモ

- BitVisor-2.0 Last Update: 2022-02-09 を読む

## 雑多なものの解説

### `asmlinkage`(include/core/linkage.h:33)
```c
#define asmlinkage __attribute__ ((regparm (0)))
```
`__attribute__ ((regparm (0)))` で引数のスタック渡しを強制する。

### BSP
BootStrap Processorの略。
電源投入時に最初に動くプロセッサコア。

### AP
Application Processorの略。
BSP以外のプロセッサコア。

# 関数

## `vmm_main`(core/main.c:513)
おそらくブートして一番最初に来る関数。
```c
asmlinkage void
vmm_main (struct multiboot_info *mi_arg)
{
	uefi_booted = !mi_arg;
	if (!uefi_booted)
		memcpy (&mi, mi_arg, sizeof (struct multiboot_info));
	initfunc_init ();
	call_initfunc ("global");
	start_all_processors (bsp_proc, ap_proc);
}
```
マルチブートから上がってきていたらその情報をグローバル変数 `mi` に格納。

`initfunc_init()` により `INITFUNC` マクロを使うための初期化していると思われる。

`call_initfunc("global")` で `INITFUNC("global1", foo);` のように登録された関数を呼び出し。
```c
INITFUNC("global1", foo);
INITFUNC("global2", bar);
INITFUNC("global3", baz);
```
このような登録がされていた場合、 `foo -> bar -> baz` の順番で呼ばれると思われる。

`start_all_processors` で全てのプロセッサコアを起動。

## `start_all_processors`(core/ap.c:397)
```c
void
start_all_processors (void (*bsp_initproc) (void), void (*ap_initproc) (void))
{
	initproc_bsp = bsp_initproc;
	initproc_ap = ap_initproc;
	bsp_continue (bspinitproc1);
}
```
最初2行は関数ポインタをグローバル変数に入れているだけ。

`bsp_continue(bspinitproc1);` でBSPの起動の続きを行う。
引数の `bspinitproc1` は関数ポインタ。

## `bsp_continue`(core/ap.c:314)
```c
static void
bsp_continue (asmlinkage void (*initproc_arg) (void))
{
	void *newstack;

	newstack = alloc (VMM_STACKSIZE);
	currentcpu->stackaddr = newstack;
	asm_wrrsp_and_jmp ((ulong)newstack + VMM_STACKSIZE, initproc_arg);
}
```
やっていることは、スタックを用意して引数の `initproc_arg` にジャンプである(と思われる)。

`asm_wrrsp_and_jmp` については次で解説する。

## `asm_wrrsp_and_jmp`(core/asm.h:722)
```c
static inline void
asm_wrrsp_and_jmp (ulong rsp, void *jmpto)
{
	asm volatile (
		"xor %%ebp,%%ebp; "
#ifdef __x86_64__
		"mov %0,%%rsp; jmp *%1"
#else
		"mov %0,%%esp; jmp *%1"
#endif
		:
		: "g" (rsp)
		, "r" (jmpto));
}
```
`rsp` を設定して、 `jmpto` にジャンプしている。

## `bspinitproc1`(core/ap.c:103)
```c
static asmlinkage void
bspinitproc1 (void)
{
	printf ("Processor 0 (BSP)\n");
	spinlock_init (&sync_lock);
	sync_id = 0;
	sync_count = 0;
	num_of_processors = 0;
	spinlock_init (&ap_lock);

	if (!uefi_booted) {
		apinit_addr = 0xF000;
		ap_start ();
	} else {
		apinit_addr = alloc_realmodemem (APINIT_SIZE + 15);
		ASSERT (apinit_addr >= 0x1000);
		while ((apinit_addr & 0xF) != (APINIT_OFFSET & 0xF))
			apinit_addr++;
		ASSERT (apinit_addr > APINIT_OFFSET);
		localapic_delayed_ap_start (ap_start);
	}
	initproc_bsp ();
	panic ("bspinitproc1");
}
```
`spinlock_init` でスピンロックの初期化をしている。
`sync_lock` はBSPのロック？ `ap_lock` はAPのロック？

UEFIブートでなければ、つまりマルチブートで上がってきていたら、 `apinit_addr = 0xF000` に設定し `ap_start();` を実行。

UEFIブートであったら、 `alloc_realmodemem` でメモリ確保を行う。
`alloc_realmodemem` はUEFIまたはBIOSの機能を用いてメモリアロケーションを行う関数(と思われる)。
`while` 文で `apinit_addr` の調整をして、 `localapic_delayed_ap_start` を実行。

最後に、 `initproc_bsp` を実行。
ここからは戻ってこない。
戻ってきたら `panic` する。

## `ap_start`(core/ap.c:270)
```c
static void
ap_start (void)
{
	volatile u32 *num;
	u8 *apinit;
	u32 tmp;
	int i;
	u8 buf[5];
	u8 *p;
	u32 apinit_segment;

	apinit_segment = (apinit_addr - APINIT_OFFSET) >> 4;
	/* Put a "ljmpw" instruction to the physical address 0 */
	p = mapmem_hphys (0, 5, MAPMEM_WRITE);
	memcpy (buf, p, 5);
	p[0] = 0xEA;		/* ljmpw */
	p[1] = APINIT_OFFSET & 0xFF;
	p[2] = APINIT_OFFSET >> 8;
	p[3] = apinit_segment & 0xFF;
	p[4] = apinit_segment >> 8;
	apinit = mapmem (MAPMEM_HPHYS | MAPMEM_WRITE, apinit_addr,
			 APINIT_SIZE);
	ASSERT (apinit);
	memcpy (apinit, cpuinit_start, APINIT_SIZE);
	num = (volatile u32 *)APINIT_POINTER (apinit_procs);
	apinitlock = (spinlock_t *)APINIT_POINTER (apinit_lock);
	*num = 0;
	spinlock_init (apinitlock);
	i = 0;
	ap_start_addr (0, ap_start_loopcond, &i);
	for (;;) {
		spinlock_lock (&ap_lock);
		tmp = num_of_processors;
		spinlock_unlock (&ap_lock);
		if (*num == tmp)
			break;
		usleep (1000000);
	}
	unmapmem ((void *)apinit, APINIT_SIZE);
	memcpy (p, buf, 5);
	unmapmem (p, 5);
	ap_started = true;
}
```
