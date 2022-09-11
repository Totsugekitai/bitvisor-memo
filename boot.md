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
entry -> uefi64_entry -> uefi_init -> uefi_entry_start -> callmain64 -> vmm_main(BSP)
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