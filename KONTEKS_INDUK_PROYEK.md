# Konteks Induk Proyek Skripsi: Implementasi RISC-V mriscv pada FPGA Basys3

> **Dokumen ini adalah SUMBER KEBENARAN TUNGGAL** untuk seluruh proyek. Seluruh isinya telah diverifikasi langsung (lewat pembacaan source code, diff ke repositori upstream, simulasi fungsional, eksekusi sintesis nyata, dan/atau pengujian perangkat keras fisik). Fakta yang tercantum di sini menggantikan/membatalkan semua draft dan catatan sebelumnya bila terjadi perbedaan.
>
> **Status verifikasi ditandai eksplisit** di tiap bagian: OK terverifikasi, PERLU CEK perlu dicek pengguna, SALAH terbukti salah.
>
> **Terakhir diperbarui: 2026-10-09.** Sumber pembaruan: pembacaan langsung repo `~/mriscv-spi` (RTL, firmware, Makefile, log sintesis), simulasi ulang, P&R ulang, dan catatan pengguna di `~/mriscv-spi/CLAUDE.md`. Fakta yang berasal dari catatan `CLAUDE.md` (terutama hasil uji di *board*) ditandai "(catatan CLAUDE.md)" karena tidak diverifikasi ulang oleh asisten.
>
> ## RINGKASAN PERUBAHAN 2026-10-09
>
> - **UART TX, UART RX, dan pengendali I2C master perangkat keras SUDAH SELESAI** dan jalan di board (dulu tercatat "BELUM"). Ditambah GPIO bank 2. Bus sekarang **2 master, 9 slave**. Lihat Bagian 4B, 6.2-6.5.
> - **KOREKSI BESAR: `MULT.v` AKTIF**, bukan "tidak dipanggil". Core = **RV32I + Zmmul**. DSP48 = 0 karena multiplier-nya iteratif (dipetakan ke LUT/FF), BUKAN bukti MULT tidak tersintesis. Lihat Bagian 7 dan 9.2. **Naskah sudah dikoreksi 2026-10-09** (Bab III 3.4.2, Bab IV 4.2 dan 4.5.1).
> - **Perbaikan baru di RTL:** dekode CSR di `UTILITY.v` (rdcycle dulu bikin FSM macet), override `.masters/.slaves` interconnect, `BUS_RST`, sinkronizer reset 3-FF, LED trap LD15. Lihat Bagian 4 dan 10.
> - **Hang "`uart_puts()` > 3 karakter" ternyata bug linker script** (`.bss` tidak rata 4 byte), bukan hardware. Lihat Bagian 10 #14.
> - **Data kuantitatif desain akhir** (UART + I2C): Yosys 2532 LUT / 1636 FF / 8 RAMB36 / 0 DSP / 241 CARRY4; Fmax seed 13 = 60,87 MHz PASS @50 MHz. Lihat Bagian 9.4-9.6.
> - **Naskah Bab III dan Bab IV (`.tex`) sudah diperbarui** dengan UART/I2C pada 2026-10-09 (subbab 3.4.4 dan 4.6). Lihat Bagian 19.
>
> ## CATATAN PENTING UNTUK PENULISAN BAB III - BACA DULU SEBELUM PAKAI DOKUMEN INI
>
> **Bab III (Metode Penelitian) TIDAK LAGI membahas ASIC sama sekali** - ini keputusan penulis yang disengaja, bukan kelalaian. Bagian 2, 2B, dan 3 di bawah tetap dipertahankan di dokumen ini sebagai **fakta latar belakang terverifikasi** (berguna untuk Bab II/tinjauan pustaka atau referensi internal), TAPI:
> - **JANGAN** masukkan kerangka "warisan ASIC", "konversi ASIC ke FPGA", atau bukti tiga-lapis ASIC ke draft Bab III manapun.
> - Bab III sekarang membingkai modifikasi RTL murni sebagai **audit kompatibilitas toolchain** (Yosys/nextpnr vs Vivado), tanpa embel-embel asal-usul ASIC.
> - Bab III 3.5.1 juga sudah **disederhanakan jadi deskripsi prosedural singkat** (5 paragraf pendek, tanpa cuplikan kode, tanpa narasi hipotesis-diuji-gagal). Analisis mendalam, cuplikan kode before/after, dan hasil pengujian dipindah ke **Bab IV**.
> - Kalau diminta menulis ulang Bab III, ikuti pola ini secara ketat.
> - **Catatan 2026-10-09:** penomoran "3.2.1 Kerangka Konseptual" dan "3.5.1 prosedural audit" di atas dan di Bagian 19 merujuk draf lama. `chapter-3.tex` saat ini tidak memuat keduanya (blok "Tahap Modifikasi Repository" masih di-comment), dan 3.5.1 di `.tex` = Analisis Fungsional. Prinsipnya (tanpa ASIC, Bab III prosedural, detail di Bab IV) tetap berlaku. Struktur `.tex` aktual ada di Bagian 19.

---

## BAGIAN 1 - Identitas Proyek

**Judul kerja:** Implementasi dan Verifikasi Prosesor RISC-V RV32I pada FPGA Basys3 dengan Mekanisme Pemuatan Program Berbasis SPI Menggunakan Toolchain Sumber Terbuka

**Target hardware:** Digilent Basys3, Xilinx Artix-7 XC7A35T-1CPG236C

**Core:** `onchipuis/mriscv` - soft-core RV32I, lisensi MIT, dikembangkan OnChipUIS (Universidad Industrial de Santander, Kolombia)

**Toolchain:** 100% open-source, tanpa Vivado
- Yosys (sintesis RTL -> netlist)
- nextpnr-xilinx (place & route)
- fasm2frames + xc7frames2bit (prjxray, -> bitstream)
- openFPGALoader (flash via JTAG)
- riscv64-unknown-elf-gcc v13.2.0 (compile firmware C) - **PERLU CEK:** mesin pengguna per 2026-10-09 memakai GCC 14.2.0 (terlihat saat compile `mul_test`)
- Total Phase Cheetah (SPI adapter untuk upload program runtime)
- Icarus Verilog (`iverilog`/`vvp`) untuk testbench unit dan simulasi SoC penuh (versi belum dicatat)
- Terminal serial di PC untuk UART 115200 8N1 (lewat virtual COM port FTDI FT2232HQ di kabel USB-JTAG yang sama, biasanya `/dev/ttyUSB1`)

**Karakteristik unik:** program di-upload ke RAM core lewat **SPI runtime loader**, bukan lewat re-synthesis. Bitstream (hardware) dan program (software) sepenuhnya independen.

**Periferal yang aktif (per 2026-10-01):** GPIO bank 1 (switch, LED, seven-segment 2 digit), GPIO bank 2 (rotary encoder + jalur I2C OLED bit-bang), UART TX, UART RX, I2C master hardware, LED trap (LD15). Satu bitstream (`basys3_top.v` tunggal) untuk semua firmware.

---

## BAGIAN 2 - Bukti mriscv Didesain untuk ASIC (fakta latar belakang, TIDAK dipakai di Bab III)

> Lihat catatan di judul dokumen - bagian ini untuk referensi/Bab II, bukan Bab III.

### Lapis 1 - Literatur

Sitasi resmi (dikonfirmasi langsung dari IEEE Xplore):

> C. Duran, D. L. Rueda, G. Castillo, A. Agudelo, C. Rojas, L. Chaparro, H. Hurtado, J. Romero, W. Ramirez, H. Gomez, J. Ardila, L. Rueda, H. D. Hernandez, J. Amaya, dan E. Roa, "A 32-bit RISC-V AXI4-lite bus-based microcontroller with 10-bit SAR ADC," dalam *2016 IEEE 7th Latin American Symposium on Circuits & Systems (LASCAS)*, Florianopolis, Brazil, **2016**, hlm. 315-318. DOI: 10.1109/LASCAS.2016.7451073

**PENTING:** tahun yang benar adalah **2016**, bukan 2022. Venue adalah konferensi IEEE peer-review.

Fakta dari abstrak paper: implementasi *fully-synthesized* 130nm CMOS, footprint 798 um x 484 um, kepadatan daya 167 uW/MHz, berisi 10-bit SAR ADC, 12-bit DAC, 8-bit GPIO, 4 kB RAM, bus AXI4-Lite & APB.

### Lapis 2 - Struktural

Repositori upstream `onchipuis/mriscv` berisi berkas `Testing_board_CIDIC_rev_aamaya.csv` dengan komponen bernama `CIDIC_AXI_UC_TEST_DIE` - istilah "die" merujuk potongan silikon hasil wafer.

### Lapis 3 - RTL langsung

- Sinyal `DSE` per-pin di `completogpio.v` (*Drive Strength Enable*, konsep pad-cell fisik)
- Port DAC 12-bit dan ADC 10-bit yang menggantung/di-tie nol di top module - sesuai spesifikasi paper [3]
- Penamaan `SP32B1024` - konvensi SRAM compiler foundry, tidak ditemukan di repo upstream sama sekali

---

## BAGIAN 2B - RTL Unpatched SUDAH Dapat Disintesis ke FPGA (fakta latar belakang)

> Fakta ini tetap penting untuk memahami proyek, tapi **framing "konversi ASIC ke FPGA" tidak dipakai di Bab III**. Gunakan versi netral: "RTL upstream yang belum dimodifikasi terbukti dapat disintesis Vivado" - tanpa embel-embel ASIC.

Repositori `onchipuis/mriscv_vivado` - implementasi FPGA oleh tim pengembang mriscv sendiri - berisi bitstream jadi:
```
File     : impl_axi_fpga.bit  (3,8 MB)
Target   : 7a100tcsg324  (Artix-7 XC7A100T, board Nexys4 DDR)
Toolchain: Vivado 2016.4
Tanggal  : 12 April 2017
```
Core yang dipakai adalah submodule `onchipuis/mriscv.git` **belum dipatch sama sekali** - tri-state `32'bz` dan wired-OR `rd` masih ada, dan berhasil jadi bitstream fungsional.

