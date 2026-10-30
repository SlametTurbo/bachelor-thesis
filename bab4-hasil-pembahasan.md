# BAB IV HASIL DAN PEMBAHASAN

Penelitian ini menghasilkan implementasi *soft-core* RISC-V RV32I mriscv yang berfungsi penuh pada *board* FPGA Digilent Basys3, dibangun sepenuhnya melalui *toolchain* sintesis sumber terbuka (Yosys, nextpnr-xilinx, openXC7) tanpa Vivado. Pengujian dilakukan bertahap: persiapan lingkungan pengembangan, audit dan modifikasi RTL, pengembangan *firmware bare-metal*, verifikasi fungsional pada perangkat keras fisik (GPIO, *seven-segment*, layar OLED SSD1306, *rotary encoder* KY-040), hingga karakterisasi kuantitatif hasil sintesis (frekuensi kerja maksimum dan utilisasi sumber daya). Temuan paling signifikan dari seluruh proses ini adalah bug tiga-keadaan (*tri-state*) pada jalur *write-back* register `mriscvcore.v`/`ALU.v`, yang terbukti lewat sintesis penuh dan dua pengujian perangkat keras independen menyebabkan seluruh instruksi penulis register — bukan hanya `lw` — menghasilkan nilai nol; temuan ini diuraikan secara rinci pada Subbab 4.5.

## 4.1 Persiapan Lingkungan Pengembangan

Lingkungan pengembangan dibangun di atas Linux Ubuntu 24.04 LTS. Komponen *toolchain* yang dipasang meliputi Yosys (sintesis RTL), nextpnr-xilinx (*place-and-route*), openXC7 (paket `fasm2frames`/`xc7frames2bit`/`bit2fasm` untuk pembuatan *bitstream*), riscv64-unknown-elf-gcc v13.2.0 (*cross-compiler* *firmware*), dan Icarus Verilog (`iverilog`) untuk simulasi fungsional sebelum sintesis penuh dijalankan.

Paket `openxc7` tidak tersedia di Snap Store publik, sehingga tidak dapat dipasang lewat `sudo snap install openxc7` secara langsung. Sebagai gantinya, berkas `.snap` diunduh langsung dari GitHub Releases resmi proyek (`openXC7/yosys-snap` dan `openXC7/openXC7-snap`) dan dipasang secara lokal menggunakan `snap install --classic --dangerous`. `pypy3` dipasang terpisah lewat `apt`, karena dibutuhkan oleh `bbaexport.py` untuk membangkitkan *chip database* (`chipdb`) namun tidak disertakan dalam paket `.snap`. `Makefile` proyek juga mengasumsikan struktur direktori *sibling*, yaitu `openXC7.mk` dan `chipdb/` berada satu level di atas *repository* kerja, bukan di dalamnya.

Selama tahap ini, ditemukan lima hambatan instalasi yang terverifikasi langsung di lingkungan kerja:

- **`make synth` gagal mencari `openXC7.mk`** — penyebabnya struktur direktori *sibling* belum tersedia; solusinya membuat skrip `install.sh` yang menyalin `openXC7.mk` dan menyiapkan direktori `chipdb/` kosong satu level di atas *repository* secara otomatis.
- **`bbasm` gagal menulis `chipdb`** — penyebabnya variabel `CHIPDB` didefinisikan sebagai *path* relatif; solusinya mengubahnya menjadi *path absolut* (`CHIPDB = $(abspath ../chipdb)`).
- **`snap install openxc7` gagal** — penyebabnya paket tidak terdaftar di Snap Store publik; solusinya mengunduh berkas `.snap` langsung dari GitHub Releases proyek `openXC7`, sebagaimana diuraikan di atas.
- **`pypy3: No such file or directory`** — penyebabnya `pypy3` bukan bagian dari paket `.snap`; solusinya memasangnya terpisah lewat `apt install pypy3`.
- **`main.c` gagal dikompilasi** — penyebabnya tipe `uint8_t` dipakai tanpa `#include <stdint.h>`; solusinya menambahkan berkas *header* tersebut.

