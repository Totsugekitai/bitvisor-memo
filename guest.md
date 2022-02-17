# VMM側の機能の登録API(`core/vmmcall.c`)

## `void vmmcall_register (char *name, vmmcall_func_t func)`
`func` と `name` を結びつける。

# ゲストからのVMM呼び出しAPI(`tools/common/call_vmm.*`)

## `CALL_VMM_GET_FUNCTION ("function name", call_vmm_function_t *vmm_function)`
`function name` を渡すことで呼び出しの準備をし、 `vmm_function` に情報を格納する。

マクロであり、インラインアセンブリにより展開されるため、 `function name` は文字列リテラルでなくてはいけない。

## `int call_vmm_function_callable (call_vmm_function_t *f)`
VMM呼び出しが使用可能かどうか判定する。
VMM呼び出し命令が使えない場合、または `function name` が正しくない場合、 `0` を返す。
VMM呼び出し命令が使える場合、 `1` を返す。

## `int call_vmm_function_no_vmm (call_vmm_function_t *f)`
ファンクション番号を返す関数。

## `void call_vmm_call_function (call_vmm_function_t *function, call_vmm_arg_t *arg, call_vmm_ret_t *ret)`
VMM呼び出しを行う。

`call_vmm_arg_t` および `call_vmm_ret_t` にはレジスタの値を渡す。
`call_vmm_arg_t` は `rbx, rcx, rdx, rsi, rdi` を、 `call_vmm_ret_t` は加えて `rax` を渡す。
