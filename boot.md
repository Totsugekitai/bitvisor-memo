# BitVisorのブート処理

## UEFIの場合

`boot/uefi-loader/loadvmm.c` を見る。

やっていることは `bitvisor.elf` の先頭0x10000バイトを読み込んで0x40000000以下のアドレスに配置、引数を渡して `entry.s` の `entry` に飛んでいると思われる。