## 4.2 Implementasi *Soft-core* mriscv pada FPGA

Audit RTL dilakukan dengan membandingkan (*diff*) setiap berkas pada *repository* kerja terhadap rilis sumber terbuka `onchipuis/mriscv`, dipakai secara retrospektif untuk mengatalogkan status akhir tujuh modifikasi yang teridentifikasi selama pengembangan — metode penemuan tiap modifikasi yang sesungguhnya bervariasi per kasus dan diuraikan pada Subbab 4.5. Ringkasan lengkap ketujuh modifikasi tersebut disajikan pada Tabel 4.1 (Subbab 4.5.d). Tiga di antaranya diuraikan berikut sebagai patch kunci.

Modul `SP32B1024.v` ditulis dari nol dalam bentuk RTL *behavioral* yang dipetakan ke *Block RAM* (BRAM) FPGA, karena modul dengan nama tersebut tidak tersedia pada rilis sumber terbuka. Modul ini menyertakan blok inisialisasi nilai awal:

```verilog
integer i;
initial begin
    if (INIT_FILE != "") $readmemh(INIT_FILE, mem);
    else for (i = 0; i < 8192; i = i + 1) mem[i] = 32'h0;
end
```

Tanpa blok ini, isi BRAM tidak terdefinisi setelah konfigurasi Artix-7, menyebabkan kegagalan *booting* acak.

Pada `axi4_interconnect.v`, dua blok kombinasional dengan *sensitivity list* eksplisit — yang menghasilkan struktur *latch* — diubah menjadi struktur sekuensial bertepi *clock*:

```verilog
// upstream (latch):
always @(m_axi_awvalid[unpk_idx] or RST or m_axi_awaddr[...]) begin
    if (RST == 1'b0) m_axi_awaddr_g[unpk_idx] = {sword{1'b0}};
    else if (m_axi_awvalid[unpk_idx]) m_axi_awaddr_g[unpk_idx] = m_axi_awaddr[...];
end

// setelah dipatch:
always @(posedge CLK) begin
    if (RST == 1'b0) m_axi_awaddr_g[unpk_idx] = {sword{1'b0}};
    else if (m_axi_awvalid[unpk_idx]) m_axi_awaddr_g[unpk_idx] = m_axi_awaddr[...];
end
```

Pada `mriscvcore.v` dan `ALU.v`, jalur *write-back* register yang semula berbentuk bus tiga-keadaan (*tri-state*) diubah menjadi struktur *multiplexer* berbasis operasi OR:

```verilog
// mriscvcore.v — setelah dipatch:
wire [31:0] rd_alu, rd_mult, rd_util, rd_mem, rd_irq;
assign rd = rd_alu | rd_mult | rd_util | rd_mem | rd_irq;

// ALU.v — sebelum: assign rd = is_rd_nr ? OUT_Alu_rd : 32'bz;
// ALU.v — setelah:  assign rd = is_rd_nr ? OUT_Alu_rd : 32'b0;
```

Sebaliknya, konstruksi `always @(negedge CLK)` pada `AXI_SP32B1024.v` sengaja **dipertahankan**, tidak dipatch:

```verilog
always @(negedge CLK) begin
    // A/CEN/WEN sengaja di-latch setengah-siklus setelah FSM AXI (posedge) settle
end
```

Alasan dan bukti empiris keputusan ini diuraikan pada Subbab 4.5.b.

Alur pembuatan *bitstream* dari RTL yang telah dimodifikasi berjalan sebagai berikut: Yosys mensintesis RTL menjadi *netlist* gerbang logika Xilinx Artix-7 (`synth_xilinx`) → nextpnr-xilinx melakukan *place-and-route* ke target XC7A35T-1CPG236C mengikuti kendala fisik `basys3_spi.xdc` → `fasm2frames` mengonversi hasil PnR menjadi *frame* konfigurasi (FASM) → `xc7frames2bit` mengubah *frame* tersebut menjadi berkas *bitstream* (`spi.bit`) yang diprogramkan ke FPGA lewat openFPGALoader melalui antarmuka JTAG.

