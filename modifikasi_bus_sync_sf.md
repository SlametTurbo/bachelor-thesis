# Modifikasi `bus_sync_sf.v` — Penghapusan Clock Terbalik (Inverted Clock Net)

## Fungsi modul

`bus_sync_sf.v` adalah sinkronisator bus untuk menyeberangkan sinyal antar dua domain *clock* (*clock domain crossing* / CDC), bekerja dua arah (`CLK1` ↔ `CLK2`).

## Verifikasi

Dibandingkan langsung terhadap repositori upstream `onchipuis/mriscv`:

| | Upstream | Patched (`mriscv-spi`) |
|---|---|---|
| Jumlah baris | 92 | 69 |
| Konstruksi clock terbalik | Ada (`~CLK1`, `~CLK2`) | Dihapus |

## Before

```verilog
wire NCLK2; assign NCLK2 = ~CLK2;
always @(posedge NCLK2) begin
    // ... efektif berjalan di negedge CLK2
end

wire NCLK1; assign NCLK1 = ~CLK1;
always @(posedge NCLK1) begin
    // ... efektif berjalan di negedge CLK1
end
```

## After

```verilog
// bus_sync_sf.v - FPGA-safe rewrite (tanpa ~CLK)
// konstruksi NCLK1/NCLK2 dihapus, logika ditulis ulang
// murni sensitif posedge pada masing-masing clock asli
```

Komentar pertama file secara eksplisit menyebut *"FPGA-safe rewrite (tanpa ~CLK)"*.

## Penjelasan

Membuat *net* clock terbalik secara manual (`assign NCLK = ~CLK`) adalah praktik lazim di ASIC — cukup satu sel inverter fisik, murah dan presisi. Di FPGA, konstruksi ini punya konsekuensi berbeda: sinyal `NCLK` harus dirutekan lewat *fabric* logika umum (bukan jalur *dedicated clock routing*), sehingga menimbulkan **skew** tambahan antara clock asli dan versi terbaliknya — berpotensi memicu masalah *timing closure* atau *hold violation* yang sulit dilacak di kemudian hari.

## Status dalam audit — OPSIONAL, belum dibuktikan empiris

Berbeda dari kasus tri-state `mriscvcore.v`/`ALU.v` (patch #4) yang naik status ke **wajib** setelah dibuktikan lewat sintesis penuh dan dua uji hardware independen, modifikasi `bus_sync_sf.v` ini:

- **Belum pernah diuji** apakah versi upstream (dengan `~CLK`) benar-benar gagal disintesis atau menimbulkan masalah nyata di `nextpnr-xilinx`
- Statusnya tetap tercatat sebagai **"opsional — praktik baik FPGA"**, bukan hasil pembuktian seperti patch #4
- Klasifikasi ini didasarkan pada prinsip umum desain FPGA (hindari clock net buatan sendiri), bukan bukti kegagalan konkret pada proyek ini

## Cara menaikkan bukti (kalau diperlukan untuk skripsi)

Pola pengujian sama seperti kasus tri-state: sintesis versi upstream `bus_sync_sf.v` apa adanya (modul lain tetap patched), lalu periksa log `nextpnr-xilinx` untuk warning terkait clock net atau kegagalan *timing closure*. Kalau tidak ada warning berarti — sama seperti kasus latch di `axi4_interconnect.v` — modifikasi ini benar-benar cuma perbaikan kualitas, bukan prasyarat sintesis.
