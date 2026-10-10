# winproc handle / truncated-procedure fix (Staxel, Steam 405710)

Fixes total loss of input in Staxel: every window message is dispatched to a bogus
address, raising `EXCEPTION_ACCESS_VIOLATION`, so clicks are ignored until the
process wedges.

## The defect

`dlls/win32u/class.c:get_winproc_ptr()` recognises a winproc handle with

```c
if ((ULONG_PTR)handle >> 16 != WINPROC_HANDLE) return NULL;
```

Wine hands an application a winproc *handle* (`0xffff00xx`) instead of a raw
procedure pointer. Staxel's FNA/SDL + CefSharp subclassing keeps that value in a
32-bit field (the ANSI `SetWindowLong`/`GetWindowLong` APIs only carry a `LONG`),
so it arrives **sign-extended** as `0xffffffffffffff00xx`, the comparison fails,
and the handle is executed as a function pointer. The same round trip truncates a
real 64-bit procedure to its low 32 bits.

```
6943.936  alloc_winproc_ptr allocated 0xffff0012 for W 0x6ffffa641cb0
6961.671  EXCEPTION_ACCESS_VIOLATION addr=FFFFFFFFFA641CB0   # 0x6ffffa641cb0 truncated
```

Unpatched Proton 9.0-4 / Experimental / GE-Proton11-3 faulted at
`FFFFFFFFFFFF0012` - the same handle, sign-extended.

In testing, the patched build produced 0 access violations, and the ~30,000
`CallWindowProc` calls per session that carry a truncated procedure resolved.

## What it changes (only `dlls/win32u/class.c`)

1. `get_winproc_ptr()` accepts `0xffff00xx` and its sign-extended form.
2. `fixup_truncated_winproc()` recovers a truncated procedure from the winproc
   table by matching its low 32 bits - exact, since every procedure Wine handed
   out is in that table. It only runs when the high half is all ones.
3. `get_winproc_params()` falls back to the other A/W variant instead of passing
   the raw handle to the callback.
4. `get_winproc()` keeps returning the handle, the only value that survives a
   32-bit round trip; `get_dialog_proc()` does the same.

## Applied by

`patches/protonprep-valve-staging.sh`, section `(2-7)`:

```sh
apply_patch "../patches/wine-hotfixes/winproc-signext/0001-win32u-accept-sign-extended-winproc-handles.patch"
```

No other patch in this tree touches `dlls/win32u/class.c`, so its position in the
stack does not matter. Base: `wine` `9578fa3613f` (GE-Proton11-3). The
`get_winproc_ptr()` hunk is the first that needs adapting on a wine bump (upstream
renamed `proc_to_handle` to `MAKE_WNDPROC`).

## Verifying

```sh
grep -c 'fixup_truncated_winproc'   wine/dlls/win32u/class.c   # 5
grep -c 'still the same handle'     wine/dlls/win32u/class.c   # 1
grep -c EXCEPTION_ACCESS_VIOLATION  ~/steam-405710.log         # 0
```

Not upstreamed; the general fix belongs in Wine.