Parameter `INIT_FILE` pada `SP32B1024.v` mendukung dua mode operasi. **Mode operasional** (`INIT_FILE=""`, digunakan pada seluruh pengujian fungsional di Subbab 4.4): BRAM diinisialisasi kosong (nol) saat konfigurasi FPGA, dan program dimuat setelahnya secara *runtime* lewat antarmuka SPI (Subbab 4.3) — memisahkan sepenuhnya siklus sintesis dari siklus pemrograman. **Mode uji verifikasi** (`INIT_FILE` menunjuk ke berkas `.hex`, dipakai khusus untuk uji hardware isolasi *tri-state* di Subbab 4.5.c): isi BRAM langsung ditentukan oleh `$readmemh` saat sintesis, sehingga *firmware* uji sudah tertanam di *bitstream* begitu FPGA dikonfigurasi, tanpa bergantung pada SPI *loader* — mempercepat siklus iterasi debugging karena mekanisme pemuatan program dieliminasi sebagai variabel.

## 4.3 Mekanisme Pemuatan Program dan Pengembangan *Firmware*

Program dimuat ke memori *soft-core* lewat *host* Python `cheetah_mriscv.py` yang mengendalikan Total Phase Cheetah SPI Adapter. Protokol yang dipakai mengirim *frame* 66-*bit* `{instr[1:0], addr[31:0], data[31:0]}`, dikemas menjadi 9 *byte* (72 *bit*, 6 *bit* padding di akhir yang dibuang perangkat keras saat CS naik), pada SPI *mode* 0 (CPOL=0, CPHA=0), MSB-*first*. Sinyal *chip-select* (CS) dipakai standar per-*frame* (aktif rendah selama transfer, sesuai konvensi SPI), sedangkan status *reset*/*release* `soft-core` (`PICORV_RST`) **tidak** dikendalikan lewat pin *reset* fisik terpisah, melainkan lewat *bit* data pada *frame* NOP yang dikirim melalui jalur data (MOSI): *frame* NOP dengan `data=0` menahan *core* dalam kondisi *reset* selama proses penulisan program berlangsung, dan *frame* NOP dengan `data=1` di akhir upload melepas *core* untuk mulai mengeksekusi program. Kecepatan SCLK dibatasi ≤ ¼ *clock core*; pada *clock core* 50 MHz (batas ~12,5 MHz) dipakai SCLK 100 kHz, jauh di bawah batas tersebut.

Pengembangan *firmware bare-metal* menggunakan pola HAL *header-only* pada `mriscv.h` (fungsi `static inline`), yang secara terukur menghemat 169 *byte* dibandingkan pola *header*/*source* terpisah (2135 *byte* vs 2304 *byte* untuk program uji yang sama). Pustaka `ssd1306.h` untuk layar OLED mendukung dua mode lewat *macro* `SSD1306_USE_BUFFER`: mode DIRECT (baku, tiap operasi gambar langsung dikirim ke I2C, `display()` menjadi *no-op*, ~8 *byte* RAM) dan mode BUFFER (`display()` wajib dipanggil, +1024 *byte* RAM untuk *buffer* piksel). Antarmuka I2C ke OLED diimplementasikan penuh secara *bit-bang* perangkat lunak (tidak ada periferal I2C perangkat keras), dengan urutan inisialisasi SSD1306 25 perintah, termasuk `0x8D,0x14` (aktivasi *charge pump*) yang krusial agar layar tidak kosong. Dekoder *rotary encoder* KY-040 membaca arah putaran pada tepi turun sinyal CLK: `DT==1` diinterpretasikan sebagai putaran searah jarum jam (CW), `DT==0` sebagai berlawanan arah jarum jam (CCW).

