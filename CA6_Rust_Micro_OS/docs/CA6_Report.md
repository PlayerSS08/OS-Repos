# 1. Introduction

**Aeon** is a small operating system written in Rust (`no_std`, no external dependencies) for 64-bit UEFI PCs. It boots from a FAT32 USB stick, shows a logo with a progress bar, leaves the firmware, sets up its own CPU and memory protection, and runs a preemptive multitasking kernel with a shell. The whole image is about 572 KB (the tests enforce a 640 KiB limit) and boots in 64 MB of RAM.

This report describes what Aeon does, how it is built and tested, and what is still missing. Everything below is taken from the project's README.

| | |
|---|---|
| **Language** | Rust (`no_std`), target `x86_64-unknown-uefi` |
| **Version** | 0.11.0 (processes), with Wi-Fi through the host added on 2026-11-19 |
| **Platform** | UEFI x86_64, tested in QEMU |
| **Size** | about 572 KB image, 64 MB RAM is enough |

# 2. Goals

* Boot a real machine from a USB stick with a clear, honest boot screen (logo, progress tied to real boot stages).
* Take control from the firmware and protect memory from the start (W^X, guard pages, SMEP/SMAP when available).
* Provide the basic OS pieces: scheduler, memory manager, processes, system calls, files, a shell.
* Make boot measurable and checkable: measured boot into the TPM, recovery mode, boot log.
* Keep the code small, in safe-by-default Rust, and test it automatically.

# 3. How Aeon Boots

| Stage | What happens |
|---|---|
| firmware-entry | stops the firmware watchdog, calibrates the timer, starts logging |
| early-init | reads CPU features, sets up the screen (framebuffer), fades in the logo |
| platform-discovery | reads firmware info, ACPI, SMBIOS, Secure Boot state, TPM presence |
| security-init | creates a random seed, measures the command line into TPM PCR 12 and reads it back to check |
| exit-boot-services | marks the boot phase in PCR 11, then leaves the firmware |
| boot menu (Dawn) | seven entries (Boot Aeon, Recovery mode, Firmware shell, Debug logging, Firmware setup, Restart, Power off), 15-second countdown, arrow keys and Enter |
| kernel-env | own memory map, page tables with W^X, GDT/TSS, IDT, protections, own stack |
| kernel-memory, kernel-tasks | frame allocator, 32 MB heap, timer tick, scheduler, `status` and `shell` tasks |

Boot options (`quiet`, `debug`, `verbose`, `nomenu`, `timeout=N`, `mode=WxH`, `recovery`, `nocount`, `nolapic`) can come from the command line or from `EFI/BOOT/aeon.cfg` on the stick.

# 4. The Kernel

* **Memory:** a frame bitmap allocator, a free-list heap (`GlobalAlloc`), identity-mapped page tables with W^X, an unmapped null page and guard pages under every stack. Memory freed by the firmware is zeroed before reuse.
* **Scheduling:** preemptive round-robin tasks driven by the local APIC timer (or the PIT as a fallback). The `spawn` and `counters` commands show two busy workers being preempted by the timer.
* **Processes (0.11.0):** every program has its own page tables; `fork`, `execve`, `wait4` and `pipe` exist; `/bin` holds small Rust programs (`hello echo cat wc ls forktest`).
* **System calls and ring 3:** user programs run in ring 3; system calls validate every pointer and a bad call or fault ends only that program.
* **Files:** a RAM filesystem with folders (up to 64 entries, 1 MiB), plus read-only FAT16/FAT32 access to a virtio disk.
* **Shell:** a line editor with Tab completion, pipes, `>`, `>>`, `&` and Ctrl+C; commands include `ls cat write edit rm df date log help`.
* **Networking and devices:** `dhcp`, `curl <url>`, `sha256sum`, a signed package manager (`apt`), USB mouse, USB tethering and a touchpad driver (the last ones untested on real hardware). Wi-Fi works through the Windows host in QEMU: the host joins the network and Aeon gets an address; Aeon has no Wi-Fi chip driver.

# 5. Security Features

