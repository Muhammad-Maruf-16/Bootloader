# TUTORIAL DETAIL: MODIFIKASI MANDIRI FSBL XILINX SDK UNTUK EBAZ4205

Dokumen ini berisi panduan teknis langkah-demi-langkah bagi Anda untuk membuat, memodifikasi, dan mengompilasi **FSBL (First Stage Boot Loader)** sendiri secara mandiri menggunakan **Xilinx SDK / Vivado** untuk board **EBAZ4205 (Zynq-7010)**.

---

## DAFTAR ISI
1. [Mengapa FSBL Default Xilinx Perlu Dimodifikasi?](#1-mengapa-fsbl-default-xilinx-perlu-dimodifikasi)
2. [Langkah 1: Persiapan Block Design di Vivado](#langkah-1-persiapan-block-design-di-vivado)
3. [Langkah 2: Membuat Project FSBL Baru di Xilinx SDK](#langkah-2-membuat-project-fsbl-baru-di-xilinx-sdk)
4. [Langkah 3: Mengatasi Masalah Path Windows (Multiple Target Patterns)](#langkah-3-mengatasi-masalah-path-windows-multiple-target-patterns)
5. [Langkah 4: Modifikasi Driver MicroSD (Patch 1-Bit & Bypass CMD6)](#langkah-4-modifikasi-driver-microsd-patch-1-bit--bypass-cmd6)
6. [Langkah 5: Memperbaiki Bug MultiBoot Fallback (`image_mover.c`)](#langkah-5-memperbaiki-bug-multiboot-fallback-image_moverc)
7. [Langkah 6: Mengaktifkan Debug Serial Console UART (`fsbl_debug.h`)](#langkah-6-mengaktifkan-debug-serial-console-uart-fsbl_debugh)
8. [Langkah 7: Modifikasi Khusus "FSBL Chainloader" (NAND -> Delay 1.5s -> MicroSD)](#langkah-7-modifikasi-khusus-fsbl-chainloader-nand---delay-15s---microsd)
9. [Langkah 8: Build Project & Membuat File `BOOT.bin`](#langkah-8-build-project--membuat-file-bootbin)
10. [Langkah 9: Perintah Diagnostik Hardware via XSDB](#langkah-9-perintah-diagnostik-hardware-via-xsdb)

---

## 1. Mengapa FSBL Default Xilinx Perlu Dimodifikasi?

Board EBAZ4205 aslinya adalah motherboard mesin tambang kripto (*Antminer*). Ada 3 karakteristik hardware unik pada board ini yang **pasti membuat FSBL template bawaan Xilinx gagal**:

1. **Slot MicroSD (U7) Tidak Memiliki Resistor Pull-up Pabrikan:**
   * Di skematik aslinya, resistor pull-up jalur data `R160, R161, R162, R163, R164` berstatus **`NC` (Not Connected / Kosong)**.
   * Driver SD bawaan Xilinx selalu memaksa mode **4-bit bus width** dan mengirim perintah `CMD6` (*Bus Speed Switch*). Karena sinyal data di DAT1..DAT3 banyak noise kapasitif, perintah `CMD6` akan **hang/freeze selamanya dalam infinite loop**.
   * **Solusi kita:** Paksa driver berjalan di mode **1-bit bus width** (`DAT0` saja) dan matikan fungsi `CMD6`.
2. **MultiBoot Fallback Offset Bug:**
   * Jika BootROM sempat gagal membaca kartu SD satu kali saja pada cold-boot, register MultiBoot Zynq akan terisi nilai non-nol.
   * FSBL standar akan menghitung alamat awal partisi sebagai `MultiBootReg * 32KB` (mencari partisi di offset `0x8000`, bukan sektor 0). Akibatnya FSBL melempar error `Partition Header Load Failed` dan melompat ke `FsblFallback()`.
   * **Solusi kita:** Paksa `ImageStartAddress = 0` saat membaca MicroSD.
3. **Timing Cold-Boot:**
   * Mikrokontroler kartu MicroSD modern butuh waktu stabilisasi tegangan 3.3V sebelum siap merespons perintah boot.

---

## Langkah 1: Persiapan Block Design di Vivado

Sebelum mengekspor hardware ke SDK, pastikan konfigurasi Processing System (PS) di Vivado sudah benar:

1. Buka Block Design di Vivado, double-click blok IP **`ZYNQ7 Processing System`**.
2. Masuk ke menu **MIO Configuration**:
   * **Memory Interfaces $\rightarrow$ NAND Flash:** Centang Enable, pilih pin `MIO 0 2.. 14` (jika ingin menggunakan NAND Flash onboard).
   * **SD Peripheral $\rightarrow$ SD 0:** Centang Enable, pilih pin `MIO 40 .. 45`. **Uncheck (matikan) Card Detect (CD)** dan **Write Protect (WP)** karena di EBAZ4205 jalur ini tidak dihubungkan ke MIO Zynq.
   * **UART Peripheral $\rightarrow$ UART 1:** Centang Enable, arahkan ke EMIO (atau MIO yang sesuai) untuk melihat log serial terminal di baudrate 115200.
3. Masuk ke menu **DDR Configuration**:
   * Memory Part: Pilih **`MT41K128M16 JT-125`**.
   * Bus Width: Pilih **`16 Bit`**.
4. Klik **OK**, lalu Generate Bitstream.
5. Export Hardware:
   * Klik menu **File $\rightarrow$ Export $\rightarrow$ Export Hardware**.
   * Centang opsi **Include bitstream** $\rightarrow$ Klik **OK**.
   * Buka SDK via **File $\rightarrow$ Launch SDK**.

---

## Langkah 2: Membuat Project FSBL Baru di Xilinx SDK

1. Di Xilinx SDK, klik **File $\rightarrow$ New $\rightarrow$ Application Project**.
2. Beri nama project (contoh: `fsblnya`).
3. Pilih **Hardware Platform** yang baru saja Anda ekspor.
4. Target OS: `standalone`, Language: `C`.
5. Klik **Next**.
6. Pada halaman template aplikasi, pilih template **Zynq FSBL**.
7. Klik **Finish**. SDK akan otomatis membuat dua folder:
   * `fsblnya` (Project aplikasi FSBL Anda)
   * `fsblnya_bsp` (Board Support Package berisi library driver Xilinx)

---

## Langkah 3: Mengatasi Masalah Path Windows (Multiple Target Patterns)

> [!WARNING]
> Jika Anda mengimpor atau menyalin proyek dari komputer Windows ke Linux, build akan gagal dengan error:
> `make: *** [src/fsbl_hooks.d:1: multiple target patterns. Stop.]`

**Penyebab:** GCC di Windows menuliskan path file dependensi dengan huruf drive `E:\...`. GNU Make di Linux menganggap tanda titik dua (`:`) tersebut sebagai sintaks yang rusak.

**Cara Memperbaikinya:**
Hapus seluruh file `.d` lama dan file biner lama di dalam folder `Debug`:
```bash
cd <path_project>/fsblnya/Debug
rm -f src/*.d src/*.o fsblnya.elf
```
Setelah dibersihkan, Make di Linux akan membuat file dependensi baru yang bersih secara otomatis.

---

## Langkah 4: Modifikasi Driver MicroSD (Patch 1-Bit & Bypass CMD6)

Ini adalah modifikasi paling krusial agar MicroSD slot U7 EBAZ4205 bisa dibaca tanpa *hang*.

### 1. Salin Driver SD ke Folder `src/` Proyek Anda
Secara default, file `xsdps.c` berada di dalam folder library BSP (`fsblnya_bsp/ps7_cortexa9_0/libsrc/sdps_vX_X/src/xsdps.c`). Agar perubahan kita tidak tertimpa saat BSP di-regenerate:
1. Salin file `xsdps.c` dari folder BSP tersebut ke dalam folder **`fsblnya/src/`**.
2. Masukkan `xsdps.c` ke dalam aturan kompilasi Makefile folder `Debug/src/subdir.mk` (tambahkan `./src/xsdps.o` pada `OBJS` dan `C_DEPS`).

### 2. Edit File `xsdps.c`
Buka file `xsdps.c` di editor kode:

#### A. Tambahkan Definisi `MAX_TIMEOUT` (di baris ~115):
```c
#ifndef MAX_TIMEOUT
#define MAX_TIMEOUT        10000000U
#endif
```

#### B. Modifikasi Fungsi `XSdPs_CardInitialize` (di baris ~670–695):
Cari blok kode yang mengatur Bus Width dan Bus Speed:

```c
/* ================= KODE ASLI XILINX ================= */
if ((SCR[1] & WIDTH_4_BIT_SUPPORT) != 0U) {
    Status = XSdPs_Change_BusWidth(InstancePtr);
    if (Status != XST_SUCCESS) {
        Status = XST_FAILURE;
        goto RETURN_PATH;
    }
}

/* Get speed supported by device */
Status = XSdPs_Get_BusSpeed(InstancePtr, ReadBuff);
if (Status != XST_SUCCESS) {
    goto RETURN_PATH;
}
```

**UBAH MENJADI SEPERTI INI:**

```c
/* ================= MODIFIKASI KHUSUS EBAZ4205 ================= */
/* 
 * Jalur DAT1..DAT3 EBAZ4205 tidak memiliki pull-up dan banyak noise.
 * KITA PAKSA HANYA MENGGUNAKAN MODE 1-BIT (DAT0 SAJA)
 * DAN MEMATIKAN PERINTAH CMD6 (Get_BusSpeed) YANG SERING HANG.
 */
InstancePtr->BusWidth = XSDPS_1_BIT_WIDTH;

/* Matikan peralihan ke 4-bit */
/*
if ((SCR[1] & WIDTH_4_BIT_SUPPORT) != 0U) {
    Status = XSdPs_Change_BusWidth(InstancePtr);
    if (Status != XST_SUCCESS) {
        Status = XST_FAILURE;
        goto RETURN_PATH;
    }
}
*/

/* Matikan CMD6 get bus speed */
/*
Status = XSdPs_Get_BusSpeed(InstancePtr, ReadBuff);
if (Status != XST_SUCCESS) {
    goto RETURN_PATH;
}
*/
```

---

## Langkah 5: Memperbaiki Bug MultiBoot Fallback (`image_mover.c`)

Buka file **`fsblnya/src/image_mover.c`**:

Cari fungsi **`LoadBootImage()`** pada baris ~220:

```c
/* ================= KODE ASLI XILINX ================= */
        /*
         * Compute the image start address
         */
        ImageStartAddress = (MultiBootReg & PCAP_MBOOT_REG_REBOOT_OFFSET_MASK)
                                    * GOLDEN_IMAGE_OFFSET;
    }

    fsbl_printf(DEBUG_INFO,"Image Start Address: 0x%08lx\r\n",ImageStartAddress);
```

**TAMBAHKAN KODE INI TEPAT DI BAWAHNYA:**

```c
/* ================= MODIFIKASI MULTIBOOT FIX ================= */
    /*
     * Jika membaca dari MicroSD, kita selalu membaca dari offset 0 (partisi FAT32),
     * jangan biarkan sisa nilai register MultiBoot menggeser offset ke 0x8000!
     */
    extern u32 SDAccess(u32 SourceAddress, u32 DestinationAddress, u32 LengthBytes);
    if (MoveImage == SDAccess) {
        ImageStartAddress = 0;
    }

    fsbl_printf(DEBUG_INFO,"Image Start Address: 0x%08lx\r\n",ImageStartAddress);
```

---

## Langkah 6: Mengaktifkan Debug Serial Console UART (`fsbl_debug.h`)

Secara default, FSBL mematikan semua output teks serial untuk menghemat ruang memori. Agar status bootloader terlihat di layar serial monitor (Putty / Minicom / Serial Terminal 115200 baud):

Buka file **`fsblnya/src/fsbl_debug.h`**:
Tambahkan define `FSBL_DEBUG_INFO` pada baris ~55:

```c
#define DEBUG_GENERAL   0x00000001    /* general debug messages */
#define DEBUG_INFO      0x00000002    /* More debug information */

/* Tambahkan baris ini: */
#ifndef FSBL_DEBUG_INFO
#define FSBL_DEBUG_INFO
#endif

#if defined (FSBL_DEBUG_INFO)
#define fsbl_dbg_current_types ((DEBUG_INFO) | (DEBUG_GENERAL))
```

---

## Langkah 7: Modifikasi Khusus "FSBL Chainloader" (NAND -> Delay 1.5s -> MicroSD)

> [!NOTE]
> Gunakan modifikasi ini jika Anda ingin menerapkan **ide Chainloader**: FSBL dibakar di NAND Flash, lalu memberi jeda delay 1.5 detik, kemudian otomatis membaca bitstream dan aplikasi dari MicroSD.

Buka file **`fsblnya/src/main.c`**:
Cari blok `NAND BOOT MODE` pada baris ~410:

```c
/* ================= KODE ASLI XILINX ================= */
#ifdef XPAR_PS7_NAND_0_BASEADDR
    if (BootModeRegister == NAND_FLASH_MODE) {
        fsbl_printf(DEBUG_GENERAL,"Boot mode is NAND\n");

        Status = InitNand();
        if (Status != XST_SUCCESS) {
            fsbl_printf(DEBUG_GENERAL,"NAND_INIT_FAIL \r\n");
            OutputStatus(NAND_INIT_FAIL);
            FsblFallback();
        }
        MoveImage = NandAccess;
        fsbl_printf(DEBUG_INFO,"NAND Init Done \r\n");
    } else
#endif
```

**UBAH MENJADI SEPERTI INI:**

```c
/* ================= LOGIKA CHAINLOADER DENGAN DELAY ================= */
#ifdef XPAR_PS7_NAND_0_BASEADDR
    if (BootModeRegister == NAND_FLASH_MODE) {
        fsbl_printf(DEBUG_GENERAL, "\r\n============================================\r\n");
        fsbl_printf(DEBUG_GENERAL, "EBAZ4205 CHAINLOADER (NAND -> MicroSD)\r\n");
        fsbl_printf(DEBUG_GENERAL, "Menunggu 1.5 detik agar daya MicroSD stabil...\r\n");
        fsbl_printf(DEBUG_GENERAL, "============================================\r\n");
        
        /* JEDA DELAY 1.5 DETIK SESUAI LOGIKA ANDA */
        usleep(1500000);

#if defined(XPAR_PS7_SD_0_S_AXI_BASEADDR) || defined(XPAR_XSDPS_0_BASEADDR)
        /* Coba inisialisasi kartu MicroSD */
        Status = InitSD("BOOT.BIN");
        if (Status != XST_SUCCESS) {
            fsbl_printf(DEBUG_GENERAL, "MicroSD belum siap, fallback ke NAND...\r\n");
            Status = InitNand();
            if (Status != XST_SUCCESS) {
                OutputStatus(NAND_INIT_FAIL);
                FsblFallback();
            }
            MoveImage = NandAccess;
        } else {
            fsbl_printf(DEBUG_GENERAL, "MicroSD SIAP! Mengambil partisi dari kartu SD...\r\n");
            MoveImage = SDAccess;
            BootModeRegister = SD_MODE; /* Alihkan mode ke SD Card */
        }
#endif
    } else
#endif
```

---

## Langkah 8: Build Project & Membuat File `BOOT.bin`

### 1. Build Project FSBL:
* Di Xilinx SDK, klik kanan pada nama project `fsblnya` $\rightarrow$ klik **Clean Project**, lalu **Build Project**.
* Pastikan pada tab *Console* muncul pesan sukses:
  ```text
  Finished building target: fsblnya.elf
  ```

### 2. Membuat File `BOOT.bin` Menggunakan GUI SDK:
1. Klik kanan pada project aplikasi Anda (misal `LED_CODE` atau `fsblnya`) $\rightarrow$ pilih **Create Boot Image**.
2. Di jendela yang muncul:
   * **Output format:** `BIN`.
   * **Output path:** Tentukan lokasi penyimpanan file `BOOT.bin`.
3. Pada tabel **Boot image partitions**, susun partisi dengan urutan mutlak berikut:
   * **Partisi 1:** Klik *Add* $\rightarrow$ Pilih file biner `fsblnya.elf` $\rightarrow$ Partition type: **`bootloader`**.
   * **Partisi 2:** Klik *Add* $\rightarrow$ Pilih file bitstream FPGA Anda (`.bit`) $\rightarrow$ Partition type: **`datafile`**.
   * **Partisi 3:** Klik *Add* $\rightarrow$ Pilih file aplikasi C Anda (`.elf`) $\rightarrow$ Partition type: **`datafile`**.
4. Klik **Create Image**. File `BOOT.bin` yang siap dipakai akan terbentuk.

### 3. Khusus untuk `BOOT_NAND.bin` (Hanya FSBL Saja):
Jika membuat image untuk dibakar ke NAND Flash (Chainloader):
* Pada tabel partisi **hanya masukkan 1 file saja**: yaitu `fsbl_chainloader.elf` dengan tipe `bootloader`.
* Simpan sebagai `BOOT_NAND.bin` (ukurannya hanya ~134 KB).

---

## Langkah 9: Perintah Diagnostik Hardware via XSDB

Jika board mengalami kendala saat boot, Anda bisa memeriksa register status Zynq secara langsung lewat terminal terminal menggunakan tool `xsdb`:

Buka terminal dan jalankan:
```bash
/tools/Xilinx/SDK/2019.1/bin/xsdb
```

Ketikkan perintah-perintah berikut:
```tcl
# 1. Sambungkan ke JTAG
connect

# 2. Pilih Core ARM #0
targets -set -nocase -filter {name =~ "ARM*#0"}

# 3. Baca Status Boot Mode Pin (Resistor Hardware)
# Jika R2584 terpasang (NAND)  --> 0x00000004
# Jika R2577 terpasang (SD)    --> 0x00000005
mrd 0xF800025C

# 4. Baca Status Error BootROM
# 0x00000000 = Normal
# 0x0040200A = BootROM Error 200A (SD Init Timeout/Fail)
mrd 0xF8000258

# 5. Baca Status Konfigurasi FPGA (DONE LED)
# 0x0E00E07F = FPGA Kosong / Belum Ter-program
# 0x4C00E07F atau 0x4E00E07F = PCFG_DONE=1 (FPGA Berhasil Ter-program, LED Hijau ON!)
mrd 0xF8007000
```

---

## Rangkuman File Hasil Kompilasi Terakhir di Workspace Anda:

Semua file yang sudah dimodifikasi dan berhasil diuji coba tersimpan di direktori berikut:
* Project FSBL Patched: [fsbl.sdk/fsblnya/src/](file:///home/maruf/fsbl/fsbl.sdk/fsblnya/src/)
* Project NAND Chainloader: [nand_chainloader/](file:///home/maruf/fpga_ebaz4205/nand_chainloader/)
* Source Driver 1-Bit: [xsdps.c](file:///home/maruf/fpga_ebaz4205/nand_chainloader/fsbl_build/xsdps.c)
* Source MultiBoot Fix: [image_mover.c](file:///home/maruf/fpga_ebaz4205/nand_chainloader/fsbl_build/image_mover.c)
* Binary NAND Siap Pakai: [BOOT_NAND.bin](file:///home/maruf/fpga_ebaz4205/nand_chainloader/BOOT_NAND.bin)