Klaim awal bahwa operator pembagian (`/`, `%`) tidak dapat dipakai pada *firmware* terbukti tidak akurat. Dengan menambahkan `-lgcc` pada perintah kompilasi (`-nostdlib -lgcc`, bukan `-nostdlib` saja), operasi pembagian dapat dikompilasi dan dijalankan; verifikasi kompilasi langsung menunjukkan rutin `__mulsi3` yang dipanggil murni berbasis *shift-and-add* perangkat lunak (bukan ekstensi instruksi perkalian perangkat keras). Dengan demikian, ketiadaan dukungan pembagian/perkalian pada *firmware* sebelumnya merupakan persoalan konfigurasi *linker* yang belum disertakan, bukan keterbatasan arsitektur perangkat keras.

## 4.4 Keberhasilan Sintesis dan Eksekusi pada FPGA

Proses sintesis pada konfigurasi operasional (RAM 32 KB, *clock core* 50 MHz) berhasil menghasilkan berkas *bitstream* (`spi.bit`) tanpa *error*, dan berhasil diprogramkan ke *board* Digilent Basys3 melalui openFPGALoader. *Firmware* diunggah lewat `cheetah_mriscv.py`, dengan status keluaran program yang khas sebagai berikut:

```
Cheetah terbuka (port 0), SCLK = 100 kHz, mode 0, MSB-first.
Upload 42 kata dari ledshow.hex ...
Selesai. Core dilepas; program berjalan.
```

Seluruh sasaran pengujian fungsional berhasil dijalankan pada perangkat keras fisik:

- **GPIO/LED** — `ledshow.c` dan `switch_led.c` berhasil dijalankan; pola nyala LED sesuai dengan program yang diunggah.
- ***Seven-segment*** — `sevensegment.c` dan `encoder_7seg.c` berhasil menampilkan digit; saat ini terbatas pada 2 dari 4 digit yang aktif, karena *multiplexing* 4 digit belum diimplementasikan pada RTL *top module* (`an = digsel ? 4'b1101 : 4'b1110`).
- ***Rotary encoder* KY-040** — arah putaran (CW/CCW) yang terbaca `encoder_test.c`/`encoder_7seg.c` sesuai dengan arah putaran fisik yang dilakukan pada perangkat.
- **Layar OLED SSD1306** — inisialisasi dan tampilan berhasil diverifikasi lewat *firmware* pada pengujian encoder+OLED, dengan kalibrasi `ihold()` dinaikkan 4× mengikuti kenaikan *clock* domain I2C *bit-bang* dari 12,5 MHz ke 50 MHz.

Keseluruhan pengujian ini konsisten dengan kriteria analisis fungsional yang ditetapkan pada Subbab Metode Analisis Hasil Akhir, Bab III.

## 4.5 Analisis Hasil Sintesis dan Verifikasi RTL

Keberhasilan yang dilaporkan pada Subbab 4.4 dicapai menggunakan RTL final yang telah melalui serangkaian keputusan modifikasi. Subbab ini membedah proses verifikasi di balik tiap keputusan tersebut.

### 4.5.a *Tri-state* pada `axi4_interconnect.v` — Ditangani Otomatis oleh Yosys (TRIBUF)

Berbeda dari modifikasi *latch* pada modul yang sama yang telah diuraikan pada Subbab 4.2, `axi4_interconnect.v` juga mengandung konstruksi *tri-state* yang terpisah. `axi4_interconnect.v` tetap mengandung konstruksi *tri-state* (`{sword{1'bz}}`) tanpa dipatch, dengan parameter *default* `addressing=0` (mode *tri-state*) yang tidak di-*override* di manapun. Verifikasi lewat dua kali sintesis independen (Yosys 0.33 dan 0.38) menunjukkan hasil identik: log sintesis mencatat tahap `27.7. Executing TRIBUF pass.` dieksekusi, dan **nol** primitif TBUF muncul pada *netlist* akhir, tanpa *error*. Warning `"Yosys has only limited support for tri-state logic"` yang tercatat pada modul ini bersifat normal, bukan indikasi kegagalan.

