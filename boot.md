# BitVisorのブート処理

## 参考文献

- [BitVisorのUEFI対応](https://www.bitvisor.org/summit2/slides/bitvisor-summit-2-03-eiraku.pdf)
- [BitVisorの仮想メモリーマップ](https://qiita.com/hdk_2/items/6c7aaa72f5dcfcfda342)
- [BitVisor本体のブート仕様#bootuefi-loader](https://qiita.com/hdk_2/items/b73161f08fefce0d99c3#bootuefi-loader)

## 豆知識

- `mov label, %rax` と `mov label(%rip), %rax` は、やっていることは変わらないが、前者は絶対アドレス、後者は相対アドレスになる
- `AllocatePages()` に `AllocateMaxAddress` を指定して呼び出すことで、ファームウェアが空き領域を見つけてくれる
- メモリタイプを例えば `EfiUnusableMemory` としておくことで、ファームウェア、他のUEFIアプリケーションやOSがその領域を使用することはなくなる

## UEFIの場合

`boot/uefi-loader/loadvmm.c` を見る。

やっていることは `bitvisor.elf` の先頭0x10000バイトを読み込んで0x40000000以下のアドレスに配置、引数を渡して `entry.s` の `entry` に飛んでいると思われる。

`bitvisor.elf` の先頭0x10000バイトが2nd VMM loaderで、本体は0x40000000-0x47ffffffに配置しているよう。

処理の順番。

```
efi_main -> entry -> uefi64_entry -> uefi_init -> uefi_entry_start -> callmain64 -> vmm_main(BSP)
                                                                                 -> apinitproc0(AP)
```

### `entry` ( `entry.s` )

ただ `uefi64_entry` に飛んでいるだけ。

### `uefi64_entry` ( `entry.s` )

1. スタックに `entry` の引数をpush
2. `uefi_entry_save_regs` でレジスタを保存
3. ページテーブル構築
4. GDTR設定
5. CR3設定
6. スタックを設定し、 `uefi_init` をcall

### `uefi_init` ( `uefi.c` )

色々設定して `uefi_entry_start` をcall。

### `uefi_entry_start` ( `entry.s` )

`callmain64` にjump。

### `callmain64` ( `entry.s` )

BSPだったら `vmm_main` をcall。
APだったら `apinitproc0` をcall。

## `vmm_main` 以降(BSP)

### `vmm_main` ( `main.c` )
`start_all_processors` に渡している `bsp_proc` が実行される。

### `bsp_proc` ( `main.c` )

`call_initfunc` の `pcpu` の最後が `create_pass_vm` で、これが最終的に呼ばれる。

### `create_pass_vm` ( `main.c` )

1. `current->vmctl.vminit` 内で `vmxon` をcall
2. `vmmcall_boot_enable` に `bsp_init_thread` を引数に渡して実行
3. `vmctl.start_vm` をcall

### `vmmcall_boot_enable` ( `vmmcall_boot.c` )

1. `thread_new` で引数に `vmmcall_boot_thread` と `bsp_init_thread` の情報を渡してスレッドを作成
2. `wait_for_boot_continue` をcall

### `wait_for_boot_continue` ( `vmmcall_boot.c` )

1. `continue_flag = false`
2. `continue_flag == false` の間 `schedule`

### `vmmcall_boot_thread` ( `vmmcall_boot.c` )

1. 引数の関数を実行( `bsp_init_thread` )
2. `continue_flag = true` , `enable = false`

### `bsp_init_thread` ( `main.c` )

1. `initregs` でVMのレジスタ初期化
2. `copy_uefi_bootcode` でVMの先頭のコードを配置
  - `uefi_entry_ret_addr` に格納されている数値を用いて `ljmpl` するコードを配置する

### `vt_start_vm` ( `vt_main.c` )

`vt_mainloop` をcall。

### `vt_mainloop` ( `vt_main.c` )

1. `schedule` はスレッドが無いのでスキップ
2. おそらく `vt__vm_run` あたりが呼び出される

### `vt__vm_run` ( `vt_main.c` )

1. `vt__vm_run_first` が呼び出され、 `vmlaunch` して `copy_uefi_bootcode` で配置したコードに飛ぶ