This is an example of a crate failing to build only when `--release` is
enabled.

Build in debug mode, everything works as expected:

```
cargo +tricore b --target=tc162-htc-none
```

Build in release mode, the build fails.

```
cargo +tricore b --target=tc162-htc-none --release
```

This is the error:

```
error: __INTERRUPT_HANDLER_2 changed binding to STB_GLOBAL

error: symbol '__INTERRUPT_HANDLER_2' is already defined

error: Crt0PreInit changed binding to STB_GLOBAL

error: symbol 'Crt0PreInit' is already defined

error: Crt0PostInit changed binding to STB_GLOBAL

error: symbol 'Crt0PostInit' is already defined

error: could not compile `can` (bin "can") due to 6 previous errors
```

To fix this:

```
error: __INTERRUPT_HANDLER_2 changed binding to STB_GLOBAL

error: symbol '__INTERRUPT_HANDLER_2' is already defined
```

I neede to remove `__INTERRUPT_HANDLER_2` from the generated interrupt handlers
at the end of `main.rs`:

```diff
diff --git a/src/main.rs b/src/main.rs
index 08ffd35..4020409 100644
--- a/src/main.rs
+++ b/src/main.rs
@@ -725,7 +725,8 @@ mod runtime {
         "   .endif",
         ".endm ",
         ".pushsection .text.default_int_handler, \"ax\",@progbits",
-        "interrupt_hnd 0, 15",
+        "interrupt_hnd 0, 1",
+        "interrupt_hnd 3, 15",
         "interrupt_hnd 16, 32",
         "   ret",
         ".popsection",

```

To fix this:

```
error: Crt0PreInit changed binding to STB_GLOBAL

error: symbol 'Crt0PreInit' is already defined

error: Crt0PostInit changed binding to STB_GLOBAL

error: symbol 'Crt0PostInit' is already defined

error: could not compile `can` (bin "can") due to 6 previous errors
```

I needed to remove these two functions:

```
    // FUNCTION: Crt0PreInit
    // User hook before 'C' runtime initialization. Empty routine in case of crt0 startup code.
    global_asm!(
        ".weak Crt0PreInit",
        ".type Crt0PreInit, %function",
        "Crt0PreInit:",
        "ret",
    );

    // FUNCTION: Crt0PostInit
    // User hook after 'C' runtime initialization. Empty routine in case of crt0 startup code.
    global_asm!(
        ".weak Crt0PostInit",
        ".type Crt0PostInit, %function",
        "Crt0PostInit:",
        "ret",
    );
```

It seems that the ".weak" directive is not working when building with
`--release` flag.

The three functions `__INTERRUPT_HANDLER_2` `Crt0PostInit` and `Crt0PreInit`
are defined in the above code. I expect that the weak versions of these
functions defined in the inline assembly should be ignored when a strong
definition exists.

## Solution

The issue with the previous approach is that both implementation of `Crt0PreInit` are placed in `main.rs` (same applies to `Crt0PostInit` and `__INTERRUPT_HANDLER_2`):

```rust
#[export_name = "Crt0PreInit"]
fn pre_init_fn() {
    let cpu_core_id: u32;
    unsafe {
        core::arch::asm!("mfcr {0}, 0xFE1C", out(reg32) cpu_core_id);
    }
    if cpu_core_id == 0 {
        disable_safety_watchdog();
    }
    disable_cpu_watchdog();
}
```

```
global_asm!(
    ".weak Crt0PreInit",
    ".type Crt0PreInit, %function",
    "Crt0PreInit:",
    "ret",
);
```

During compilation this results in an asm file containing 2 different directives (`.weak` and `.global`) for the symbol `Crt0PreInit`:

```
.weak Crt0PreInit
.global Crt0PreInit
```

Which leads to the following error:

```
error: Crt0PreInit changed binding to STB_GLOBAL
```

**An easy solution is to move the runtime (which contains the inline asm and `abort` functions) into a library and link it from there.**