Temuan ini menunjukkan Yosys (`synth_xilinx`) memiliki *pass* internal TRIBUF yang secara otomatis mengonversi pola *tri-state* **lokal sederhana** (satu `assign` per *net* dengan operator ternary) menjadi *multiplexer*, tanpa memerlukan RTL ditulis ulang manual — berbeda dengan kasus pada `mriscvcore.v`/`ALU.v` (Subbab 4.5.c), yang melibatkan pola *tri-state* **lintas-modul** yang tidak tertangani otomatis oleh *pass* yang sama.

### 4.5.b *Negedge* pada `AXI_SP32B1024.v` — Disengaja, Patch Naif Terbukti Merusak

Kode upstream menyertakan komentar pengembang asli yang menjelaskan alasan penggunaan `negedge`: *"we provide the signals in negedge, because the setup and hold sh*t"*. Hipotesis awal menduga pola ini adalah praktik desain lama yang perlu dikonversi ke `posedge` agar seluruhnya bertepi naik.

Pengujian empiris lewat simulasi fungsional `iverilog` dengan *testbench* protokol SPI nyata membuktikan hipotesis tersebut **keliru**: mengganti `negedge` menjadi `posedge` pada kedua baris terkait merusak fungsi tulis SRAM secara total. Penyebabnya adalah semantik *non-blocking assignment* — setelah konversi, `SP32B1024` membaca alamat `A` yang tertinggal satu siklus penuh, karena hubungan waktu setengah-siklus yang sebelumnya sengaja dijaga antara *state machine* AXI (bertepi naik) dan modul memori (bertepi naik murni) menjadi hilang.

Berdasarkan bukti ini, patch `negedge`→`posedge` **dibatalkan**, dan konstruksi `negedge` dipertahankan sebagaimana adanya. Konfirmasi *netlist* menunjukkan 47 *flip-flop* bertepi negatif (`FDRE_1`/`FDSE_1`) tetap ada pada hasil sintesis akhir, sesuai desain yang dipertahankan.

### 4.5.c *Tri-state* pada `mriscvcore.v`/`ALU.v` — Temuan Utama: Terbukti Merusak Seluruh Jalur *Write-back*

Investigasi ini mengoreksi status modifikasi #4 (Tabel 4.1) dari semula "opsional" menjadi **wajib**, berdasarkan bukti sintesis penuh dan dua pengujian perangkat keras independen.

**Langkah 1 — Sintesis Yosys-saja (mode *stat*):** `mriscvcore.v`/`ALU.v` dikembalikan ke versi *upstream* (tri-state asli), modul lain tetap dalam kondisi terpatch. Hasil: proses selesai tanpa *error* fatal, namun log mencatat **32 warning** *"multiple conflicting drivers"* — satu per *bit* dari *bus* `rd` 32-*bit*, dengan lima modul (`ALU`, `IRQ`, `MEMORY_INTERFACE`, `MULT`, `UTILITY`) tercatat sebagai *driver* yang saling bentrok pada *net* yang sama. Total sel hasil sintesis 1689–1691 (dibandingkan 3996 pada versi terpatch), mengindikasikan banyak logika yang gugur karena resolusi konflik.

**Langkah 2 — Sintesis penuh via `make synth`:** Warning berubah nama menjadi *"Driver-driver conflict"*, dengan total **128 warning** (32 *bit* × 4 modul: `ALU`, `MEMORY_INTERFACE`, `MULT`, `UTILITY` — `IRQ` tidak muncul pada pencarian ini, kemungkinan disebabkan tahap *pipeline* sintesis yang berbeda, belum diverifikasi tuntas). Cuplikan aktual dari log sintesis:

```
Warning: Driver-driver conflict for \u_impl.mriscvcore_inst.ALU_inst.OUT_Alu_rd [31]
between cell $flatten...ALU_inst.$procdff$6064.Q and constant 1'0 in basys3_top:
Resolved using constant.
```

