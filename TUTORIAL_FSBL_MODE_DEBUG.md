# TUTORIAL FSBL EBAZ4205 — MODE 1: DEBUG (VERBOSE SERIAL UART)

Dokumen ini adalah panduan teknis langkah-demi-langkah untuk membuat dan mengompilasi **FSBL (First Stage Boot Loader) Mode Debug** untuk board **EBAZ4205 (Zynq-7010)**.

> [!NOTE]
> **Karakteristik Mode Debug:**
> * **Tujuan:** Analisis mendalam, verifikasi pembacaan MicroSD, dan diagnosis error code.
> * **Output UART:** **AKTIF** (Menampilkan teks log proses booting di Serial Monitor 115200 baud).
> * **Kapan Dipakai:** Saat pertama kali mencoba board, mengganti MicroSD baru, atau saat lampu LED DONE tidak menyala.

---

## DAFTAR ISI
1. [Kenapa EBAZ4205 Butuh Patch Khusus?](#1-kenapa-ebaz4205-butuh-patch-khusus)
2. [Langkah 1: Setup Vivado Block Design](#langkah-1-setup-vivado-block-design)
3. [Langkah 2: Buat Project FSBL Baru di Xilinx SDK](#langkah-2-buat-project-fsbl-baru-di-xilinx-sdk)
4. [Langkah 3: Pembersihan Dependensi Build (Fix Error Multiple Target Patterns)](#langkah-3-pembersihan-dependensi-build-fix-error-multiple-target-patterns)
5. [Langkah 4: Modifikasi Driver MicroSD 1-Bit + Diagnostic Log (`xsdps.c`)](#langkah-4-modifikasi-driver-microsd-1-bit--diagnostic-log-xsdpsc)
6. [Langkah 5: Fix MultiBoot Offset Bug (`image_mover.c`)](#langkah-5-fix-multiboot-offset-bug-image_moverc)
7. [Langkah 6: Aktifkan Debug Console UART (`fsbl_debug.h`)](#langkah-6-aktifkan-debug-console-uart-fsbl_debugh)
8. [Langkah 7: Setup Chainloader NAND Flash dengan Log Verbose (`main.c`)](#langkah-7-setup-chainloader-nand-flash-dengan-log-verbose-mainc)
9. [Langkah 8: Build ELF & Generate `BOOT.bin`](#langkah-8-build-elf--generate-bootbin)
10. [Langkah 9: Membaca Output Serial Monitor 115200](#langkah-9-membaca-output-serial-monitor-115200)

---

## 1. Kenapa EBAZ4205 Butuh Patch Khusus?
1. **Slot U7 Jalur Data Kosong (NC):** Resistor pull-up `R161–R165` tidak dipasang oleh pabrik. Driver bawaan Xilinx yang memaksa 4-bit dan perintah `CMD6` pasti akan *freeze/hang* selamanya. Solusinya adalah memaksanya ke **Mode 1-Bit** (`DAT0` saja).
2. **Kapasitor Daya MicroSD Kosong (`C93 & C94 NC`):** Kartu MicroSD mengalami lonjakan arus saat cold-boot dan kalah cepat siap dibanding MCU. Solusinya adalah memberi jeda **delay 1.5 detik** sebelum membaca kartu.
3. **MultiBoot Offset Bug:** Nilai sisa register BootROM menggeser pembacaan partisi ke offset `0x8000`. Solusinya adalah mengunci `ImageStartAddress = 0`.

---

## Langkah 1: Setup Vivado Block Design
1. Buka Block Design di Vivado, klik ganda blok IP **`ZYNQ7 Processing System`**.
2. Masuk ke **MIO Configuration**:
   * **NAND Flash:** Centang Enable, pilih pin `MIO 0 2.. 14`.
   * **SD 0:** Centang Enable, pilih pin `MIO 40 .. 45`. **Uncheck Card Detect (CD)** dan **Write Protect (WP)**.
   * **UART 1:** Centang Enable (arahkan ke EMIO atau MIO pin serial Anda) dengan baudrate **115200**.
3. Masuk ke **DDR Configuration**:
   * Memory Part: Pilih **`MT41K128M16 JT-125`**, Bus Width: **`16 Bit`**.
4. Generate Bitstream $\rightarrow$ **File $\rightarrow$ Export $\rightarrow$ Export Hardware** (centang *Include bitstream*) $\rightarrow$ **Launch SDK**.

---

## Langkah 2: Buat Project FSBL Baru di Xilinx SDK
1. Klik **File $\rightarrow$ New $\rightarrow$ Application Project**.
2. Beri nama: `fsbl_debug`.
3. Pilih Hardware Platform Anda, Target OS: `standalone`.
4. Klik **Next**, pilih template **Zynq FSBL**, lalu klik **Finish**.

---

## Langkah 3: Pembersihan Dependensi Build (Fix Error Multiple Target Patterns)
Jika project disalin dari Windows ke Linux, jalankan pembersihan file `.d` agar make tidak error:
```bash
cd <path_workspace>/fsbl_debug/Debug
rm -f src/*.d src/*.o fsbl_debug.elf
```

---

## Langkah 4: Modifikasi Driver MicroSD 1-Bit + Diagnostic Log (`xsdps.c`)
1. Salin file `xsdps.c` dari folder BSP (`fsbl_debug_bsp/ps7_cortexa9_0/libsrc/sdps_vX_X/src/xsdps.c`) ke dalam folder project **`fsbl_debug/src/`**.
2. Buka `fsbl_debug/src/xsdps.c`:
   * Pada baris ~115, tambahkan:
     ```c
     #ifndef MAX_TIMEOUT
     #define MAX_TIMEOUT        10000000U
     #endif
     ```
   * Pada fungsi **`XSdPs_CardInitialize()`** (baris ~675):
     Ganti blok pengaturan bus width dan `CMD6` menjadi:
     ```c
     /* ================= PATCH 1-BIT MODE EBAZ4205 (DEBUG) ================= */
     xil_printf("\r\n[SD DEBUG] Mengunci Bus Width ke 1-Bit Mode (DAT0)...\r\n");
     InstancePtr->BusWidth = XSDPS_1_BIT_WIDTH;
     
     /* Matikan XSdPs_Change_BusWidth & CMD6 yang menyebabkan freeze */
     /*
     if ((SCR[1] & WIDTH_4_BIT_SUPPORT) != 0U) {
         Status = XSdPs_Change_BusWidth(InstancePtr);
         if (Status != XST_SUCCESS) { goto RETURN_PATH; }
     }
     Status = XSdPs_Get_BusSpeed(InstancePtr, ReadBuff);
     if (Status != XST_SUCCESS) { goto RETURN_PATH; }
     */
     xil_printf("[SD DEBUG] MicroSD 1-Bit Siap Digunakan!\r\n");
     ```

---

## Langkah 5: Fix MultiBoot Offset Bug (`image_mover.c`)
Buka file **`fsbl_debug/src/image_mover.c`**, cari fungsi **`LoadBootImage()`** (baris ~220):
```c
        ImageStartAddress = (MultiBootReg & PCAP_MBOOT_REG_REBOOT_OFFSET_MASK)
                                    * GOLDEN_IMAGE_OFFSET;
    }

    /* TAMBAHKAN BLOK KODE INI: */
    extern u32 SDAccess(u32 SourceAddress, u32 DestinationAddress, u32 LengthBytes);
    if (MoveImage == SDAccess) {
        ImageStartAddress = 0;
        xil_printf("[SD DEBUG] MultiBoot Offset di-reset ke 0x00000000\r\n");
    }

    fsbl_printf(DEBUG_INFO,"Image Start Address: 0x%08lx\r\n",ImageStartAddress);
```

---

## Langkah 6: Aktifkan Debug Console UART (`fsbl_debug.h`)
Buka file **`fsbl_debug/src/fsbl_debug.h`**:
Tambahkan define `FSBL_DEBUG_INFO` pada baris ~55:

```c
#define DEBUG_GENERAL   0x00000001    /* general debug messages */
#define DEBUG_INFO      0x00000002    /* More debug information */

/* AKTIFKAN DEBUG PRINT UNTUK MODE DEBUG: */
#ifndef FSBL_DEBUG_INFO
#define FSBL_DEBUG_INFO
#endif

#if defined (FSBL_DEBUG_INFO)
#define fsbl_dbg_current_types ((DEBUG_INFO) | (DEBUG_GENERAL))
#endif
```

---

## Langkah 7: Setup Chainloader NAND Flash dengan Log Verbose (`main.c`)
Buka file **`fsbl_debug/src/main.c`**, cari bagian `NAND BOOT MODE` (baris ~410) dan ubah menjadi:

```c
#ifdef XPAR_PS7_NAND_0_BASEADDR
    if (BootModeRegister == NAND_FLASH_MODE) {
        fsbl_printf(DEBUG_GENERAL, "\r\n============================================\r\n");
        fsbl_printf(DEBUG_GENERAL, "   EBAZ4205 CHAINLOADER (MODE DEBUG UART)   \r\n");
        fsbl_printf(DEBUG_GENERAL, " Menunggu 1.5 detik agar daya SD stabil...  \r\n");
        fsbl_printf(DEBUG_GENERAL, "============================================\r\n");
        
        usleep(1500000); /* Delay 1.5 detik */

#if defined(XPAR_PS7_SD_0_S_AXI_BASEADDR) || defined(XPAR_XSDPS_0_BASEADDR)
        fsbl_printf(DEBUG_GENERAL, "[CHAINLOADER] Mencoba membaca MicroSD...\r\n");
        Status = InitSD("BOOT.BIN");
        if (Status != XST_SUCCESS) {
            fsbl_printf(DEBUG_GENERAL, "[WARNING] MicroSD gagal dibuka! Fallback ke NAND...\r\n");
            Status = InitNand();
            if (Status != XST_SUCCESS) {
                OutputStatus(NAND_INIT_FAIL);
                FsblFallback();
            }
            MoveImage = NandAccess;
        } else {
            fsbl_printf(DEBUG_GENERAL, "[SUCCESS] File BOOT.BIN ditemukan di MicroSD!\r\n");
            MoveImage = SDAccess;
            BootModeRegister = SD_MODE;
        }
#endif
    } else
#endif
```

---

## Langkah 8: Build ELF & Generate `BOOT.bin`
1. Klik kanan project `fsbl_debug` $\rightarrow$ **Clean Project** $\rightarrow$ **Build Project**.
2. Membuat Boot Image via GUI SDK:
   * Klik kanan project `fsbl_debug` $\rightarrow$ **Create Boot Image**.
   * Tambahkan 3 partisi berurutan:
     1. `fsbl_debug.elf` (Type: `bootloader`)
     2. `bitstream.bit` (Type: `datafile`)
     3. `aplikasi.elf` (Type: `datafile`)
   * Klik **Create Image** $\rightarrow$ Simpan sebagai `BOOT.BIN` di kartu MicroSD.

---

## Langkah 9: Membaca Output Serial Monitor 115200
Buka Serial Monitor (Arduino IDE / Putty) dengan setting:
* **Port:** `/dev/ttyUSB0` (atau COM port Windows Anda)
* **Baudrate:** `115200`
* **Data bits:** 8, **Parity:** None, **Stop bits:** 1

**Tampilan Normal yang Akan Muncul:**
```text
Xilinx First Stage Boot Loader 
Release 2019.1

============================================
   EBAZ4205 CHAINLOADER (MODE DEBUG UART)   
 Menunggu 1.5 detik agar daya SD stabil...  
============================================
[CHAINLOADER] Mencoba membaca MicroSD...
[SD DEBUG] Mengunci Bus Width ke 1-Bit Mode (DAT0)...
[SD DEBUG] MicroSD 1-Bit Siap Digunakan!
[SUCCESS] File BOOT.BIN ditemukan di MicroSD!
[SD DEBUG] MultiBoot Offset di-reset ke 0x00000000
Loading Partition #1: Bitstream
Loading Partition #2: Application
Handoff Address: 0x00100000
```
Jika log di atas muncul, lampu **LED Hijau DONE** akan menyala terang dan aplikasi Anda berjalan!
