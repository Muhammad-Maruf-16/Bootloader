# TUTORIAL FSBL EBAZ4205 — MODE 2: "LANGSUNG GASS" (FAST BOOT & HEMAT MEMORI)

Dokumen ini adalah panduan teknis langkah-demi-langkah untuk membuat dan mengompilasi **FSBL (First Stage Boot Loader) Mode "Langsung Gass" (Production / Silent Fast Boot)** untuk board **EBAZ4205 (Zynq-7010)**.

> [!TIP]
> **Karakteristik Mode "Langsung Gass":**
> * **Tujuan:** Produksi / penggunaan harian saat sistem sudah dipastikan normal.
> * **Keunggulan:** **Menghemat memori RAM internal (OCM) hingga ~50%** dan proses booting berjalan secepat kilat tanpa terhambat latensi pengiriman karakter serial UART.
> * **Output UART:** **NONAKTIF (Silent)**. Tidak ada string teks debugging yang di-load ke memori.

---

## DAFTAR ISI
1. [Mengapa Mode "Langsung Gass" Sangat Menguntungkan?](#1-mengapa-mode-langsung-gass-sangat-menguntungkan)
2. [Langkah 1: Setup Vivado Block Design](#langkah-1-setup-vivado-block-design)
3. [Langkah 2: Buat Project FSBL Baru di Xilinx SDK](#langkah-2-buat-project-fsbl-baru-di-xilinx-sdk)
4. [Langkah 3: Pembersihan Dependensi Build (Fix Error Multiple Target Patterns)](#langkah-3-pembersihan-dependensi-build-fix-error-multiple-target-patterns)
5. [Langkah 4: Modifikasi Driver MicroSD Clean 1-Bit (`xsdps.c`)](#langkah-4-modifikasi-driver-microsd-clean-1-bit-xsdpsc)
6. [Langkah 5: Fix MultiBoot Offset Bug (`image_mover.c`)](#langkah-5-fix-multiboot-offset-bug-image_moverc)
7. [Langkah 6: Matikan Debug UART di `fsbl_debug.h`](#langkah-6-matikan-debug-uart-di-fsbl_debugh)
8. [Langkah 7: Trik Ekstra Optimasi Ukuran GCC (`-Os`)](#langkah-7-trik-ekstra-optimasi-ukuran-gcc--os)
9. [Langkah 8: Setup Chainloader NAND Flash Ringkas (`main.c`)](#langkah-8-setup-chainloader-nand-flash-ringkas-mainc)
10. [Langkah 9: Build ELF & Generate `BOOT.bin`](#langkah-9-build-elf--generate-bootbin)
11. [Perbandingan Konsumsi Memori OCM (Debug vs Langsung Gass)](#perbandingan-konsumsi-memori-ocm-debug-vs-langsung-gass)

---

## 1. Mengapa Mode "Langsung Gass" Sangat Menguntungkan?
Zynq-7010 hanya memiliki **256 KB On-Chip Memory (OCM)** internal:
* Jika Mode Debug aktif: ratusan string teks `xil_printf` memakan ruang memori hingga **~168 KB**, menyisakan sedikit ruang OCM untuk aplikasi bare-metal Anda. Selain itu, kecepatan boot melambat karena CPU harus menunggu UART mentransmisikan karakter satu per satu pada baudrate 115200.
* Pada **Mode Langsung Gass**: semua string teks dibuang dan compiler dioptimasi untuk ukuran (`-Os`). Ukuran biner FSBL menyusut drastis menjadi hanya **~65 KB s/d 80 KB**, dan Zynq langsung menembakkan bitstream FPGA dalam hitungan pecahan detik!

---

## Langkah 1: Setup Vivado Block Design
1. Buka Block Design di Vivado, klik ganda blok IP **`ZYNQ7 Processing System`**.
2. Masuk ke **MIO Configuration**:
   * **NAND Flash:** Centang Enable, pilih pin `MIO 0 2.. 14`.
   * **SD 0:** Centang Enable, pilih pin `MIO 40 .. 45`. **Uncheck Card Detect (CD)** dan **Write Protect (WP)**.
   * **UART 1:** Boleh dimatikan atau dibiarkan aktif (tidak akan digunakan oleh FSBL, tapi tetap bisa digunakan oleh aplikasi user Anda nantinya).
3. Masuk ke **DDR Configuration**:
   * Memory Part: Pilih **`MT41K128M16 JT-125`**, Bus Width: **`16 Bit`**.
4. Generate Bitstream $\rightarrow$ **File $\rightarrow$ Export $\rightarrow$ Export Hardware** (centang *Include bitstream*) $\rightarrow$ **Launch SDK**.

---

## Langkah 2: Buat Project FSBL Baru di Xilinx SDK
1. Klik **File $\rightarrow$ New $\rightarrow$ Application Project**.
2. Beri nama: `fsbl_fast`.
3. Pilih Hardware Platform Anda, Target OS: `standalone`.
4. Klik **Next**, pilih template **Zynq FSBL**, lalu klik **Finish**.

---

## Langkah 3: Pembersihan Dependensi Build (Fix Error Multiple Target Patterns)
Jika project disalin dari Windows ke Linux, jalankan pembersihan file `.d` agar make tidak error:
```bash
cd <path_workspace>/fsbl_fast/Debug
rm -f src/*.d src/*.o fsbl_fast.elf
```

---

## Langkah 4: Modifikasi Driver MicroSD Clean 1-Bit (`xsdps.c`)
1. Salin file `xsdps.c` dari folder BSP (`fsbl_fast_bsp/ps7_cortexa9_0/libsrc/sdps_vX_X/src/xsdps.c`) ke dalam folder project **`fsbl_fast/src/`**.
2. Buka `fsbl_fast/src/xsdps.c`:
   * Pada baris ~115, tambahkan:
     ```c
     #ifndef MAX_TIMEOUT
     #define MAX_TIMEOUT        10000000U
     #endif
     ```
   * Pada fungsi **`XSdPs_CardInitialize()`** (baris ~675):
     Ganti blok pengaturan bus width dan `CMD6` menjadi versi bersih tanpa overhead print:
     ```c
     /* ================= PATCH 1-BIT MODE EBAZ4205 (FAST / SILENT) ================= */
     /* 
      * Kunci langsung ke 1-Bit Mode (DAT0) dan lewati CMD6.
      * Tidak ada teks printout UART untuk menghemat memori OCM.
      */
     InstancePtr->BusWidth = XSDPS_1_BIT_WIDTH;
     
     /* Matikan panggilan XSdPs_Change_BusWidth & CMD6 yang menyebabkan freeze */
     /*
     if ((SCR[1] & WIDTH_4_BIT_SUPPORT) != 0U) {
         Status = XSdPs_Change_BusWidth(InstancePtr);
         if (Status != XST_SUCCESS) { goto RETURN_PATH; }
     }
     Status = XSdPs_Get_BusSpeed(InstancePtr, ReadBuff);
     if (Status != XST_SUCCESS) { goto RETURN_PATH; }
     */
     ```

---

## Langkah 5: Fix MultiBoot Offset Bug (`image_mover.c`)
Buka file **`fsbl_fast/src/image_mover.c`**, cari fungsi **`LoadBootImage()`** (baris ~220):
```c
        ImageStartAddress = (MultiBootReg & PCAP_MBOOT_REG_REBOOT_OFFSET_MASK)
                                    * GOLDEN_IMAGE_OFFSET;
    }

    /* TAMBAHKAN KODE INI TEPAT DI BAWAHNYA: */
    extern u32 SDAccess(u32 SourceAddress, u32 DestinationAddress, u32 LengthBytes);
    if (MoveImage == SDAccess) {
        ImageStartAddress = 0;
    }

    fsbl_printf(DEBUG_INFO,"Image Start Address: 0x%08lx\r\n",ImageStartAddress);
```

---

## Langkah 6: Matikan Debug UART di `fsbl_debug.h`
Buka file **`fsbl_fast/src/fsbl_debug.h`**:
Pastikan `FSBL_DEBUG_INFO` **di-comment / dinonaktifkan**, dan arahkan `fsbl_dbg_current_types` ke `0`:

```c
#define DEBUG_GENERAL   0x00000001    /* general debug messages */
#define DEBUG_INFO      0x00000002    /* More debug information */

/* ================= MODE LANGSUNG GASS (SILENT) ================= */
/* Pastikan baris define di-comment (dimatikan): */
/* #define FSBL_DEBUG_INFO */

#if defined (FSBL_DEBUG_INFO)
#define fsbl_dbg_current_types ((DEBUG_INFO) | (DEBUG_GENERAL))
#else
/* Semua pesan debug dimatikan total agar string dibuang oleh linker */
#define fsbl_dbg_current_types 0
#endif
```

---

## Langkah 7: Trik Ekstra Optimasi Ukuran GCC (`-Os`)
Untuk memangkas ukuran biner FSBL hingga titik terkecil:
1. Di Xilinx SDK, klik kanan pada project **`fsbl_fast` $\rightarrow$ Properties**.
2. Masuk ke **C/C++ Build $\rightarrow$ Settings**.
3. Di tab *Tool Settings*, buka **ARM v7 gcc compiler $\rightarrow$ Optimization**.
4. Ubah **Optimization Level** menjadi:
   👉 **`Optimize for size (-Os)`**
5. Klik **Apply and Close**.

---

## Langkah 8: Setup Chainloader NAND Flash Ringkas (`main.c`)
Buka file **`fsbl_fast/src/main.c`**, cari bagian `NAND BOOT MODE` (baris ~410) dan ubah menjadi logika ringkas tanpa print:

```c
#ifdef XPAR_PS7_NAND_0_BASEADDR
    if (BootModeRegister == NAND_FLASH_MODE) {
        /* Beri jeda 1.5 detik agar daya kartu MicroSD stabil */
        usleep(1500000);

#if defined(XPAR_PS7_SD_0_S_AXI_BASEADDR) || defined(XPAR_XSDPS_0_BASEADDR)
        Status = InitSD("BOOT.BIN");
        if (Status != XST_SUCCESS) {
            /* Jika SD belum siap, fallback kembali ke NAND */
            Status = InitNand();
            if (Status != XST_SUCCESS) {
                OutputStatus(NAND_INIT_FAIL);
                FsblFallback();
            }
            MoveImage = NandAccess;
        } else {
            /* MicroSD siap! Langsung gass load partisi dari kartu SD */
            MoveImage = SDAccess;
            BootModeRegister = SD_MODE;
        }
#endif
    } else
#endif
```

---

## Langkah 9: Build ELF & Pembuatan Boot Image (PENTING: Ada 2 File Berbeda!)

1. Klik kanan project `fsbl_fast` $\rightarrow$ **Clean Project** $\rightarrow$ **Build Project**.
2. Amati tab *Console*:
   ```text
      text    data     bss     dec     hex filename
     58420    2110   12192   72722   11c12 fsbl_fast.elf
   ```
   Perhatikan bagian `text` dan `data` yang menyusut drastis dibanding Mode Debug!

Dalam konsep **Chainloader**, ada **2 file biner berbeda** yang harus Anda buat:

---

### Bagian A: Membuat File Biner untuk NAND FLASH (`BOOT_NAND.bin`)
File ini yang akan **dibakar ke chip NAND Flash board** menggunakan menu *Program Flash* di Xilinx SDK.

1. Klik kanan project `fsbl_fast` $\rightarrow$ **Create Boot Image**.
2. Tentukan nama output: `BOOT_NAND.bin`.
3. Pada tabel partisi, **HANYA MASUKKAN 1 FILE SAJA**:
   * File: `fsbl_fast.elf` $\rightarrow$ Type: **`bootloader`**
   *(PERHATIAN: JANGAN masukkan bitstream atau aplikasi di sini! NAND hanya butuh FSBL sebagai kunci pembuka).*
4. Klik **Create Image**. File `BOOT_NAND.bin` siap dibakar.

---

### Bagian B: Membuat File `BOOT.BIN` untuk KARTU MICROSD
File ini yang akan **dicopy-paste ke dalam kartu MicroSD** (format FAT32) dan dicolokkan ke slot TF U7 EBAZ4205.

1. Klik kanan project `fsbl_fast` $\rightarrow$ **Create Boot Image**.
2. Tentukan nama output: `BOOT.BIN`.
3. Masukkan **3 partisi lengkap berurutan**:
   1. `fsbl_fast.elf` $\rightarrow$ Type: **`bootloader`** *(Gunakan file FSBL yang sama persis)*
   2. `bitstream.bit` $\rightarrow$ Type: **`datafile`** *(File konfigurasi gerbang FPGA dari Vivado)*
   3. `aplikasi.elf` $\rightarrow$ Type: **`datafile`** *(Program C ARM Anda, misal: hello_world.elf atau test_led.elf)*
4. Klik **Create Image**. File `BOOT.BIN` siap dicopy ke kartu SD.

---

## Langkah 10: Cara Membakar (Flashing) ke Board EBAZ4205

### 1. Cara Flashing ke NAND Flash (Chip Internal Board)
Pastikan kabel JTAG sudah terhubung ke PC dan board sudah menyala.

**A. Lewat GUI Xilinx SDK (Paling Mudah):**
1. Di menu atas SDK, klik **Xilinx $\rightarrow$ Program Flash** (atau klik kanan project $\rightarrow$ *Program Flash Memory*).
2. Isi kolom konfigurasi berikut:
   * **Hardware Platform:** Pilih hardware platform Anda (`design_1_wrapper_hw_platform_0`).
   * **Connection:** `Local` (atau JTAG cable yang terdeteksi otomatis).
   * **Image File:** Browse $\rightarrow$ Pilih file **`BOOT_NAND.bin`** yang tadi dibuat.
   * **FSBL File:** Browse $\rightarrow$ Pilih file **`fsbl_fast.elf`** Anda. *(Wajib diisi karena Xilinx butuh FSBL pembantu untuk mengaktifkan controller RAM DDR3 & NAND saat proses flashing)*.
   * **Offset:** Ketik **`0x0`** (wajib mulai dari alamat nol).
   * **Flash Type:** Pilih **`nand-x8-single`** (atau `NAND 128MB`).
   * Centang opsi: **Verify after flash**.
3. Klik tombol **Program**. Tunggu sampai muncul status: `Flash Operation Successful`.

**B. Lewat Terminal Linux (Opsional):**
```bash
source /tools/Xilinx/SDK/2019.1/settings64.sh
program_flash -f /path/ke/BOOT_NAND.bin \
              -fsbl /path/ke/fsbl_fast.elf \
              -flash_type nand-x8-single \
              -blank_check -verify \
              -cable type xilinx_tcf
```

---

### 2. Cara "Flashing" / Memasang Aplikasi Utama Anda

Ada 3 cara untuk menjalankan / membakar aplikasi Anda:

* **Metode 1: Lewat MicroSD (Metode Chainloader — Sangat Praktis):**
  * Tidak perlu kabel JTAG lagi! Cukup colokkan kartu MicroSD ke laptop dengan Card Reader.
  * Copy-paste file **`BOOT.BIN`** (yang berisi FSBL + Bitstream + Aplikasi) ke dalam MicroSD.
  * Masukkan kartu ke slot TF U7 EBAZ4205, lalu nyalakan board. Selesai!
  * *Keunggulan:* Setiap kali Anda mengubah kodingan aplikasi, Anda tidak perlu men-flash ulang chip board, cukup ganti file `BOOT.BIN` di kartu SD.

* **Metode 2: Full Standalone NAND Flash (Jika Ingin Berjalan Tanpa Kartu SD):**
  * Jika suatu saat Anda ingin board menyala mandiri tanpa kartu SD:
  * Buat Boot Image baru dengan 3 partisi lengkap:
    1. `fsbl_fast.elf` (Type: `bootloader`)
    2. `bitstream.bit` (Type: `datafile`)
    3. `aplikasi.elf` (Type: `datafile`)
  * Simpan sebagai `BOOT_NAND_FULL.bin`.
  * Lalu bakar file tersebut ke NAND Flash menggunakan menu **Program Flash** seperti pada langkah di atas. Board akan menyala dan menjalankan aplikasi langsung dari NAND Flash!

* **Metode 3: Uji Coba Cepat via JTAG RAM (Debug Interaktif):**
  * Saat masih membuat program C di SDK dan ingin mencoba langsung tanpa copy ke SD atau flash:
  * Klik kanan project aplikasi $\rightarrow$ **Run As** $\rightarrow$ **Launch on Hardware (System Debugger)**.
  * Aplikasi akan langsung ter-download ke RAM DDR3 dan dieksekusi secara instan.

---

## Perbandingan Konsumsi Memori OCM (Debug vs Langsung Gass)

```
MEMORI INTERNAL ZYNQ OCM (Total Kapasitas: 256 KB)

[ Mode Debug ]
█████████████████████████████████░░░░░░░░░░░░░░░░
Dipakai FSBL: ~168 KB              Sisa untuk User: ~88 KB

[ Mode Langsung Gass (-Os) ]
█████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░
Dipakai FSBL: ~72 KB               Sisa untuk User: ~184 KB  (HEMAT 96 KB!)
```

### Hasil Akhir Saat Dinyalakan:
Begitu adaptor 12V dicolok:
1. NAND Flash menyala instan $\rightarrow$ jeda 1.5 detik hening.
2. MicroSD langsung dibaca secepat kilat.
3. Lampu **LED Hijau DONE langsung MENYALA seketika**, dan program FPGA/MCU Anda langsung berjalan penuh!