Seluruh 128 konflik (128/128) diselesaikan Yosys dengan cara yang identik: **`Resolved using constant`**, bernilai `1'0`. Artinya *net* `rd` pada *netlist* akhir pada dasarnya **selalu bernilai nol**, apa pun instruksi yang dieksekusi.

**Kenapa ini merusak seluruh eksekusi, bukan hanya `lw`:** Pada `mriscvcore.v`, `rd` bukan sekadar jalur hasil `MEMORY_INTERFACE` (operasi *load*) — ia adalah **satu-satunya *bus write-back*** menuju `REG_FILE`, dipakai bersama oleh seluruh modul yang menghasilkan nilai untuk ditulis ke register (`ALU`, `MEMORY_INTERFACE`, `MULT`, `IRQ`, `UTILITY`). Karena `rd` di-*hardwire* ke nol, **setiap instruksi yang menulis register — termasuk operasi aritmetika biasa seperti `add`/`addi`/`li` — juga menghasilkan nilai nol**, bukan hanya instruksi `lw`.

**Uji perangkat keras, dua *firmware* independen, program di-*bake* langsung ke *bitstream* lewat `INIT_FILE`:**

*Firmware* 1 (uji tulis-baca RAM): `sw` nilai `0xAA` ke alamat RAM `0x400`, `lw` kembali, tampilkan hasilnya ke LED lewat `led_set()`. **Hasil: seluruh LED mati** (ekspektasi: pola `10101010`).

*Firmware* 2 (`test_tristate.c`, uji kontrol — murni tulis, tanpa `lw` sama sekali):

```c
#include "mriscv.h"
int main(void) {
    led_set(0xAA);   /* langsung tulis konstanta ke LED, TANPA sw+lw RAM */
    for(;;){}
}
```

**Hasil: seluruh LED mati juga.** *Firmware* ini semula dirancang untuk mengisolasi apakah kegagalan spesifik pada jalur `lw`. Namun karena instruksi `li`/`addi` (dipakai `led_set()`) **juga** melewati *bus* `rd` yang sama untuk *write-back* ALU, desain uji kontrol ini tidak benar-benar mengisolasi variabel seperti yang dimaksudkan — kelemahan yang ditemukan lewat analisis log sintesis, bukan lewat uji perangkat keras tambahan.

**Verifikasi mekanisme `INIT_FILE` bekerja (menyingkirkan kemungkinan alternatif):** Parameter `INIT_FILE` dibaca langsung dari *netlist* hasil sintesis dalam bentuk 136-*bit* biner (`136'0111010001100101...`), didekode manual 8-*bit* per karakter, menghasilkan persis string `"test_tristate.hex"`. Hasil ini membuktikan mekanisme pemuatan *firmware* langsung ke *bitstream* (tanpa SPI *loader*) bekerja dengan benar — kegagalan LED pada kedua *firmware* uji **bukan** disebabkan kegagalan pemuatan program, melainkan murni akibat bug *tri-state* pada jalur *write-back*.

**Kesimpulan:** Modifikasi #4 pada Tabel 4.1 (`mriscvcore.v`/`ALU.v`, *tri-state* → OR-*mux*) bersifat **wajib**, bukan opsional. Berbeda dari kasus `axi4_interconnect.v` (Subbab 4.5.a), di mana *pass* TRIBUF menangani *tri-state* otomatis dengan sempurna, pola *tri-state* **lintas-modul** (banyak instansiasi modul terpisah berbagi satu *wire* lewat *port*, bukan satu `assign` lokal) tidak tertangani oleh *pass* yang sama — Yosys justru menyelesaikannya secara diam-diam menjadi konstanta nol, sebuah kegagalan senyap yang lolos dari `CHECK pass` Yosys (yang melaporkan "0 problems", karena `CHECK pass` memang tidak memeriksa *driver-driver conflict* — cakupan pemeriksaan yang berbeda, bukan kontradiksi).