**Konsekuensi:** klaim "tri-state/latch/dual-edge wajib diganti karena FPGA tidak mendukungnya" tidak akurat sebagai generalisasi. Yang akurat: **Vivado** menangani semuanya otomatis. **Yosys** menangani sebagian (lihat Bagian 5 - tri-state di `axi4_interconnect.v` ditangani otomatis, tapi tri-state di `mriscvcore.v`/`ALU.v` TERBUKTI TIDAK - lihat 5.3, temuan besar sesi ini).

**Kontribusi penelitian ini** (tanpa menyebut ASIC): (1) portasi ke toolchain sumber terbuka penuh (Yosys, nextpnr-xilinx, prjxray) dengan kendala berbeda dari Vivado; (2) target FPGA lebih kecil (XC7A35T vs XC7A100T); (3) arsitektur SPI runtime loader yang memisahkan siklus sintesis dari siklus pemrograman; (4) karakterisasi kuantitatif Fmax; (5) ekspansi RAM 4->32 KB; (6) audit sistematis yang membedakan modifikasi wajib vs opsional vs harus-dipertahankan berdasarkan pengujian nyata (bukan asumsi); (7) penambahan periferal UART TX/RX dan pengendali I2C master perangkat keras yang ditulis sendiri sebagai slave AXI4-Lite, diverifikasi tiga tingkat (testbench unit, simulasi SoC, board).

---

## BAGIAN 3 - Perbedaan Fisik ASIC vs FPGA (fakta latar belakang, TIDAK dipakai di Bab III)

RTL hanya mendeskripsikan perilaku logis; realisasi fisik ditentukan saat sintesis. ASIC: silikon custom, RTL bebas konstruksi apapun. FPGA: blok generik tetap (LUT/FF/BRAM), sintesis = pemetaan bukan penciptaan sirkuit baru.

**Nuansa penting:** "tidak ada padanan langsung" TIDAK SAMA DENGAN "tidak dapat disintesis" - tool sintesis modern (Vivado, Yosys) bisa mentransformasikan banyak konstruksi otomatis (lihat TRIBUF, Bagian 5.1).

Data kuantitatif [Darvishi]: pada FPGA, *routing delay* 62-74% dari critical path (rata-rata 68%); pada ASIC, *logic delay* mendominasi (55-65%). **Catatan:** sitasi Darvishi [14] tetap dipakai di Bab III 3.5.2 dan Bab IV 4.5.2 (lokasi per `.tex` 2026-10-09) untuk konteks *lain* (variabilitas Fmax antar-seed P&R) yang tidak menyinggung perbandingan ASIC sama sekali - itu pemakaian yang sah dan dipertahankan.

---

## BAGIAN 4 - Audit Modifikasi RTL: Status & Tingkat Keperluan (DIPERBARUI - perubahan besar di #4)

**Metodologi audit:** repo kerja adalah hasil `clone` langsung dari `onchipuis/mriscv` (upstream) yang dimodifikasi bertahap selama pengembangan - bukan ditulis terpisah lalu dibandingkan. `diff` byte-demi-byte terhadap upstream dipakai secara **retrospektif** untuk mengatalogkan seluruh perubahan final menjadi tabel di bawah ini; metode penemuan tiap modifikasi yang sebenarnya (kegagalan elaborasi, warning sintesis penuh, simulasi, atau uji hardware) bervariasi per kasus dan tercatat di kolom "Cara ditemukan" pada tabel Bagian 10. Kolom "Keperluan" sekarang didasarkan pada **bukti pengujian nyata** (sintesis + hardware), bukan lagi asumsi dari perbandingan ke `mriscv_vivado` saja.

