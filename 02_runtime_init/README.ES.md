# Tutorial 02 - Inicialización del `runtime`

## tl;dr

* Extendimos la funcionalidad de `boot.s` para que sea capaz de llamar código Rust por primera vez. Antes de que el cambio a Rust ocurra, se realizan algunos trabajos de inicialización del `runtime` (soporte para ejecución de código).
* El código Rust que es llamado solo pausa la ejecución con una llamada a `panic!()`.
* Ejecuta `make qemu` de nuevo para que puedas ver el código adicional en acción.

## Adiciones importantes

* Adiciones importantes al script `link.ld`:
  
  * Nuevas secciones: `.rodata`, `.got`, `.data`, `.bss`.
  
  * Un lugar totalmente dedicado a enlazar argumentos de tiempo de arranque (boot-time) que necesitan estar listos cuando se llame a `_start()`.

* `_start()` en `_arch/__arch_name__/cpu/boot.s`:
  
  1. Para todos los núcleos expecto el núcleo 0.
  
  2. Inicializa la [`DRAM`](https://es.wikipedia.org/wiki/DRAM) poniendo a cero la sección de [`bss`](https://en.wikipedia.org/wiki/.bss).
  
  3. Configura el `stack pointer` (puntero a la memoria [pila](https://es.wikipedia.org/wiki/Pila_(inform%C3%A1tica))).
  
  4. Salta hacia la función `_start_rust()`, definida en `arch/__arch_name__/cpu/boot.rs`.

* `_start_rust()`:
  
  * Llama a `kernel_init()`, que llama a `panic!()`, que al final también pone al núcleo 0 en pausa.

* La librería ahora usa el crate [cortex-a](https://github.com/rust-embedded/cortex-a), que nos da abstracciones sin coste y envuelve las partes que hacen uso de un `unsafe` (partes con código que no es seguro y podría causar errores) cuando se trabaja directamente con los recursos del procesador.
  
  * Lo puedes ver en acción en `_arch/__arch_name__/cpu.rs`.

## Diferencia con el archivo anterior

```diff
diff -uNr 01_wait_forever/Cargo.toml 02_runtime_init/Cargo.toml
--- 01_wait_forever/Cargo.toml
+++ 02_runtime_init/Cargo.toml
@@ -1,6 +1,6 @@
 [package]
 name = "mingo"
-version = "0.1.0"
+version = "0.2.0"
 authors = ["Andre Richter <andre.o.richter@gmail.com>"]
 edition = "2021"

@@ -21,3 +21,7 @@
 ##--------------------------------------------------------------------------------------------------

 [dependencies]
+
+# Platform specific dependencies
+[target.'cfg(target_arch = "aarch64")'.dependencies]
+cortex-a = { version = "7.x.x" }

diff -uNr 01_wait_forever/Makefile 02_runtime_init/Makefile
--- 01_wait_forever/Makefile
+++ 02_runtime_init/Makefile
@@ -153,6 +153,8 @@
     $(call colorecho, "\nLaunching objdump")
     @$(DOCKER_TOOLS) $(OBJDUMP_BINARY) --disassemble --demangle \
                 --section .text   \
+                --section .rodata \
+                --section .got    \
                 $(KERNEL_ELF) | rustfilt

 ##------------------------------------------------------------------------------

diff -uNr 01_wait_forever/src/_arch/aarch64/cpu/boot.rs 02_runtime_init/src/_arch/aarch64/cpu/boot.rs
--- 01_wait_forever/src/_arch/aarch64/cpu/boot.rs
+++ 02_runtime_init/src/_arch/aarch64/cpu/boot.rs
@@ -13,3 +13,15 @@

 // Assembly counterpart to this file.
 core::arch::global_asm!(include_str!("boot.s"));
+
+//--------------------------------------------------------------------------------------------------
+// Public Code
+//--------------------------------------------------------------------------------------------------
+
+/// The Rust entry of the `kernel` binary.
+///
+/// The function is called from the assembly `_start` function.
+#[no_mangle]
+pub unsafe fn _start_rust() -> ! {
+    crate::kernel_init()
+}

diff -uNr 01_wait_forever/src/_arch/aarch64/cpu/boot.s 02_runtime_init/src/_arch/aarch64/cpu/boot.s
--- 01_wait_forever/src/_arch/aarch64/cpu/boot.s
+++ 02_runtime_init/src/_arch/aarch64/cpu/boot.s
@@ -3,6 +3,24 @@
 // Copyright (c) 2021-2022 Andre Richter <andre.o.richter@gmail.com>

 //--------------------------------------------------------------------------------------------------
+// Definitions
+//--------------------------------------------------------------------------------------------------
+
+// Load the address of a symbol into a register, PC-relative.
+//
+// The symbol must lie within +/- 4 GiB of the Program Counter.
+//
+// # Resources
+//
+// - https://sourceware.org/binutils/docs-2.36/as/AArch64_002dRelocations.html
+.macro ADR_REL register, symbol
+    adrp    \register, \symbol
+    add    \register, \register, #:lo12:\symbol
+.endm
+
+.equ _core_id_mask, 0b11
+
+//--------------------------------------------------------------------------------------------------
 // Public Code
 //--------------------------------------------------------------------------------------------------
 .section .text._start
@@ -11,6 +29,34 @@
 // fn _start()
 //------------------------------------------------------------------------------
 _start:
+    // Only proceed on the boot core. Park it otherwise.
+    mrs    x1, MPIDR_EL1
+    and    x1, x1, _core_id_mask
+    ldr    x2, BOOT_CORE_ID      // provided by bsp/__board_name__/cpu.rs
+    cmp    x1, x2
+    b.ne    .L_parking_loop
+
+    // If execution reaches here, it is the boot core.
+
+    // Initialize DRAM.
+    ADR_REL    x0, __bss_start
+    ADR_REL x1, __bss_end_exclusive
+
+.L_bss_init_loop:
+    cmp    x0, x1
+    b.eq    .L_prepare_rust
+    stp    xzr, xzr, [x0], #16
+    b    .L_bss_init_loop
+
+    // Prepare the jump to Rust code.
+.L_prepare_rust:
+    // Set the stack pointer.
+    ADR_REL    x0, __boot_core_stack_end_exclusive
+    mov    sp, x0
+
+    // Jump to Rust code.
+    b    _start_rust
+
     // Infinitely wait for events (aka "park the core").
 .L_parking_loop:
     wfe

diff -uNr 01_wait_forever/src/_arch/aarch64/cpu.rs 02_runtime_init/src/_arch/aarch64/cpu.rs
--- 01_wait_forever/src/_arch/aarch64/cpu.rs
+++ 02_runtime_init/src/_arch/aarch64/cpu.rs
@@ -0,0 +1,26 @@
+// SPDX-License-Identifier: MIT OR Apache-2.0
+//
+// Copyright (c) 2018-2022 Andre Richter <andre.o.richter@gmail.com>
+
+//! Architectural processor code.
+//!
+//! # Orientation
+//!
+//! Since arch modules are imported into generic modules using the path attribute, the path of this
+//! file is:
+//!
+//! crate::cpu::arch_cpu
+
+use cortex_a::asm;
+
+//--------------------------------------------------------------------------------------------------
+// Public Code
+//--------------------------------------------------------------------------------------------------
+
+/// Pause execution on the core.
+#[inline(always)]
+pub fn wait_forever() -> ! {
+    loop {
+        asm::wfe()
+    }
+}

diff -uNr 01_wait_forever/src/bsp/raspberrypi/cpu.rs 02_runtime_init/src/bsp/raspberrypi/cpu.rs
--- 01_wait_forever/src/bsp/raspberrypi/cpu.rs
+++ 02_runtime_init/src/bsp/raspberrypi/cpu.rs
@@ -0,0 +1,14 @@
+// SPDX-License-Identifier: MIT OR Apache-2.0
+//
+// Copyright (c) 2018-2022 Andre Richter <andre.o.richter@gmail.com>
+
+//! BSP Processor code.
+
+//--------------------------------------------------------------------------------------------------
+// Public Definitions
+//--------------------------------------------------------------------------------------------------
+
+/// Used by `arch` code to find the early boot core.
+#[no_mangle]
+#[link_section = ".text._start_arguments"]
+pub static BOOT_CORE_ID: u64 = 0;

diff -uNr 01_wait_forever/src/bsp/raspberrypi/link.ld 02_runtime_init/src/bsp/raspberrypi/link.ld
--- 01_wait_forever/src/bsp/raspberrypi/link.ld
+++ 02_runtime_init/src/bsp/raspberrypi/link.ld
@@ -3,6 +3,8 @@
  * Copyright (c) 2018-2022 Andre Richter <andre.o.richter@gmail.com>
  */

+__rpi_phys_dram_start_addr = 0;
+
 /* The physical address at which the the kernel binary will be loaded by the Raspberry's firmware */
 __rpi_phys_binary_load_addr = 0x80000;

@@ -13,21 +15,58 @@
  *     4 == R
  *     5 == RX
  *     6 == RW
+ *
+ * Segments are marked PT_LOAD below so that the ELF file provides virtual and physical addresses.
+ * It doesn't mean all of them need actually be loaded.
  */
 PHDRS
 {
-    segment_code PT_LOAD FLAGS(5);
+    segment_boot_core_stack PT_LOAD FLAGS(6);
+    segment_code            PT_LOAD FLAGS(5);
+    segment_data            PT_LOAD FLAGS(6);
 }

 SECTIONS
 {
-    . =  __rpi_phys_binary_load_addr;
+    . =  __rpi_phys_dram_start_addr;
+
+    /***********************************************************************************************
+    * Boot Core Stack
+    ***********************************************************************************************/
+    .boot_core_stack (NOLOAD) :
+    {
+                                             /*   ^             */
+                                             /*   | stack       */
+        . += __rpi_phys_binary_load_addr;    /*   | growth      */
+                                             /*   | direction   */
+        __boot_core_stack_end_exclusive = .; /*   |             */
+    } :segment_boot_core_stack

     /***********************************************************************************************
-    * Code
+    * Code + RO Data + Global Offset Table
     ***********************************************************************************************/
     .text :
     {
         KEEP(*(.text._start))
+        *(.text._start_arguments) /* Constants (or statics in Rust speak) read by _start(). */
+        *(.text._start_rust)      /* The Rust entry point */
+        *(.text*)                 /* Everything else */
     } :segment_code
+
+    .rodata : ALIGN(8) { *(.rodata*) } :segment_code
+    .got    : ALIGN(8) { *(.got)     } :segment_code
+
+    /***********************************************************************************************
+    * Data + BSS
+    ***********************************************************************************************/
+    .data : { *(.data*) } :segment_data
+
+    /* Section is zeroed in pairs of u64. Align start and end to 16 bytes */
+    .bss (NOLOAD) : ALIGN(16)
+    {
+        __bss_start = .;
+        *(.bss*);
+        . = ALIGN(16);
+        __bss_end_exclusive = .;
+    } :segment_data
 }

diff -uNr 01_wait_forever/src/bsp/raspberrypi.rs 02_runtime_init/src/bsp/raspberrypi.rs
--- 01_wait_forever/src/bsp/raspberrypi.rs
+++ 02_runtime_init/src/bsp/raspberrypi.rs
@@ -4,4 +4,4 @@

 //! Top-level BSP file for the Raspberry Pi 3 and 4.

-// Coming soon.
+pub mod cpu;

diff -uNr 01_wait_forever/src/cpu.rs 02_runtime_init/src/cpu.rs
--- 01_wait_forever/src/cpu.rs
+++ 02_runtime_init/src/cpu.rs
@@ -4,4 +4,13 @@

 //! Processor code.

+#[cfg(target_arch = "aarch64")]
+#[path = "_arch/aarch64/cpu.rs"]
+mod arch_cpu;
+
 mod boot;
+
+//--------------------------------------------------------------------------------------------------
+// Architectural Public Reexports
+//--------------------------------------------------------------------------------------------------
+pub use arch_cpu::wait_forever;

diff -uNr 01_wait_forever/src/main.rs 02_runtime_init/src/main.rs
--- 01_wait_forever/src/main.rs
+++ 02_runtime_init/src/main.rs
@@ -102,6 +102,7 @@
 //!
 //! 1. The kernel's entry point is the function `cpu::boot::arch_boot::_start()`.
 //!     - It is implemented in `src/_arch/__arch_name__/cpu/boot.s`.
+//! 2. Once finished with architectural setup, the arch code calls `kernel_init()`.

 #![no_main]
 #![no_std]
@@ -110,4 +111,11 @@
 mod cpu;
 mod panic_wait;

-// Kernel code coming next tutorial.
+/// Early init code.
+///
+/// # Safety
+///
+/// - Only a single core must be active and running this function.
+unsafe fn kernel_init() -> ! {
+    panic!()
+}

diff -uNr 01_wait_forever/src/panic_wait.rs 02_runtime_init/src/panic_wait.rs
--- 01_wait_forever/src/panic_wait.rs
+++ 02_runtime_init/src/panic_wait.rs
@@ -4,9 +4,10 @@

 //! A panic handler that infinitely waits.

+use crate::cpu;
 use core::panic::PanicInfo;

 #[panic_handler]
 fn panic(_info: &PanicInfo) -> ! {
-    unimplemented!()
+    cpu::wait_forever()
 }
```