*Status pengujian lanjutan:* pengujian *positive control* (memprogram ulang versi RTL yang sudah dipatch/OR-*mux* dengan *firmware* uji yang sama, memverifikasi LED menyala pola `10101010`) direkomendasikan sebagai langkah verifikasi penutup, namun hasilnya belum dilaporkan secara terpisah pada penelitian ini.

### 4.5.d Ringkasan Audit Modifikasi RTL

Tabel 4.1 merangkum status dan tingkat keperluan seluruh tujuh modifikasi RTL yang diaudit pada penelitian ini, berdasarkan bukti sintesis dan pengujian perangkat keras sebagaimana diuraikan pada Subbab 4.2 dan 4.5.a–4.5.c.

**Tabel 4.1** Ringkasan Audit Modifikasi RTL

| No | Modul | Status Modifikasi | Tingkat Keperluan |
|---|---|---|---|
| 1 | `SP32B1024.v` | Ditulis dari nol | **Wajib** — modul tidak tersedia pada rilis sumber terbuka |
| 2 | `AXI_SP32B1024.v` | Tidak dipatch | **Tidak perlu** — patch `negedge`→`posedge` terbukti merusak (4.5.b) |
| 3 | `axi4_interconnect.v` | Dipatch (2 blok, *latch*→FF) | **Opsional** — peningkatan kualitas; konstruksi asli terbukti tetap dapat disintesis Yosys (4.5.a) |
| 4 | `mriscvcore.v` + `ALU.v` | Dipatch (*tri-state*→OR-*mux*) | **Wajib** — terbukti lewat sintesis penuh dan 2× uji perangkat keras bahwa RTL tanpa patch menyebabkan seluruh *write-back* register di-*hardwire* nol (4.5.c) |
| 5 | `spi_axi_master.v` | Ditulis ulang (445→101 baris) | **Wajib** untuk arsitektur *SPI runtime loader* ini |
| 6 | `basys3_spi.xdc` | Berkas baru | **Wajib** — kebutuhan struktural setiap proyek FPGA |
| 7 | `bus_sync_sf.v` | Dipatch (152 baris) | **Opsional** — praktik baik FPGA (penghapusan *net clock* terbalik) |