| # | Modul | Status modifikasi | **Tingkat keperluan** |
|---|---|---|---|
| 1 | `SP32B1024.v` | Ditulis dari nol | **WAJIB** - modul tidak ada di rilis open-source |
| 2 | `AXI_SP32B1024.v` | **TIDAK DIPATCH** | **TIDAK PERLU** - patch terbukti merusak (Bagian 5.2) |
| 3 | `axi4_interconnect.v` | Dipatch (2 baris) | **OPSIONAL** - perbaikan kualitas (latch->FF). Konstruksi asli terbukti dapat disintesis Yosys (lihat 5.1) |
| 4 | `mriscvcore.v` + `ALU.v` | Dipatch | **WAJIB - DIUBAH DARI "OPSIONAL"** - terbukti lewat sintesis PENUH + 2x uji hardware bahwa RTL tanpa patch ini menyebabkan seluruh bus write-back register di-*hardwire* jadi konstanta 0 (lihat 5.3, temuan besar) |
| 5 | `spi_axi_master.v` | Ditulis ulang (445->101 baris) | **WAJIB untuk arsitektur ini** - konsekuensi pilihan desain SPI runtime loader |
| 6 | `basys3_spi.xdc` | Dimodifikasi dari `basys3_master.xdc` | **WAJIB** - kebutuhan struktural setiap proyek FPGA |
| 7 | `bus_sync_sf.v` | Dipatch (152 baris) | **OPSIONAL** - praktik baik FPGA (hapus inverted clock net) |
| 8 | `UTILITY.v` (BARU 2026-09-28) | Dipatch: tambah dekode CSRRS/CSRRC/CSRRWI/CSRRSI/CSRRCI | **WAJIB untuk firmware yang memakai `rdcycle`/`rdinstret`/`rdtime`** (mis. `oled_i2c_stress.c`). Tanpa patch, `UTILITY.v` hanya mencocokkan funct3=000 (ECALL), tidak ada unit yang menaikkan `done_exec`, FSM macet permanen di `S2_exec` (tanpa trap). CSR bersifat read-only; CSRRW tetap tidak didukung (kodenya sama dengan EBREAK, di-trap FSM) |
| 9 | `impl_axi.v` | Dipatch berat | **WAJIB** - override eksplisit `.masters/.slaves/.addr_mask/.addr_use` ke `axi4_interconnect` (dua bug kelas sama, Bagian 10 #3 dan #11), peta alamat 9 slave, `BUS_RST`, instansiasi 4 slave baru |

**Ringkasan terbaru:** dari 9 modifikasi upstream, **6 wajib** (#1, #4, #5, #6, #8, #9), dua opsional (#3, #7), satu dibatalkan (#2). **Catatan naskah:** Tabel "Ringkasan Pemeriksaan Modifikasi RTL" di Bab IV baru memuat 6 baris (belum ada `UTILITY.v` dan `impl_axi.v`).

---

## BAGIAN 4B - Penambahan RTL Baru (bukan modifikasi upstream)

Semua modul di bawah ditulis sendiri (tidak ada modul UART/I2C di RTL upstream `onchipuis/mriscv` - dicek lewat grep 2026-10-09). Semua slave baru di-reset oleh `BUS_RST`.

| Modul | Slave | Status | Verifikasi |
|---|---|---|---|
| `UART_TX/uart_tx.v` (183 baris) | 5 | Selesai 2026-09-28 | Testbench unit PASS (dijalankan ulang 2026-10-09); board: `uart_hello`, `uart_stress` (catatan CLAUDE.md) |
| `UART_RX/uart_rx.v` (164 baris) | 6 | Selesai 2026-09-30 | Testbench unit PASS + SoC `impl_axi_uartrx_tb.v` + `uart_echo` PASS (keduanya dijalankan ulang 2026-10-09); board: `uart_echo` (catatan CLAUDE.md) |
| `completogpio.v` instansiasi kedua (GPIO bank 2) | 7 | Selesai 2026-10-01 | Simulasi `basys3_top_gpiobank_tb.v` + `gpio_bank_test.c` PASS (catatan CLAUDE.md) |
| `I2C/i2c_master.v` (217 baris) | 8 | Selesai 2026-10-01 | Testbench unit PASS + SoC `basys3_top_i2c_tb.v` + `i2c_test` PASS (keduanya dijalankan ulang 2026-10-09); board: `oled_hw`, `oled_i2c_stress` (catatan CLAUDE.md) |
| `basys3_top.v` (top tunggal) | - | Diperbarui | Sinkronizer reset 3-FF, LED trap LD15, `oled_scl/sda` jadi `inout` + wired-AND GPIO bank 2 & I2C master |
| `BUS_RST` di `impl_axi.v` | - | Selesai 2026-09-28 | Simulasi `impl_axi_busrst_tb.v` + board (catatan CLAUDE.md) |

Referensi desain I2C menurut catatan pengguna: OpenCores i2c (Herveille) dan `verilog-i2c` (Forencich), ditulis ulang sendiri. **Belum disitasi di naskah.**

### Detail patch #1 - SRAM -> BRAM behavioral
```verilog
integer i;
initial begin
    if (INIT_FILE != "") $readmemh(INIT_FILE, mem);
    else for (i = 0; i < 8192; i = i + 1) mem[i] = 32'h0;
end
```
Tanpa blok ini, isi BRAM tidak terdefinisi setelah konfigurasi Artix-7 -> kegagalan booting acak.

### Detail patch #3 - Latch yang DISENGAJA (bukan "inadvertent")
Kode upstream, `axi4_interconnect.v`:
```verilog
// Phase 1 - Convert to latched array
always @(m_axi_awvalid[unpk_idx] or RST or m_axi_awaddr[...]) begin
    if(RST == 1'b0) m_axi_awaddr_g[unpk_idx] = {sword{1'b0}};
    else if (m_axi_awvalid[unpk_idx] == 1'b1) m_axi_awaddr_g[unpk_idx] = m_axi_awaddr[...];
end
```
Komentar asli `"Convert to latched array"` membuktikan latch ini disengaja (hemat area silikon), bukan kecelakaan koding. Setelah dipatch: `always @(posedge CLK) begin`.

### Detail patch #4 - Tri-state core read path -> OR-mux (SEKARANG WAJIB, lihat 5.3)
Upstream: 5 modul (`MEMORY_INTERFACE`, `ALU`, `IRQ`, `MULT`, `UTILITY`) sama-sama menggerakkan wire `rd` (wired-OR/tri-state), dikonsumsi `REG_FILE` untuk write-back register.

Setelah dipatch, di `mriscvcore.v`:
```verilog
wire [31:0] rd_alu, rd_mult, rd_util, rd_mem, rd_irq;
assign rd = rd_alu | rd_mult | rd_util | rd_mem | rd_irq;
```
Dan di `ALU.v`: `assign rd = is_rd_nr ? OUT_Alu_rd : 32'b0;` (bukan lagi `32'bz`).

### Detail patch #5 - Multi-clock -> single-clock
Upstream `spi_axi_master.v` (445 baris): dua clock domain (`posedge SCLK` x7, `posedge CLK` x2). Setelah ditulis ulang (101 baris): hanya `posedge CLK` (x3), deteksi SCLK lewat oversampling.

---

## BAGIAN 5 - Temuan Detail: Tri-state, negedge, dan Verifikasi Hardware

### 5.1 Tri-state di `axi4_interconnect.v` - TIDAK dipatch, ditangani Yosys otomatis (TRIBUF)

`axi4_interconnect.v` masih berisi kode tri-state (`{sword{1'bz}}`), parameter default `addressing=0` (mode tri-state) tidak di-override di manapun.

**Verifikasi (2x sintesis independen, Yosys 0.33 & 0.38):** 0 primitif TBUF di netlist, 0 error, `TRIBUF pass` dieksekusi otomatis. Warning `"Yosys has only limited support for tri-state logic"` normal, bukan error.

**Kesimpulan:** Yosys `synth_xilinx` punya *pass* internal **TRIBUF** yang otomatis mengonversi pola tri-state **lokal sederhana** (satu `assign` dengan ternary) jadi mux. Modul ini berhasil tanpa RTL ditulis ulang.

### 5.2 `negedge` di `AXI_SP32B1024.v` - DISENGAJA, patch naif TERBUKTI MERUSAK

Kode upstream (dipertahankan): `always @(negedge CLK) begin` dengan komentar developer *"we provide the signals in negedge, because the setup and hold sh*t"*.

**Hipotesis awal:** praktik ASIC yang perlu dikonversi ke `posedge`.

**Hasil uji empiris** (simulasi `iverilog`, testbench protokol SPI nyata): konversi ke `posedge` **merusak fungsi tulis SRAM total** - akibat semantik *non-blocking assignment*, `SP32B1024` membaca alamat `A` satu siklus tertinggal.

**Kesimpulan:** relasi `negedge`<->`posedge` adalah pipa data setengah-siklus yang disengaja. **Patch dibatalkan.** Konfirmasi netlist: `FDRE_1`+`FDSE_1` = 47 flip-flop negative-edge tetap ada.

### 5.3 TEMUAN BESAR SESI INI - Tri-state di `mriscvcore.v`/`ALU.v` TERBUKTI MERUSAK FUNGSI (via sintesis PENUH + uji hardware 2x)

Ini investigasi lanjutan yang **mengoreksi status "OPSIONAL"** yang sebelumnya diberikan ke patch #4 (Bagian 4).

**Langkah 1 - Sintesis Yosys-saja (stat only):** Revert `mriscvcore.v`/`ALU.v` ke versi upstream (tri-state asli), sisanya tetap patched. Hasil: exit code 0 (tidak fatal), tapi **32 warning "multiple conflicting drivers"** - satu per bit dari `rd` 32-bit, menyebut kelima modul (`ALU`, `IRQ`, `MEMORY_INTERFACE`, `MULT`, `UTILITY`) sebagai driver yang bentrok. Total cell **1691** (dibanding **3996** versi patched, dikonfirmasi identik di 3 log sintesis independen: `mriscv-spi/synth_log.txt`, `tristate_test/synth_log.txt`, `tristate_hwtest/synth_log.txt`) - indikasi banyak logika terbuang. **Direplikasi independen di mesin pengguna sendiri: 1689 cell, 32 warning, identik.**

**Langkah 2 - Sintesis PENUH via `make synth` (bukan cuma Yosys-saja):** Warning yang muncul berbeda nama - **"Driver-driver conflict"**, total **128 warning** (32 bit x 4 modul: `ALU`, `MEMORY_INTERFACE`, `MULT`, `UTILITY` - `IRQ` tidak muncul di pencarian ini, kemungkinan selisih tahap pipeline, belum diverifikasi tuntas kenapa). **Setiap satu dari 128 konflik diselesaikan Yosys dengan cara yang sama: `Resolved using constant.` -> nilai `1'0`.** Artinya net `rd` di netlist akhir pada dasarnya **selalu bernilai 0**, apapun instruksinya.

**Langkah 3 - Kenapa ini merusak SEMUA eksekusi, bukan cuma `lw`:** `rd` bukan cuma jalur hasil `MEMORY_INTERFACE` (load) - ini **satu-satunya bus write-back ke `REG_FILE`**, dipakai bersama oleh SEMUA modul yang menghasilkan nilai untuk ditulis ke register (termasuk `ALU` untuk operasi aritmatika biasa seperti `add`/`addi`/`li`). Karena `rd` di-hardwire 0, **setiap instruksi yang menulis register - bukan cuma `lw` - mendapat hasil 0.**

**Langkah 4 - Uji hardware, dua firmware independen, program di-bake langsung ke bitstream via `INIT_FILE`:**

*Firmware 1* (`test_tristate.c`): `sw` nilai `0xAA` ke RAM alamat `0x400`, `lw` kembali, tampilkan ke LED lewat `led_set()`.
-> **Hasil: LED mati semua.** (Ekspektasi benar: pola `10101010`.)

*Firmware 2 - uji kontrol* (`led_set(0xAA)` langsung, **tanpa `lw` sama sekali**, murni `li a0,0xAA` (ALU) lalu `sw` ke GPIO):
-> **Hasil: LED mati semua juga.**

Awalnya firmware 2 dirancang untuk "mengisolasi" apakah masalah spesifik di `lw`. Tapi karena `li`/`addi` **juga** lewat bus `rd` yang sama (write-back ALU), firmware 2 **bukan kontrol yang valid** untuk isolasi itu - desain uji kontrol yang salah, ditemukan lewat analisis log, bukan lewat uji hardware ketiga.

**Verifikasi mekanisme `INIT_FILE` bekerja (menyingkirkan kemungkinan alternatif):** parameter `INIT_FILE` dibaca langsung dari netlist hasil sintesis dalam bentuk 136-bit biner, didekode manual (8 bit/karakter) -> persis menghasilkan string `"test_tristate.hex"`. **Dikonfirmasi ulang secara independen.** Ini membuktikan mekanisme *loading* firmware langsung ke bitstream (tanpa SPI loader) bekerja dengan benar - kegagalan LED **bukan** karena firmware gagal termuat, melainkan karena bug tri-state itu sendiri.

**Kesimpulan final:** patch #4 (`mriscvcore.v`/`ALU.v`, tri-state -> OR-mux) **WAJIB**, bukan opsional. Berbeda dari `axi4_interconnect.v` (5.1) di mana TRIBUF pass menangani tri-state otomatis dengan sempurna, pola tri-state **lintas-modul** (banyak instansiasi modul terpisah berbagi satu wire lewat port, bukan satu `assign` lokal) **TIDAK tertangani** oleh TRIBUF - Yosys malah menyelesaikannya secara diam-diam jadi konstanta 0, sebuah *silent failure* yang lolos dari `CHECK pass` Yosys (yang melaporkan "0 problems" - `CHECK` memang tidak memeriksa driver-driver conflict, jadi ini bukan kontradiksi, tapi cakupan pemeriksaan yang berbeda).

**Status pengujian:** *positive control* (flash ulang versi PATCHED/OR-mux dengan firmware yang sama, verifikasi LED menyala `10101010`) **direkomendasikan tapi belum dilaporkan hasilnya** - lihat Bagian 17.

---

## BAGIAN 6 - Arsitektur Core mriscv

### 6.1 Multi-cycle FSM, bukan pipeline

FSM 7-state, strictly sequential:
```
S0_fetch -> S1_decode -> S2_exec -> S3_memory -> (kembali S0_fetch)
                v (instruksi ilegal / error bus)
             S4_trap (HALT PERMANEN, perlu reset eksternal)
+ SW0_fetch_wait, SW3_mem_wait - wait state latensi BRAM sinkron
```
**Implikasi:** CPI > 1 (minimal 4 siklus/instruksi). Hazard control unit tidak diperlukan (tidak ada overlap instruksi). Landasan teori: ASMD-FSMD [Solov'ev; Harris & Harris].

### 6.2 Hierarki modul

```
basys3_top.v          <- lapisan papan (POR, clock /2, sinkronizer reset 3-FF, sinkronisasi
                         switch & encoder 2-FF, open-drain I2C wired-AND, 7-seg 2 digit, LED trap)
  impl_axi.v           <- lapisan "chip" (BUS_RST + 12 instansiasi)
    mriscvcore.v       <- CPU (master #0)
        FSM.v, DECO_INSTR.v, ALU.v, REG_FILE.v
        MEMORY_INTERFACE.v (pintu ke bus)
        MULT.v (AKTIF, Zmmul), IRQ.v, UTILITY.v (CSR dipatch)
    spi_axi_master.v   <- SPI loader (master #1)
    axi4_interconnect.v <- bus (2 master, 9 slave)
    AXI_SP32B1024.v + SP32B1024.v  <- RAM (slave 0)
    DAC_interface_AXI.v  <- slave 1 (tidak terpakai)
    ADC_interface_AXI.v  <- slave 2 (tidak terpakai)
    completogpio.v       <- slave 3 (GPIO bank 1: switch/LED/7-seg)
    spi_axi_slave.v      <- slave 4
    uart_tx.v            <- slave 5 (BARU)
    uart_rx.v            <- slave 6 (BARU)
    completogpio.v       <- slave 7 (GPIO bank 2: encoder + OLED bit-bang, BARU)
    i2c_master.v         <- slave 8 (BARU)
```
**Kunci SPI loader:** `spi_axi_master` adalah master sejajar CPU di bus - bisa menulis RAM sendiri tanpa lewat CPU.

**Penomoran:** komentar di `impl_axi.v` memakai penomoran 1-based ("Slave 1..9", "Master 1..2"); naskah dan dokumen ini memakai indeks 0-based (slave 0..8, master 0..1).

### 6.3 Peta alamat AXI (9 slave, per 2026-10-01)
```verilog
localparam slaves = 9;
addr_mask = {32'h00000003,32'h0000000F,32'h00000000,32'h00000000,32'h00000000,32'h0000000F,32'h00000001,32'h00000001,32'h00001FFF};
addr_use  = {32'h00004040,32'h00004030,32'h00004021,32'h00004020,32'h04000000,32'h00004010,32'h00004008,32'h00004000,32'h00000000};
```
Dibaca dari kanan (slave 0 = elemen paling kanan). Byte address = 4 x word address.

| Slave | Peripheral | Word address | Byte address | Catatan |
|---|---|---|---|---|
| 0 | SRAM | `0x0000`-`0x1FFF` | `0x00000`-`0x07FFF` | 8192 words = 32 KB |
| 1 | DAC | `0x4000` | `0x10000` | Tidak terpakai, `dac_data` menggantung |
| 2 | ADC | `0x4008` | `0x10020` | Tidak terpakai, di-tie nol |
| 3 | GPIO bank 1 | `0x4010`-`0x401F` | `0x10040 + i*4` | `pindata` = switch, `datanw` -> LED + 7-seg |
| 4 | SPI slave | `0x04000000` | - | Output menggantung |
| 5 | UART TX | `0x4020` | `0x10080` | mask 0 (exact) |
| 6 | UART RX | `0x4021` | `0x10084` | mask 0 (exact) |
| 7 | GPIO bank 2 | `0x4030`-`0x403F` | `0x100C0 + i*4` | `pindata[0]`=enc CLK, `[1]`=enc DT; `datanw[0]`=OLED SCL, `[1]`=SDA |
| 8 | I2C master | `0x4040`-`0x4043` | `0x10100`-`0x1010C` | mask 0x3, 4 register |

### 6.4 GPIO (`completogpio`, dua bank)
`pindata` (input) dan `datanw` (output) sinyal terpisah, berbagi indeks pin. Tulis: `0x3`=HIGH, `0x2`=LOW (bit1 = DSE, selalu 1). Baca: bit0 = `pindata[i]`. HAL: `gpio_pin()/gpio_rd()` (bank 1), `gpio2_pin()/gpio2_rd()` (bank 2). Bank 2 dibuat supaya encoder/OLED tidak berbagi indeks pin dengan switch/LED. **Firmware lama yang membaca encoder lewat `gpio_rd(0/1)` harus diganti `gpio2_rd`.**

### 6.5 UART dan I2C master (desain, terverifikasi dari source)

Ketiga modul memakai pola AXI **non-blocking**: transaksi tulis/baca selalu selesai dalam beberapa siklus; tulis saat modul sibuk **diterima tapi diabaikan**; firmware wajib polling bit busy. Parameter `CLK_HZ=50_000_000` di instansiasi **wajib ikut clock core** kalau clock diubah.

**UART TX** (`0x10080`): 8N1, LSB-first, 115200 baud tetap. `DIV = CLK_HZ/BAUD = 434` -> baud aktual 115.207 (galat 0,0064%). FSM 4 state (IDLE/START/DATA/STOP). Tulis = kirim `WDATA[7:0]`; baca bit0 = busy. `busy = (tstate != S_IDLE) || start_pulse` (fix race, Bagian 10 #12). Pin A18. 1 byte = 10 bit x 434 = 4340 siklus.

**UART RX** (`0x10084`): sinkronizer 2-FF, cek ulang di tengah start bit (glitch < 1/2 bit diabaikan), sampel di tengah bit, stop bit 0 = framing error. Holding register 1 byte, **tanpa FIFO**. Baca = pop: `[7:0]` data, `[8]` valid, `[9]` overrun, `[10]` ferr; membaca menghapus flag. Tulis diterima & diabaikan. Pin B18.

**I2C master** (level byte, 7-state FSM, tiap bit SCL = 4 fase):

| Byte addr | Reg | Fungsi |
|---|---|---|
| `0x10100` | DATA | tulis: byte TX; baca: byte RX terakhir |
| `0x10104` | CMD | bit0 START, bit1 STOP, bit2 WRITE, bit3 READ, bit4 NACK. Satu tulis = satu operasi `[START/rep. START] -> [kirim/terima 1 byte] -> [STOP]` |
| `0x10108` | STATUS | bit0 busy, bit1 nack, bit2 arb_lost |
| `0x1010C` | DIV | siklus clock per 1/4 periode SCL, min 4, default 31 |

- **DIV default 31 -> SCL = 50 MHz/(4x31) = 403,2 kHz**, 0,8% di atas batas Fast-mode 400 kHz (31,25 dibulatkan ke bawah). DIV=32 -> 390,6 kHz. SSD1306 jalan normal di 403 kHz.
- Mendukung clock stretching (maju hanya kalau SCL sudah benar-benar high), deteksi ACK/NACK, deteksi arbitration lost.
- 1 byte + ACK = 9 x 124 = 1116 siklus. Overhead polling firmware ~290 siklus/byte (diukur di simulasi, catatan di `oled_i2c_stress.c`) -> engine aktif ~79% saat transfer beruntun.
- **Wired-AND di top:** `oled_scl = (gpio2_datanw[0] & i2c_scl_o) ? 1'bz : 1'b0;` (sama untuk SDA). Firmware bit-bang lama tetap jalan tanpa mux. **GPIO bank 2 reset ke 0 = bus ditarik low** -> firmware I2C hardware WAJIB panggil `i2c_hw_init()` dulu.

---

## BAGIAN 7 - Status Fitur Core (DIPERBARUI 2026-10-09 - ada koreksi besar)

| Subsistem | Status | Bukti |
|---|---|---|
| `MULT.v` (Karatsuba+Booth iteratif, MUL/MULH/MULHSU/MULHU) | **AKTIF** - core = **RV32I + Zmmul** | `MULT_inst` diinstansiasi di `mriscvcore.v:178`; `enable_mul = enable_exec` (baris 240), `done_mul & is_inst_mul` ikut `done_exec` (244), `done_mul` ikut `rdw_rsrn` (250) - dicek 2026-10-09. Netlist akhir memuat ratusan sel LUT bernama `MULT_inst` (dicek di `basys.json`). `mul_test.c` 200/200 vektor lulus di simulasi SoC dan board 50 MHz (catatan CLAUDE.md, 2026-09-28) |
| `IRQ.v` (timer interrupt) | Modul diinstansiasi di dalam `mriscvcore.v` (`IRQ_inst`), tapi port `outirr` **tidak disambung** di `impl_axi.v` (`//.outirr (irq ),`) | Belum pernah diuji |
| CSR counter (`rdcycle`/`rdinstret`/`rdtime` + versi `H`) | **AKTIF setelah patch `UTILITY.v`** (Bagian 4 #8) | Catatan CLAUDE.md: selisih dua `rdcycle` = 22, sama dengan hitungan simulator. `rdtime` naik tiap 101 siklus (bukan waktu nyata). CSRRW tidak didukung (trap) |
| Division (DIV/DIVU) | Tidak ada modul sama sekali | Test upstream `.disabled` |

**SALAH (klaim lama):** "`MULT.v` tidak dipanggil, dibuktikan DSP48 = 0". Yang benar: `MULT.v` memang tidak direferensikan di `FSM.v`, tapi tersambung di `mriscvcore.v` dan **tersintesis ke LUT/FF**. DSP48 = 0 karena multiplier-nya iteratif, bukan karena tidak tersintesis. Klaim lama ini **sudah dikoreksi di naskah 2026-10-09** (Bab III 3.4.2, Bab IV 4.2 dan 4.5.1).

**Bukti tambahan (dicek 2026-10-09):** `MULT.v` tidak memakai operator perkalian runtime (semua `*` di file adalah `SWORD*2`, `always@*`, atau komentar). Sintesis standalone `synth_xilinx -flatten`: **651 LUT (LUT2-6), 248 FF, 103 CARRY4, 32 MUXF7, 0 DSP48**, setara ~30% LUT konfigurasi 32 KB. `MULT_inst` tercatat di log sintesis lama (`synth_new_run1.txt` 542 referensi, `synth_old_run1.txt` 556) dan di netlist akhir, wiring-nya ada sejak import repo 2026-06-23. **`mul_test.c` diverifikasi ulang di simulasi SoC penuh 2026-10-09: `MUL TEST: 000000C8/000000C8 operasi LULUS`, trap=0, badvec=0** (tb sementara di scratchpad, turunan `impl_axi_uartrx_tb.v` dengan upload 528 word). Instruksi `div`/`rem` tidak dikenali `DECO_INSTR.v` (codif default = ILLISN) -> `FSM.v` masuk `S4_trap`.

**Cara pakai perkalian hardware:** `make build FW=x MARCH=rv32i_zmmul` -> operator `*` di C jadi instruksi `mul`. **JANGAN `-march=rv32im`**: `/` dan `%` jadi instruksi `div`/`rem` yang tidak ada -> trap saat runtime. Dengan `zmmul`, `/` tetap gagal di tahap link (aman). `make build` tidak rebuild otomatis kalau hanya `MARCH` yang berubah - hapus `.elf`/`.bin` dulu.

**Pembagian** tetap tidak ada di hardware, tapi bisa dipakai di firmware lewat rutin software libgcc dengan `-lgcc` (lihat Bagian 11.1B) - konfigurasi linker, bukan keterbatasan arsitektur. Makefile default **tidak** menyertakan `-lgcc`.

---

## BAGIAN 8 - Kondisi Sistem Saat Ini

| Parameter | Nilai awal | **Nilai sekarang** |
|---|---|---|
| Clock core | 1,5625 MHz (`clk100/64`) | **50 MHz** (`clk100/2`) |
| RAM | 4 KB (`A[9:0]`) | **32 KB** (`A[12:0]`) |
| GPIO base | `0x1040 + i*4` | **`0x10040 + i*4`** |
| `link_c.ld` LENGTH | 4K | **32K** |
| `link_c.ld` `.bss` | `. = ALIGN(4);` di luar section | **`.bss : ALIGN(4) {`** (fix 2026-09-28, Bagian 10 #14) |
| Jumlah slave AXI | 5 | **9** (UART TX, UART RX, GPIO bank 2, I2C master) |
| UART | Tidak ada | **TX + RX, 115200 8N1 tetap** |
| I2C | Bit-bang saja | **Bit-bang (GPIO bank 2) ATAU controller hardware 403 kHz**, dipilih saat kompilasi (`SSD1306_HW_I2C`) |
| Reset | `rst_n = por_n & ~btnC` langsung | **Sinkronizer 3-FF di domain `clk`** + `BUS_RST` untuk interconnect & semua slave |
| Indikator trap | Tidak ada | **LED LD15 (pin L1)**, latch sampai CPU di-reset |
| Target P&R | default nextpnr (12 MHz) | **`--freq 50`** (`PNR_FREQ ?= 50` di Makefile) |
| Seed P&R | default internal | **`PNR_SEED ?= 13`** (dikunci dari sweep 20 seed, Bagian 9.5) |
| ISA | RV32I | **RV32I + Zmmul** (opsional lewat `MARCH=rv32i_zmmul`) |

Diverifikasi berfungsi di hardware fisik (SPI upload, ledshow, encoder+OLED, UART TX/RX, OLED lewat I2C hardware - catatan CLAUDE.md). Kalibrasi: `SPEED` di `ledshow.c` x32; `ihold()` OLED/I2C x4. Board SRAM volatile - setelah power cycle wajib `make flash` ulang.

---

## BAGIAN 9 - Data Benchmark Kuantitatif

> **Konteks data 9.1-9.3:** diukur pada RTL LAMA (sebelum UART/I2C, sebelum BUS_RST/sinkronizer reset) dengan target P&R default nextpnr (log: `PASS at 12.00 MHz`). Masih sah sebagai eksperimen pada konfigurasi tersebut (dan dipakai di Bab IV 4.5), tapi **bukan** angka bitstream yang sekarang ada di board. Data desain akhir ada di 9.4-9.6, dan **tidak bisa dibandingkan langsung** dengan 9.1-9.3 karena target P&R berbeda (`--freq 50`).
>
> **Metrik utilisasi yang dipakai naskah:** LUT = jumlah sel LUT1-LUT6 dari `stat` Yosys (TIDAK termasuk sel `INV` dan `RAM32M`); FF = FDRE + FDSE + FDCE + FDRE_1 + FDSE_1. Angka `SLICE_LUTX` dari nextpnr berbeda (lebih besar) dan persentase nextpnr dihitung terhadap die XC7A50T (65200 LUTX, 75 RAMB36, 120 DSP), bukan kapasitas XC7A35T - **jangan kutip persentase nextpnr**.

### 9.1 Eksperimen A - Repeatability (5x per config)
| Config | n | Mean Fmax | Std Dev |
|---|---|---|---|
| RAM 4KB / 1,5625 MHz | 5 | 66,16 MHz | 0,00 |
| RAM 32KB / 50 MHz | 5 | 67,57 MHz | 0,00 |

Std dev = 0 -> `nextpnr-xilinx` pakai seed default tetap, P&R deterministik untuk RTL sama.

### 9.2 Utilisasi resource
| Config | LUT | FF | RAMB36E1 | DSP48 |
|---|---|---|---|---|
| RAM 4KB | 2163 (10,4%) | 1395 (3,35%) | 1 (2,0%) | 0 |
| RAM 32KB | 2181 (10,5%) | 1403 (3,37%) | 8 (16,0%) | 0 |

CARRY4: 225 (4 KB) / 223 (32 KB). Sumber: `stat` Yosys di `results/logs/synth_old_run*.txt` dan `synth_new_run*.txt` (2181 = LUT2..LUT6 dari `synth_new_run1.txt`, dicek 2026-10-09).

~~DSP48=0 di kedua config -> konfirmasi kuantitatif `MULT.v` tidak tersintesis jadi hardware.~~ **SALAH** - DSP48 = 0 karena `MULT.v` iteratif dan dipetakan ke LUT/FF; `MULT.v` aktif (Bagian 7). Standalone `MULT.v` = 651 LUT / 248 FF / 103 CARRY4 / 0 DSP48.

### 9.3 Eksperimen B - Sensitivitas seed (10 seed eksplisit)
| Seed | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Fmax (MHz) | 69,88 | 68,74 | 70,12 | 69,95 | 63,59 | 73,31 | 69,48 | 68,97 | 66,37 | 67,01 |

Mean 68,74 MHz, std dev 2,61 MHz (~3,8%), rentang 63,59-73,31 MHz. Margin ke 50 MHz tetap positif di semua seed (1,27x-1,47x). Placer memang stokastik terhadap seed eksplisit - beda dari eksperimen A yang pakai seed default tetap.

### 9.4 Utilisasi desain akhir (UART + I2C, RAM 32 KB, 50 MHz) - OK diverifikasi 2026-10-09

Sumber: `stat` Yosys di `synth_log0.txt` (root repo, 2026-10-01), dicocokkan identik dengan `read_json basys.json; stat` pada netlist 2026-10-05.

| Konfigurasi | LUT | FF | RAMB36E1 | DSP48 | CARRY4 |
|---|---|---|---|---|---|
| Sebelum periferal (9.2) | 2181 | 1403 | 8 | 0 | 223 |
| **Desain akhir** | **2532** | **1636** | **8** | **0** | **241** |
| Selisih | +351 (+16%) | +233 (+17%) | 0 | 0 | +18 |

Rincian desain akhir: LUT2 648, LUT3 432, LUT4 398, LUT5 698, LUT6 356; FDRE 1563, FDRE_1 45, FDSE 13, FDSE_1 2, FDCE 13; INV 175; RAM32M 8; SRL16E 1. nextpnr: SLICE_LUTX 4276, SLICE_FFX 1636, CARRY4 244. LUT ~12% dari 20.800 LUT XC7A35T.

### 9.5 Fmax desain akhir - OK diverifikasi 2026-10-09 (P&R ulang)

- **Desain akhir, `--freq 50 --seed 13`: Fmax 60,87 MHz (`PASS at 50.00 MHz`)**, margin 1,22x. Laporan pertama (pra-routing) 47,28 MHz FAIL - abaikan, pakai yang kedua. Kebetulan angkanya sama persis dengan build `synth_log_rstsync_freq50.txt` (2026-09-28, desain berbeda) - sudah dicek, memang kebetulan.
- **Sweep 20 seed pada konfigurasi antara** (UART TX/RX + GPIO bank 2, SEBELUM I2C; `results/logs/seed_summary_unified.md`): mean 60,16 MHz, std 3,16, min 55,74, max 66,28 (seed 13), **semua PASS di 50 MHz**. Seed 13 dikunci di Makefile dari sweep ini.
- Setelah I2C ditambahkan, seed 13 turun dari 66,28 ke 60,87 MHz -> seed terbaik tidak bertahan setelah RTL berubah. **Sweep seed desain akhir BELUM dilakukan.**
- Fmax ini bukan sign-off timing penuh - lihat keterbatasan STA di Bagian 13.

### 9.6 Biaya per modul (sintesis terisolasi `synth_xilinx -noiopad`, Yosys 0.38) - OK 2026-10-09

| Modul | LUT (LUT1-6) | FF | CARRY4 | MUXF7/F8 |
|---|---|---|---|---|
| `uart_tx.v` | 41 | 39 | 4 | 0 |
| `uart_rx.v` | 39 | 62 | 4 | 1 |
| `i2c_master.v` | 95 | 100 | 10 | 10 |
| Total | 175 | 201 | 18 | 11 |

Sisa kenaikan desain (~176 LUT, ~32 FF) dari perubahan lain yang tidak diukur terpisah: interconnect 5->9 slave, GPIO bank 2, sinkronizer reset, `BUS_RST`, patch CSR `UTILITY.v`.

### 9.7 Throughput UART (board, catatan CLAUDE.md)

`uart_stress.c`: 3202 baris x 66 byte dalam 20 detik = ~10.570 byte/s, ~92% dari batas teoretis 11.520 byte/s (115200 baud, 10 bit/byte). 0 baris rusak, 0 loncatan counter.

---

## BAGIAN 10 - Bug yang Ditemukan & Solusinya

| # | Bug | Penyebab | Solusi | Cara ditemukan |
|---|---|---|---|---|
| 1 | Booting acak | `SP32B1024` tanpa blok `initial` | Blok inisialisasi nol | Debugging hardware |
| 2 | "Encoder stuck di 00" | Pin `enc_sw` mengambang | Hapus reset berbasis SW + instansiasi pin | Debugging bertahap |
| 3 | Ekspansi RAM tidak berpengaruh | `axi4_interconnect` tanpa override parameter `addr_mask`/`addr_use` (scope berbeda) | Override eksplisit saat instansiasi | Simulasi `iverilog` |
| 4 | Patch `negedge`->`posedge` merusak SRAM | Non-blocking assignment, alamat tertinggal 1 siklus | Revert, pertahankan `negedge` | Simulasi testbench SPI |
| 5 | Tri-state `mriscvcore`/`ALU` merusak write-back (BARU) | Driver-driver conflict di-resolve Yosys jadi konstanta 0 | Patch OR-mux (WAJIB, bukan opsional) | Sintesis penuh + 2x uji hardware |
| 6 | `make synth` gagal cari `openXC7.mk` | Struktur sibling belum ada | `install.sh` | Uji fresh clone |
| 7 | `bbasm` gagal tulis chipdb | Path relatif | `CHIPDB = $(abspath ../chipdb)` | Debugging build |
| 8 | `snap install openxc7` gagal | Tidak ada di Snap Store publik | Unduh `.snap` dari GitHub Releases | Investigasi repo openXC7 |
| 9 | `pypy3: No such file` | Bukan bagian paket snap | `apt install pypy3` | Log error |
| 10 | `main.c` gagal compile | `uint8_t` tanpa `<stdint.h>` | Tambah include | Uji compile |
| 11 | Parameter `.masters/.slaves` tidak di-override saat instansiasi `axi4_interconnect` | Kelas bug SAMA dengan #3: default `masters=2, slaves=5` kebetulan benar sampai slave ditambah | Override eksplisit `.masters(masters), .slaves(slaves)` | Warning lebar port `iverilog` saat `slaves` naik ke 6 |
| 12 | Race `uart_tx`: write kedua menimpa byte pertama | Jendela 1 siklus antara `start_pulse` dan FSM keluar `S_IDLE` | `busy = (tstate != S_IDLE) \|\| start_pulse` | Testbench unit |
| 13 | Race `uart_rx`: byte hilang tanpa flag overrun | Pop meng-clear `valid` 1 siklus SETELAH snapshot RDATA | Pop di siklus yang sama dengan snapshot; byte baru menang atas pop | Gejala di firmware lain, dikunci dengan testbench (12 offset x 8 byte) |
| 14 | Hang "`uart_puts()` > 3 karakter" (tanpa output sama sekali) | **Linker script**: `. = ALIGN(4);` di luar `.bss` -> `__bss_start` ganjil (mis. `0x105` saat `.rodata` 5 byte) -> `sw` misaligned di `crt0.S` -> trap sebelum `main()` | `.bss : ALIGN(4) {` | Uji board: firmware dgn instruksi identik hang dgn linker lama, jalan dgn linker baru; hang juga terjadi di 25 MHz (bukan timing) |
| 15 | Semua upload SPI gagal diam-diam setelah CPU trap | `axi4_interconnect` & slave hanya di-reset `RST` global; interconnect tertahan menunggu handshake CPU yang sudah di-reset | `BUS_RST` = `RST` + pulsa singkat saat CPU masuk reset (`spi_axi_master` TETAP pakai `RST`) | Simulasi `impl_axi_busrst_tb.v` + board |
| 16 | `rdcycle` bikin core macet (tanpa trap) | `UTILITY.v` hanya mencocokkan funct3=000 | Tambah dekode CSRRS/C/WI/SI/CI | Analisis + simulasi SoC |

**Pelajaran metodologis:** bug #3, #4, #5 semuanya **lolos dari inspeksi source manual** dan hanya terdeteksi lewat simulasi/sintesis penuh/uji hardware - menegaskan pentingnya verifikasi berlapis.

**Catatan bug #14 (penting untuk naskah):** sebelum akar masalah ditemukan, hang ini sempat didiagnosis sebagai bug hardware - `REG_FILE.v` read-during-write, counter kecil `axi4_interconnect`, `jal`/`ret` berulang, "ambang iterasi loop 3 vs 4", dan timing marginal. **Semua diagnosis itu SALAH/terbantah** (lihat `~/mriscv-spi/CLAUDE.md` bagian UART). Workaround lama (`always_inline`, `uart_puts` dua fase, larangan string > 3 karakter) sudah dibuang; HAL sekarang pola biasa. Satu-satunya fakta dari masa itu yang tetap berlaku: bug #11, redesain non-blocking + race fix (#12), sifat seed nextpnr, baud 115200. Pelajaran: firmware yang hang TANPA output -> cek `riscv64-unknown-elf-nm <fw>.elf | grep __bss_start` (harus kelipatan 4) sebelum curiga RTL.

---

## BAGIAN 11 - Firmware & Peripheral

### 11.1 Aturan kompilasi bare-metal
```bash
riscv64-unknown-elf-gcc -march=rv32i -mabi=ilp32 -nostdlib -nostartfiles \
    -ffreestanding -Os -T link_c.ld crt0.S <prog>.c -o <prog>.elf
riscv64-unknown-elf-objcopy -O binary <prog>.elf <prog>.bin
```
`crt0.S`, `-nostartfiles`, `-ffreestanding` wajib. Sekarang lewat Makefile: `make build FW=<nama>` (mencari `<nama>.c` otomatis di `firmware/{apps,tests,probes}/`, output ke `build/fw/`), `make prog FW=<nama>` (compile + upload SPI). CFLAGS Makefile: `-march=$(MARCH) -mabi=ilp32 -nostdlib -nostartfiles -ffreestanding -Os -Wall -Ifirmware/include`, `MARCH ?= rv32i`. Linker script tunggal `firmware/startup/link_c.ld` (`.bss : ALIGN(4)` wajib, Bagian 10 #14).

### 11.1B - Pembagian SEBENARNYA bisa dipakai (temuan `-lgcc`)
Klaim lama "`/` dan `%` tidak bisa dipakai" **tidak akurat** - onchipuis sendiri pakai `-nostdlib -lgcc` di `mriscv_vivado/commands.tcl`. **Diverifikasi lewat uji kompilasi langsung:** tanpa `-lgcc` -> gagal (`undefined reference`); dengan `-lgcc` -> berhasil, dan `__mulsi3` terkonfirmasi murni shift-and-add (nol instruksi ekstensi M). Ini **bukan keterbatasan hardware**, melainkan konfigurasi linker yang belum disertakan.

### 11.2 HAL `mriscv.h` - header-only
Header-only = 2135 byte vs split = 2304 byte -> hemat 169 byte. `led_set(v)` loop 8x `gpio_pin()`.

Fungsi HAL per 2026-10-01 (`firmware/include/mriscv.h`):
- GPIO: `gpio_pin/gpio_rd` (bank 1, `0x10040`), `gpio2_pin/gpio2_rd` (bank 2, `0x100C0`), konstanta `GPIO2_ENC_CLK/DT`, `GPIO2_OLED_SCL/SDA`.
- UART TX: `uart_busy()`, `uart_putc()` (polling busy dulu), `uart_puts()` = `while (*s) uart_putc(*s++);` (pola biasa, tanpa batas panjang).
- UART RX: `uart_getc_nb()` (return -1 kalau kosong), `uart_getc()` (blocking), `UART_RXR`.
- I2C hardware: `i2c_hw_init()` (tulis 1 ke pin GPIO bank 2), `i2c_hw_write(b, flags)` (return 1 = ACK), `i2c_hw_read(last, flags)`, `i2c_hw_stop()`. Prefix `i2c_hw_` karena beberapa firmware bit-bang punya `i2c_stop` sendiri.
- CSR: `rdcycle()`, `rdinstret()` (butuh patch `UTILITY.v`).

### 11.3 Library `ssd1306.h` - dual mode + dual I2C
DIRECT (~8 byte RAM) vs BUFFER (+1024 byte). Demo encoder: DIRECT 2623 byte, BUFFER 3735 byte. Font: 91 glyph, kolom-major. Satu `ssd1306_display()` di mode BUFFER = 6 perintah x 3 byte + (alamat + kontrol + 1024 data) = **1044 byte** I2C.

`#define SSD1306_HW_I2C` sebelum `#include` -> memakai `i2c_master.v`; default tetap bit-bang. (Komentar di header masih bilang "belum diuji di board" - sudah usang, menurut catatan CLAUDE.md mode hardware sudah jalan di board 2026-10-01.)

### 11.4 I2C: bit-bang dan hardware
Open-drain wired-AND (sejak 2026-10-01): `assign oled_scl = (gpio2_datanw[0] & i2c_scl_o) ? 1'bz : 1'b0;` (dulu `gdat[0] ? 1'bz : 1'b0`). Init SSD1306 = 25 perintah, krusial: charge pump `0x8D,0x14`.

Firmware uji I2C/UART:
- `firmware/apps/uart_hello.c`, `uart_echo.c`, `oled_hw.c`; `firmware/tests/uart_stress.c`, `i2c_test.c` (untuk simulasi SoC, slave dummy 0x3C), `oled_i2c_stress.c`, `mul_test.c`, `gpio_bank_test.c`; `firmware/apps/oled_stress.c` (bit-bang).
- `oled_i2c_stress.c`: mode BUFFER + HW I2C, ronde berulang (putih/hitam penuh, 16 frame sweep, papan catur, layar status, 512 x 2 transaksi ganti kontras), DIV bergiliran 31/62/125, laporan UART per ronde `R= DIV= F= N= A= S= T= C=`; N (NACK), A (arbitrasi), S (checksum buffer), T (waktu frame via `rdcycle`) harus 0000. **Durasi/jumlah ronde yang diamati di board belum dicatat** (PERLU CEK).

### 11.5 Dekoder kuadratur KY-040
Tepi turun CLK: `DT==1`->CW, `DT==0`->CCW.

---

## BAGIAN 12 - Pemetaan Pin Fisik

| Fungsi | Pin | Pmod |
|---|---|---|
| Clock 100 MHz | `W5` | onboard |
| Reset manual | `U18` | onboard |
| **SPI SCLK** | **`J1`** | JA1 |
| **SPI MOSI** | **`L2`** | JA2 |
| **SPI CS** | **`J2`** | JA3 |
| **SPI MISO** | **`G2`** | JA4 |
| Encoder CLK | `A14` | JB1 |
| Encoder DT | `A16` | JB2 |
| Encoder SW | `B15` | **TIDAK disambungkan** |
| OLED SCL | `K17` | JC1 |
| OLED SDA | `M18` | JC2 |
| UART TX (FPGA -> PC) | `A18` | onboard (FTDI) |
| UART RX (PC -> FPGA) | `B18` | onboard (FTDI) |
| LED trap core | `L1` | LD15 onboard |

Semua pin encoder dan OLED memakai `PULLUP true` di `rtl/basys3_spi.xdc`. `oled_scl`/`oled_sda` sekarang port `inout` (dibaca balik oleh I2C master). Pin UART A18/B18 sesuai master XDC Basys3 (`RsTx`/`RsRx`); belum dicek ke reference manual resmi.

---

## BAGIAN 13 - Catatan Toolchain

- `openxc7` tidak ada di Snap Store publik - install `.snap` dari GitHub Releases.
- `pypy3` wajib, install terpisah via apt.
- **nextpnr cetak Fmax DUA KALI** - laporan kedua (setelah `Router1 time...`) yang valid.
- Warning tri-state Yosys normal (lihat Bagian 5.1), TAPI **"multiple conflicting drivers"/"Driver-driver conflict" TIDAK normal** - itu tanda RTL benar-benar butuh perbaikan (lihat Bagian 5.3).
- `CHECK pass` Yosys yang melaporkan "0 problems" **tidak memeriksa** driver-driver conflict - cakupan berbeda, bukan kontradiksi kalau ada conflict warning di tempat lain.
- `libffi.so.7` hilang -> `fasm2frames` fallback parser Python (lebih lambat, tidak mempengaruhi hasil).
- **`create_clock` di XDC TIDAK dipakai nextpnr-xilinx** - tanpa `--freq`, placer hanya mengejar 12 MHz (log: `PASS at 12.00 MHz`). Sejak 2026-09-28 Makefile selalu menambahkan `--freq $(PNR_FREQ)` (default 50) dan `--seed $(PNR_SEED)` (default 13) lewat `override PNR_ARGS +=`.
- **nextpnr-xilinx deterministik untuk seed yang sama.** Untuk variasi nyata: `make clean` dulu (Make melewati `synth` kalau `.bit` sudah ada), lalu `PNR_ARGS="--seed N"`.
- **Keterbatasan STA nextpnr-xilinx** (catatan CLAUDE.md, dari source `xilinx/arch.cc`): port BRAM (RAMB36) dan port TULIS LUTRAM = `TMG_IGNORE` (jalur BRAM dan penulisan register file tidak dicek); semua FF dimodelkan rising-edge (47 FF negedge `AXI_SP32B1024` dicek seolah punya 1 periode penuh, padahal setengah). **Jadi Fmax bukan sign-off timing penuh** - layak masuk batasan penelitian (belum ada di naskah).
- Yosys menyisipkan BUFG otomatis untuk clock core `divcnt` (clock core lewat jaringan clock global, bukan routing fabric biasa).
- Persentase utilisasi nextpnr relatif ke die XC7A50T, bukan XC7A35T (Bagian 9).

---

## BAGIAN 14 - Perbandingan dengan `mriscv_vivado`

Implementasi FPGA oleh tim pengembang aslinya (kemungkinan Ckristian Duran, penulis [3]).

| Aspek | **Proyek ini** | **mriscv_vivado** |
|---|---|---|
| FPGA | Basys3, XC7A35T | Nexys4 DDR, XC7A100T (3x lebih besar) |
| Toolchain | Open-source penuh | Vivado proprietary |
| Memori | BRAM 32 KB | BRAM + DDR2 eksternal |
| Loading program | **SPI runtime loader** (independen dari bitstream) | `write_cfgmem` (digabung satu file `.mcs`) |
| Peripheral | GPIO (2 bank), UART TX/RX, I2C master hardware, OLED, encoder | +UART, xADC, DAC fungsional, DDR2 |
| Linker | `-nostdlib` saja | `-nostdlib -lgcc` (pembagian jalan) |
| Perkalian | Hardware `MULT.v` iteratif (Zmmul) | - (belum dicek) |

**Poin diferensiasi:** pemisahan bitstream-program (SPI loader) memberi siklus iterasi jauh lebih cepat. **Konvergensi independen:** onchipuis juga menulis ulang modul RAM dari nol untuk port Vivado (template Xilinx `xilinx_single_port_ram_write_first`, mode write-first - beda dari versi read-old-data proyek ini).

---

## BAGIAN 15 - Struktur Repositori

Tiga repositori sejajar: `SlametTurbo/mriscv-spi` (inti, RTL+firmware dasar), `SlametTurbo/mriscv-basys3-peripherals` (peripheral non-game, `VARIAN=switch|encoder|oled`), `SlametTurbo/snake-btn` (game standalone, I2C bit-bang sendiri tanpa `ssd1306.h`).

**Struktur `mriscv-spi` setelah dirapikan (2026-10-01):**
```
rtl/            basys3_top.v (top tunggal), SP32B1024.v, basys3_spi.xdc,
                mriscv/ (RTL upstream; mriscv_axi/ nested di dalam mriscv/,
                         + UART_TX/, UART_RX/, I2C/ baru, testbench SoC di mriscv_axi/impl_axi/)
firmware/       include/ (mriscv.h, ssd1306.h, font5x7_data.h), startup/ (crt0.S, link_c.ld),
                apps/, tests/, probes/ (probe riwayat debugging UART)
tools/          cheetah_mriscv.py (uploader SPI)
results/        logs/ (synth_*.txt, seed summary), seedsweep/
bitstreams/     .bit tersimpan + log
build/fw/       hasil compile firmware (gitignored)
```
Folder `oled-with-encoder/` sudah dihapus (diganti top tunggal). Per 2026-10-09, perubahan I2C (`I2C/`, `impl_axi.v`, `basys3_top.v`, HAL, firmware I2C) **belum di-commit** di repo `mriscv-spi`, dan `CLAUDE.md` sudah dikeluarkan dari git (file masih ada di disk).

---

## BAGIAN 16 - Daftar Pustaka Terverifikasi

**[3]** C. Duran dkk., "A 32-bit RISC-V AXI4-lite bus-based microcontroller with 10-bit SAR ADC," *2016 IEEE 7th LASCAS*, Florianopolis, 2016, hlm. 315-318. DOI: 10.1109/LASCAS.2016.7451073

**[14]** M. Darvishi, "Pipeline-Stage-Resolved Timing Characterization of FPGA and ASIC Implementations of a RISC-V Processor," arXiv:2512.13866. *(preprint, belum terkonfirmasi peer-review; dipakai di Bab III 3.5.2 dan Bab IV 4.5.2 HANYA untuk konteks variabilitas Fmax antar-seed, bukan perbandingan ASIC)*

**[15]** R. Mashimo dkk., "An Open Source FPGA-Optimized Out-of-Order RISC-V Soft Processor," *ICFPT 2019*, IEEE.

**[16]** D. Shah dkk., "Yosys+nextpnr: An Open Source Framework from Verilog to Bitstream for Commercial FPGAs," *FCCM 2019*, IEEE, hlm. 1-4. DOI: 10.1109/FCCM.2019.00028

**[17]** V. V. Solov'ev, "ASMD-FSMD technique in designing signal processing devices on FPGA," *J. Commun. Technol. Electron.*, vol. 66, no. 12, hlm. 1336-1345, 2021.

**[18]** S. L. Harris dan D. Harris, *Digital Design and Computer Architecture: RISC-V Edition*. Morgan Kaufmann, 2021.

---

## BAGIAN 17 - HAL YANG BELUM DIKERJAKAN

| Item | Status |
|---|---|
| UART (`uart_tx.v` + `uart_rx.v`) | **SELESAI** 2026-09-28/30 - permintaan dosen terpenuhi |
| I2C master hardware | **SELESAI** 2026-10-01 |
| Sinyal `trap` -> LED | **SELESAI** (LD15) |
| Aktivasi `MULT.v` | **TIDAK PERLU** - sudah aktif sejak awal (Bagian 7) |
| Verifikasi CPI/RISC-V compliance test | **BELUM** |
| Seven-segment 4 digit | Masih 2 digit (`an = digit_sel ? 4'b1101 : 4'b1110`) |
| Aktivasi `IRQ.v` (sambung `outirr` + simpan/pulihkan PC) | Belum |
| Unit division hardware | Belum |
| Sweep seed pada desain akhir (dengan I2C) | **BELUM** - hanya seed 13 |
| Ulang Eksperimen A/B pada RTL akhir (opsional, supaya data Fmax naskah = bitstream sekarang) | Belum diputuskan |
| Uji ulang `encoder_oled.c` di board setelah jalur I2C jadi wired-AND | **BELUM** (butuh encoder fisik) |
| Durasi/jumlah ronde `oled_i2c_stress` di board | **PERLU CEK** - belum dicatat, ada `% CATATAN` di Bab IV |
| UART: FIFO RX, baud bisa diubah dari firmware | Belum (keterbatasan, sudah ditulis di Bab IV 4.6.6) |
| Commit perubahan I2C di repo `mriscv-spi` | Belum |
| Positive control tri-state: flash ulang RTL versi PATCHED (OR-mux) dengan firmware `test_tristate.c`/`control_test.c` yang sama, verifikasi LED menyala `10101010` | **Direkomendasikan, belum dilaporkan hasilnya** |
| Klarifikasi selisih "5 modul" (uji Yosys-saja) vs "4 modul" (uji `make synth`) di warning driver conflict - apakah `IRQ` genuinely resolve beda tahap atau cuma luput dari pencarian | **Belum diverifikasi tuntas** |

**Detail spesifikasi yang BELUM diverifikasi:** spesifikasi PC Host, batas SCLK maksimum Cheetah, versi persis `iverilog`/`openFPGALoader`, angka resource XC7A35T dari datasheet resmi.

---

## BAGIAN 18 - Prinsip Metodologis yang Dijalankan

1. **Diff langsung ke upstream** - pernah ditemukan patch tercatat "selesai" padahal tidak diterapkan.
2. **Simulasi fungsional** (bukan hanya elaborate-check) - bug #3 dan #4 (Bagian 10) lolos inspeksi manual, hanya terdeteksi simulasi.
3. **Hipotesis yang terbukti salah tetap didokumentasikan** - kasus `negedge`: asumsi "bug ASIC" ternyata keliru.
4. **Eksperimen terkontrol dengan variabel eksplisit** - repeatability vs sensitivitas seed dipisah.
5. **Data dari log sintesis aktual** - Fmax dicetak dua kali, pakai yang kedua.
6. **Sintesis Yosys-saja tidak selalu representasi lengkap** - uji `stat`-only (32 warning "multiple conflicting drivers") dan sintesis penuh via `make synth` (128 warning "Driver-driver conflict") menunjukkan pesan/detail berbeda meski RTL sama. Untuk klaim final, gunakan alur build **sesungguhnya** (`make synth`), bukan potongan tahap saja.
7. **Uji kontrol perlu dianalisis arsitekturnya dulu, bukan diasumsikan valid** - firmware kontrol yang dirancang untuk "menghindari jalur bermasalah" (`led_set(0xAA)` tanpa `lw`) ternyata **tetap melewati bus yang sama** (write-back ALU lewat `rd`), sehingga tidak benar-benar mengisolasi variabel seperti yang diniatkan. Ditemukan lewat analisis log, bukan lewat uji tambahan.
8. **Uji hardware dengan firmware di-bake ke bitstream** (`INIT_FILE`/`$readmemh`) - bypass SPI loader untuk iterasi lebih cepat, diverifikasi bekerja lewat dekode parameter langsung dari netlist.
9. **Verifikasi tiga tingkat untuk periferal baru** - testbench unit (termasuk kondisi error) -> simulasi SoC penuh yang menjalankan firmware hasil kompilasi -> uji board. Dua race condition (Bagian 10 #12, #13) tertangkap di tingkat testbench, sebelum sampai board.
10. **Gejala "seperti bug hardware" bisa berasal dari perangkat lunak pendukung** - kasus `.bss` (Bagian 10 #14) sempat lama didiagnosis sebagai bug `REG_FILE`/interconnect/timing. Variabel diisolasi dengan firmware yang instruksinya identik dan hanya beda data, plus uji di clock lebih rendah.
11. **Observability dulu** - tanpa LED trap, "core trap" dan "bus macet" tampak identik. Indikator murah ini mempercepat diagnosis.
12. **Sweep seed perlu diulang setiap RTL berubah** - seed terbaik (13: 66,28 MHz) turun ke 60,87 MHz setelah I2C ditambahkan.

**Penggunaan AI:** proses dibantu asisten AI (Claude Code) untuk eksekusi kode dan analisis; keputusan desain dan validasi tetap oleh penulis. Beberapa rekomendasi AI terbukti salah (kasus `negedge`; desain uji kontrol yang cacat di 5.3; diagnosis "bug hardware" `REG_FILE`/`jal`-`ret`/ambang iterasi loop pada kasus `.bss` di Bagian 10 #14; klaim "`MULT.v` tidak dipanggil" di Bagian 7) dan dikoreksi berdasarkan bukti - validasi independen konsisten dijalankan, bukan penerimaan pasif.

---

## BAGIAN 19 - Keputusan Struktur Bab III (untuk konsistensi prompt selanjutnya)

> Dua butir pertama di bawah adalah catatan keputusan dari draf lama; penomoran 3.2.1/3.5.1-nya tidak ada di `.tex` saat ini (lihat catatan di awal dokumen). Struktur aktual: subbagian "Struktur naskah per 2026-10-09".

- **ASIC dihapus total dari Bab III.** Sub-bab 3.2.1 sekarang berjudul "Kerangka Konseptual: Audit dan Modifikasi RTL" - bicara soal karakteristik RTL (tri-state, latch, negedge) dan `mriscv_vivado` sebagai pembanding toolchain (Vivado vs Yosys), tanpa narasi asal-usul ASIC. Sub-bab lama "Karakteristik Penundaan Jalur pada FPGA dan ASIC" **dihapus seluruhnya** (bukan direname).
- **3.5.1 disederhanakan jadi prosedural murni** - 5 paragraf pendek, tanpa cuplikan kode, tanpa narasi hipotesis-diuji-gagal, tanpa penjelasan mekanisme TRIBUF. Kalimat penutup paragraf audit menunjuk eksplisit ke Bab IV untuk detail verifikasi.
- **Materi yang "dipindah" ke Bab IV** (bukan dihapus, tinggal disusun): cuplikan kode before/after tiap patch, penjelasan TRIBUF, kasus `negedge` lengkap, **seluruh temuan Bagian 5.3 (driver-driver conflict + uji hardware)**, tabel audit tingkat-keperluan (Bagian 4), data benchmark (Bagian 9), perbandingan `mriscv_vivado` (Bagian 14).
- ~~Penomoran tabel Bab III final: 3.1-3.7 (Perangkat Keras, Perangkat Lunak, Bahan, Peta Alamat AXI, Pemetaan Pin, Skenario Pengujian, Hambatan Teknis).~~ **Usang.** Isi `.tex` per 2026-10-09 hanya punya 2 tabel di Bab III: **3.1 Peta Alamat Bus AXI4-Lite** (9 slave) dan **3.2 Peta Register Modul UART dan I2C**. Alat/bahan ditulis sebagai daftar, bukan tabel.

### Struktur naskah per 2026-10-09 (setelah UART/I2C ditambahkan)

**Bab III:** 3.1 Alat dan Bahan (+ kanal UART FTDI, `iverilog`, terminal serial) - 3.2 Metode - 3.3 Alur (item 2 menyebut penambahan 3 modul periferal, item 4 menyebut uji UART dan I2C dua cara) - 3.4 Struktur Hierarki: 3.4.1 Hierarki (dirtree 9 slave + **diagram blok TikZ `fig:hierarki-rtl`** di bawahnya: `impl_axi` > `mriscvcore` + 8 modul fungsional, `spi_axi_master`, `axi4_interconnect`, S0-S8, perangkat eksternal, legenda modul baru/tidak digunakan, nama modul tanpa `.v`), 3.4.2 Struktur Internal mriscvcore, 3.4.3 Bus AXI4-Lite dan Periferal (Tabel 3.1, GPIO 2 bank), **3.4.4 Rancangan Modul Periferal UART dan I2C** (pola non-blocking, spesifikasi TX/RX/I2C, wired-AND, Tabel 3.2, verifikasi tiga tingkat) - 3.5 Metode Analisis: 3.5.1 Fungsional (**lima** sasaran: GPIO/LED, 7-seg, encoder, OLED bit-bang & HW I2C, UART), 3.5.2 Kuantitatif (+ sintesis terisolasi per modul dan pembandingan sebelum/sesudah).

**Bab IV:** 4.1 Sintesis - 4.2 Pemuatan Program dan Firmware (I2C dua mode) - 4.3 Hasil Fungsional (+ butir UART dan I2C hardware) - 4.4 Analisis RTL (Tabel 4.1, 6 modifikasi) - 4.5 Fmax dan Utilisasi (Eksperimen A/B, RTL lama) - **4.6 Penambahan Periferal UART dan Pengendali I2C**: 4.6.1 Integrasi ke Bus (bug #11), 4.6.2 Implementasi UART (race #12, #13, HAL), 4.6.3 Implementasi I2C (wired-AND, 403,2 kHz, ~79% aktif), 4.6.4 Hasil Verifikasi (Tabel 4.5), 4.6.5 Temuan `.bss` (bug #14, #15, LED trap), 4.6.6 Utilisasi dan Fmax (Tabel 4.6 per modul, Tabel 4.7 sebelum/sesudah, seed 13 = 60,87 MHz, keterbatasan) - 4.7 Perbandingan (baris periferal diperbarui).

### Aturan gaya istilah di naskah (keputusan penulis 2026-10-09)

1. **Istilah yang merujuk ke sesuatu yang teknis memakai istilah bahasa Inggris** dalam `\textit{...}`, tanpa terjemahan dan tanpa pola "terjemahan (\textit{English})". Tidak ada lagi daftar pengecualian: *hardware*/*software*, *interface*, *resource utilization*, *clock cycle*, *address*, *file*, *interconnect bus*, *maximum operating frequency*, *gate-level netlist*, *program loading*, *asynchronous*/*synchronous*, *open-source*, *circuit*, *line*/*path*, *device*, *control unit*, *state*, *address map*, *register map*, *timing analysis*, *safety margin*, *repeatability*, *clock domain*, *falling edge*, *transmitter*/*receiver*, *controller*, *arbitration lost*. Kata serapan yang ejaannya sudah diindonesiakan (sintesis, simulasi, frekuensi, memori, register, modul, sinyal, utilisasi, konfigurasi) tetap dipakai.
2. **Tanpa titik koma (;) dan tanpa em-dash/tanda pisah** sebagai tanda baca kalimat (termasuk `--` di tengah kalimat dan di judul subbab, diganti titik dua/koma/kalimat baru). Yang tetap boleh: rentang angka/alamat/subbab (`0x0000--0x1FFF`, `4.4.1--4.4.7`), sintaks di dalam listing kode, dan output program yang dikutip apa adanya di `verbatim`.

Bab III dan IV sudah diseragamkan dengan kedua aturan ini pada 2026-10-09 (diverifikasi dari teks PDF). Judul skripsi di `main.tex` masih placeholder, jadi "sumber terbuka" tidak lagi jadi pengecualian. Teks di blok yang di-comment belum ikut diseragamkan.

### Kesalahan di naskah yang BELUM diperbaiki (per 2026-10-09)

| Lokasi | Masalah | Koreksi |
|---|---|---|
| ~~Bab III 3.4.2~~ | ~~"`MULT.v` ... dan `IRQ.v` ... tidak dipakai"~~ | **SUDAH DIPERBAIKI 2026-10-09**: `MULT.v` dijelaskan sebagai Zmmul aktif, `IRQ.v` tidak dipakai karena `outirr` tidak disambung |
| ~~Bab IV 4.5.1~~ | ~~"DSP48 bernilai 0 ... `MULT.v` tidak pernah tersintesis"~~ | **SUDAH DIPERBAIKI 2026-10-09**: penjelasan Booth iteratif + angka standalone 651 LUT/248 FF. Verifikasi `mul_test` (200/200, simulasi + board) ditambahkan di Bab IV 4.2 |
| Bab IV Tabel 4.1 | Hanya 6 modifikasi | Belum memuat `UTILITY.v` (patch CSR) dan `impl_axi.v` (Bagian 4 #8, #9) |
| ~~Bab III 3.1.1 butir 1~~ | ~~"16 tombol, 5 saklar"~~ | **SUDAH DIPERBAIKI 2026-10-09** saat penyeragaman istilah: "16 *switch*, 5 *push button*, *seven-segment display* 4 digit, 16 LED" |
| Bab III (blok keterbatasan yang di-comment) | "Sinyal `trap` belum disambungkan ke LED" | Sudah tidak berlaku (butir `MULT.v` di blok ini sudah dikoreksi 2026-10-09) |
| Bab IV 4.6.3 | Desain I2C mengacu OpenCores (Herveille) dan `verilog-i2c` (Forencich) | Belum disitasi |
| Bab IV/batasan | Keterbatasan STA nextpnr-xilinx (Bagian 13) | Belum disebut sebagai batasan penelitian |
| `thesisdtetiugm.cls` baris 66 | `\usepackage[figurewithin=none]{caption}` (bawaan template) membuat counter gambar tidak di-reset per bab, sehingga gambar pertama Bab III bernomor 3.2 karena Bab II punya 1 gambar contoh template | Belum diubah (menunggu keputusan penulis). Akan hilang sendiri kalau gambar contoh Bab II dihapus, tapi bug muncul lagi begitu Bab II punya gambar sungguhan |