* **Secure Boot ready:** the release image is signed and its certificate ships on the USB stick.
* **Measured boot:** the command line is hashed into TPM PCR 12 and read back; the TPM event log is replayed and compared with the PCRs before the firmware is left.
* **Recovery mode:** a boot counter in firmware variables; after three failed boots in a row the next boot starts a minimal recovery mode.
* **Last-panic record:** a short text (no addresses) is kept to the next boot and shown once.
* **Memory protection:** W^X, NX, SMEP/SMAP/UMIP when the CPU has them, guard pages, stack canary seeded from the boot random seed, scrubbed freed heap blocks.
* **Safe parsing:** firmware tables are bounds-checked and validated; the parsers get random and mutated input in the unit tests.
* **Honest limits:** CET is detected but not enabled (it cannot be tested in QEMU); no attestation is claimed.

# 6. Comparison With Other Boot Systems

| Idea | Linux | Windows | Aeon |
|---|---|---|---|
| Signed chain | shim, signed loader and kernel | `bootmgfw`, `winload` | one signed image |
| Measured boot | PCRs 4, 7, 9, 11, 12 | PCR 7 and TCG log (BitLocker) | command line into PCR 12, verified by read-back |
| Boot menu | GRUB, systemd-boot | Windows Boot Manager | Dawn menu |
| Automatic fallback | boot counting in systemd-boot | recovery environment | recovery mode after 3 failures (no second image to fall back to) |
| Memory protection | kernel W^X, guard pages | HVCI, kernel DEP | W^X, guard pages, SMEP/SMAP |

# 7. Build and Test

```text
rustup target add x86_64-unknown-uefi
cargo build --release
cargo clippy --release -- -D warnings
cargo test --lib --target x86_64-unknown-linux-gnu
bash tools/release.sh        # build, sign, refresh usb/
bash tests/run.sh            # lint, unit tests, QEMU boot tests
```

The QEMU boot tests start the real image (with OVMF firmware and a software TPM) and check: all boot stages, the splash and menu, the shell and preemption, measured boot (PCR 12 against an independent calculation), Secure Boot (signed image boots, unsigned or unknown-key images are refused), `aeon.cfg`, screen modes, recovery mode, boot counting, the last-panic record, event-log replay, the image size, and that every `fault-*` test build ends in the expected stop screen.

To try it on Windows: `powershell -ExecutionPolicy Bypass -File tools\run-qemu.ps1 -Build -Disk`. To boot on a PC, copy the `usb` folder to a FAT32 stick and boot in UEFI mode (Secure Boot off, or enroll `aeon-secureboot.cer`).

# 8. Testing Status and Limitations

**Tested:** QEMU with EDK II OVMF, a software TPM, Secure Boot with test and release keys, host unit tests. Version 0.11.0 was built and unit-tested on the host and was then booted in QEMU, where it reached the shell.

**Not tested:** real hardware, USB-only keyboards, VirtualBox/Hyper-V/VMware, a hardware TPM, enrolling the certificate through a real firmware screen, 4K screens, and the USB tethering and touchpad features.

**Limits:** one CPU, identity mapping for the kernel, RAM-only files (disks are read-only), no Wi-Fi chip driver, no kernel signature check beyond the firmware's.

# 9. Roadmap

Loading programs from files and per-program address spaces (started in 0.11.0), a USB keyboard driver, AHCI and NVMe disks, device interrupts through the IOAPIC, SMP, and an `aarch64` port.

# 10. Conclusion

Aeon shows that a complete small OS can be built in Rust with no dependencies: it boots from UEFI, protects its own memory, schedules tasks preemptively, runs separate processes with system calls, gives the user a shell, and records measured boot evidence in the TPM. Its strength is that every claim is backed by an automated test; its limits (one CPU, no real-hardware testing yet) are stated openly.

# 11. AI Disclosure

Generative AI (Claude) was used to turn the project's README into this report. All statements come from the README; nothing was measured or run for this report.

# References

1. Aeon `README.md` and `docs/` (release notes, design notes, OS landscape).
2. UEFI Specification; TCG PC Client Platform specifications (measured boot).
3. A. Silberschatz, P. B. Galvin, G. Gagne, *Operating System Concepts*, 10th ed., Wiley, 2018.
