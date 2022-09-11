# BitVisor コードリーディングメモ

- BitVisor-2.0 Last Update: 2022-02-09 を読む

## 参考資料

- [BitVisor本体のブート仕様](https://qiita.com/hdk_2/items/b73161f08fefce0d99c3)
- [BitVisor ブート時に通るソースコードを辿ってみる](https://qiita.com/deep_tkkn/items/a05f87749ee352547e40)
- [BitVisorの仮想メモリーマップ](https://qiita.com/hdk_2/items/6c7aaa72f5dcfcfda342)
- [BitVisorのVMM呼び出しAPI](https://qiita.com/hdk_2/items/c8c47ac91ab59b75549c)

## 雑多なものの解説

### `asmlinkage`(`include/core/linkage.h:33`)
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

## `vmm_main`(`core/main.c:513`)
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

## `start_all_processors`(`core/ap.c:397`)
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

## `bsp_continue`(`core/ap.c:314`)
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

## `asm_wrrsp_and_jmp`(`core/asm.h:722`)
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

## `bspinitproc1`(`core/ap.c:103`)
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

## `ap_start`(`core/ap.c:270`)
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
APをすべてスタートしているっぽい。

物理アドレス0番地に `ljmpw` でどこかに飛ぶようなコードを設定している。

`void *mapmem_hphys (u64 physaddr, uint len, int flags)` は `physaddr` から `physaddr + len - 1` を確保する関数のようだ。
`hphys` の `h` はhostのhだろう。

`void *mapmem (int flags, u64 physaddr, uint len)` は `physaddr` から `physaddr + len - 1` を確保する関数のようだが、内部で `mapped_hphys_addr()` と `mapped_gphys_addr()` に分岐している。
ホストとゲスト両対応の関数だと思われる。
ここでは `flag` に `MAPMEM_HPHYS | MAPMEM_WRITE` を設定しているので、ホストの物理アドレスを書き込み可能にしていると思われる。

`for` 文では多分APが起動するのを待っている。
最後に起動したフラグを `true` にしてエンド。

## `bsp_proc`(`core/main.c:505`)
`bspinitproc1()` に戻ると、最終的には `initproc_bsp()` を呼び出している。
`initproc_bsp()` は関数ポインタで、これは `vmm_main()` で `bsp_proc()` を代入されている。
```c
static void
bsp_proc (void)
{
	call_initfunc ("bsp");
	call_parallel ();
	call_initfunc ("pcpu");
}
```
`call_initfunc()` で登録された初期化関数を呼び出している。

`call_initfunc ("pcpu")` で最後に `create_pass_vm()` という関数が呼ばれるらしい。

`call_parallel()` を追う。

## `call_parallel`(`core/main.c:476`)
```c
static void
call_parallel (void)
{
	static struct {
		char *name;
		ulong not_called;
	} paral[] = {
		{ "paral0", 1 },
		{ "paral1", 1 },
		{ "paral2", 1 },
		{ "paral3", 1 },
		{ NULL, 0 }
	};
	int i;

	for (i = 0; paral[i].name; i++) {
		if (asm_lock_ulong_swap (&paral[i].not_called, 0))
			call_initfunc (paral[i].name);
	}
}
```
これも結局 `paralx` でマークされた初期化関数を呼び出しているようである。

## `create_pass_vm` ( `core/main.c` )

```c
static void
create_pass_vm (void)
{
	bool bsp = false;
	static struct vcpu *vcpu0;

	if (currentcpu->cpunum == 0)
		bsp = true;
	sync_all_processors ();
	if (bsp) {
		load_new_vcpu (NULL);
		vcpu0 = current;
	}
	sync_all_processors ();
	if (!bsp)
		load_new_vcpu (vcpu0);
	set_fullvirtualize ();
	sync_all_processors ();
	current->pass_vm = true;
	current->vmctl.vminit ();
	call_initfunc ("pass");
	sync_all_processors ();
	if (bsp) {
		vmmcall_boot_enable (bsp_init_thread, NULL);
	} else {
		initregs ();
		current->vmctl.init_signal ();
	}
	current->vmctl.enable_resume ();
	current->initialized = true;
	sync_all_processors ();
	if (bsp)
		print_startvm_msg ();
	currentcpu->pass_vm_created = true;
#ifdef DEBUG_GDB
	if (!bsp)
		for (;;)
			asm_cli_and_hlt ();
#endif
	current->vmctl.start_vm ();
	panic ("VM stopped.");
}
```

BSP、APともに最終的に `current->vmctl.start_vm()` を呼び出してそこから戻らない。
ここからはメーカー別の処理になる。

## `vt_start_vm` ( `core/vt_main.c` )

```c
void
vt_start_vm (void)
{
	vt_paging_start ();
	vt_mainloop ();
}
```

`vt_mainloop` をcall。

## `vt_mainloop` ( `core/vt_main.c` )

```c
static void
vt_mainloop (void)
{
	enum vmmerr err;
	ulong cr0, acr;
	u64 efer;
	bool nmi;

	for (;;) {
		schedule ();
		vt_vmptrld (current->u.vt.vi.vmcs_region_phys);
		panic_test ();
		if (current->initipi.get_init_count ())
			vt_init_signal ();
		if (current->halt) {
			vt__halt ();
			current->halt = false;
			continue;
		}
		/* when the state is switching between real mode and
		   protected mode, we try emulation first */
		/* SWITCHING:
		   mov  %cr0,%eax
		   or   $CR0_PE_BIT,%eax
		   mov  %eax,%cr0
		   ljmp $0x8,$1f       | SWITCHING STATE
		   1:                  |
		   mov  $0x10,%eax     | segment registers hold the contents
		   mov  %eax,%ds       | in previous mode. we use interpreter
		   mov  %eax,%es       | to emulate this situation.
		   mov  %eax,%fs       | maximum 32 instructions are emulated
		   mov  %eax,%gs       | because the interpreter is too slow.
		   mov  %eax,%ss       |
		   ...
		 */
		if (current->u.vt.vr.sw.enable) {
			current->u.vt.vr.sw.num++;
			if (current->u.vt.vr.sw.num >= 32) {
				/* consider emulation is not needed after
				   32 instructions are executed */
				current->u.vt.vr.sw.enable = 0;
				vt_update_exception_bmp ();
				continue;
			}
			vt_read_control_reg (CONTROL_REG_CR0, &cr0);
			if (cr0 & CR0_PG_BIT) {
				vt_read_msr (MSR_IA32_EFER, &efer);
				if (efer & MSR_IA32_EFER_LME_BIT) {
					vt_read_sreg_acr (SREG_CS, &acr);
					if (acr & ACCESS_RIGHTS_L_BIT) {
						/* long mode */
						current->u.vt.vr.sw.enable = 0;
						vt_update_exception_bmp ();
						continue;
					}
				}
			}
			err = cpu_interpreter ();
			if (err == VMMERR_SUCCESS) /* emulation successful */
				continue;
			else if (err == VMMERR_UNSUPPORTED_OPCODE ||
				 err == VMMERR_SW)
				; /* unsupported/run as it is */
			else	/* failed */
				panic ("vt_mainloop ERR %d", err);
			/* continue when the instruction is not supported
			   or should be executed as it is.
			   (sw.enable may be changed after cpu_interpreter())
			*/
		}
		/* when the state is switching, do single step */
		if (current->u.vt.vr.sw.enable) {
			vt__nmi ();
			vt_interrupt ();
			vt__event_delivery_setup ();
			vt_msr_own_process_msrs ();
			nmi = vt__vm_run_with_tf ();
			vt_paging_tlbflush ();
			if (!nmi) {
				vt__event_delivery_check ();
				vt__exit_reason ();
			}
		} else {	/* not switching */
			vt__nmi ();
			vt_interrupt ();
			vt__event_delivery_setup ();
			vt_msr_own_process_msrs ();
			nmi = vt__vm_run ();
			vt_paging_tlbflush ();
			if (!nmi) {
				vt__event_delivery_check ();
				vt__exit_reason ();
			}
		}
	}
}
```