Dari tujuh modifikasi yang diaudit, empat bersifat wajib (#1, #4, #5, #6), dua opsional (#3, #7), dan satu dibatalkan (#2).

## 4.6 Karakterisasi Kuantitatif Frekuensi Kerja (*Fmax*) dan Utilisasi Sumber Daya

### Eksperimen A — Keterulangan (*Repeatability*)

Proses `make synth` dijalankan lima kali berturut-turut per konfigurasi, tanpa mengubah RTL di antara *run* (`make clean` dijalankan sebelum tiap *run* agar Yosys/nextpnr benar-benar dieksekusi ulang, bukan dilewati oleh *dependency cache* Make). Hasil ditunjukkan pada Tabel 4.2.

**Tabel 4.2** Hasil Eksperimen Keterulangan Sintesis (*n* = 5 per konfigurasi)

| Konfigurasi | *n* | *Fmax* Rata-rata (MHz) | Std. Dev. (MHz) |
|---|---|---|---|
| RAM 4 KB, *clock* 1,5625 MHz | 5 | 66,16 | 0,00 |
| RAM 32 KB, *clock* 50 MHz | 5 | 67,57 | 0,00 |

Standar deviasi bernilai nol pada kedua konfigurasi — kelima *run* per konfigurasi menghasilkan *Fmax* yang identik *bit*-demi-*bit*. Hal ini menunjukkan `nextpnr-xilinx` bersifat **deterministik** untuk RTL dan *constraint* yang sama ketika memakai nilai *seed default* internalnya (bukan *seed* eksplisit yang diacak), sehingga sintesis berulang pada RTL tak berubah selalu mereproduksi hasil *place-and-route* yang identik persis.

Utilisasi sumber daya turut identik pada kelima *run* per konfigurasi, sebagaimana diharapkan karena RTL yang disintesis sama; hasilnya disajikan pada Tabel 4.3.

**Tabel 4.3** Utilisasi Sumber Daya FPGA (Xilinx Artix-7 XC7A35T)

| Konfigurasi | LUT | FF | RAMB36E1 | DSP48 | CARRY4 |
|---|---|---|---|---|---|
| RAM 4 KB, 1,5625 MHz | 2163 | 1395 | 1 | 0 | 225 |
| RAM 32 KB, 50 MHz | 2181 | 1403 | 8 | 0 | 223 |

Utilisasi RAMB36E1 naik dari 1 menjadi 8 *block*, konsisten dengan ekspansi kapasitas SRAM 4 KB→32 KB (kenaikan 8×). Utilisasi LUT, FF, dan CARRY4 relatif hampir sama antarkonfigurasi (selisih <1%), karena perubahan RTL utama (lebar bus alamat 10→13 *bit*, relokasi peta alamat AXI) hanya menyentuh sedikit logika kombinasional, bukan menambah blok fungsional besar. **DSP48 bernilai 0 pada kedua konfigurasi**, mengonfirmasi secara kuantitatif bahwa modul `MULT.v` (perkalian Booth) tidak pernah tersintesis menjadi perangkat keras nyata karena tidak pernah dipanggil dari `FSM.v`.

### Eksperimen B — Sensitivitas *Seed*

RTL yang disintesis pada eksperimen ini tidak diubah dari konfigurasi RAM 32 KB/50 MHz (sama dengan Eksperimen A). *Netlist* Yosys (`spi.json`) hanya dihasilkan satu kali dan dipakai ulang untuk seluruh 10 *run* `nextpnr-xilinx` dengan nilai `--seed` eksplisit berbeda-beda, sehingga perbedaan hasil murni berasal dari algoritma *placement-and-routing*, bukan variasi *netlist*. Hasil ditunjukkan pada Tabel 4.4.

**Tabel 4.4** Sensitivitas *Fmax* terhadap Nilai *Seed* `nextpnr-xilinx` (RAM 32 KB, 50 MHz)

| *Seed* | 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8 | 9 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| *Fmax* (MHz) | 69,88 | 68,74 | 70,12 | 69,95 | 63,59 | 73,31 | 69,48 | 68,97 | 66,37 | 67,01 |

Rata-rata *Fmax* dari 10 *seed* tersebut adalah 68,74 MHz dengan standar deviasi 2,61 MHz (≈3,8% dari rata-rata), rentang 63,59–73,31 MHz (selisih 9,72 MHz, ≈14,1% dari rata-rata). Hasil ini menegaskan bahwa algoritma *placer* (*simulated annealing*) pada `nextpnr-xilinx` memang bersifat stokastik terhadap nilai *seed* eksplisit — berbeda dari kondisi pada Eksperimen A yang memakai *seed default* tetap [14]. Nilai *Fmax* pada konfigurasi *seed default* (67,57 MHz, Tabel 4.2) berada di dalam rentang hasil *sweep* 10-*seed* ini dan dekat dengan mediannya, menunjukkan bahwa *seed default* bukan kasus kebetulan terbaik atau terburuk, melainkan representasi yang cukup tipikal.

*Clock core* beroperasi pada 50 MHz. Bahkan pada *seed* terburuk dari 10 sampel (*seed* 5, 63,59 MHz), *Fmax* masih ≈1,27× di atas titik operasi 50 MHz; margin keamanan desain berkisar 1,27×–1,47× di seluruh 10 *seed* yang diuji, dan tidak ada *seed* dalam sampel ini yang menyebabkan *Fmax* turun di bawah 50 MHz. Hasil ini bukan jaminan formal atas keseluruhan ruang nilai *seed* (10 *seed* adalah sampel, bukan populasi lengkap), namun memberikan bukti empiris tambahan bahwa desain saat ini memiliki margin yang cukup terhadap variasi hasil *place-and-route*, bukan hanya kebetulan dari satu hasil *seed default* saja.
