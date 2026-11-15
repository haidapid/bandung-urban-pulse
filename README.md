# Bandung Urban Pulse: Dashboard Observatorium Kota Cerdas

Dashboard Power BI 3 halaman yang memantau **kualitas udara, kebisingan, energi surya, dan keandalan sensor IoT** dari 12 stasiun fiktif di Kota Bandung (Januari 2025 – Oktober 2026).

> **Catatan penting:** seluruh data adalah **data simulasi** yang saya buat untuk latihan dan portofolio. Angka dan lokasi stasiun **bukan** hasil pengukuran asli, jadi jangan dipakai sebagai acuan kondisi udara Bandung yang sebenarnya.

![Halaman 1 - Denyut Kota](images/Halaman_1_Denyut_Kota.png)

---

## Tujuan Proyek

Proyek ini dibuat untuk menunjukkan alur kerja analisis data dari awal sampai akhir:

1. Menyusun dataset bertema IoT dalam bentuk **model bintang** (2 tabel fakta, 2 tabel dimensi).
2. Membersihkan dan memodelkan data di **Power Query** dan Power BI.
3. Menulis **measure DAX** untuk KPI, perbandingan tahun ke tahun, dan kategori kualitas udara.
4. Merancang dashboard yang tidak biasa (bukan dashboard penjualan) dengan tema gelap dan navigasi drill-through.

Tema IoT dipilih karena saya pernah magang sebagai IoT Engineer, sehingga alur data dari sensor ke dashboard terasa dekat dengan pengalaman saya.

## Isi Dashboard

| Halaman | Fungsi | Visual utama |
|---|---|---|
| **1. Denyut Kota** | Ringkasan seluruh kota | 5 kartu KPI, peta stasiun, kalender heatmap PM2.5 ("napas kota"), ranking stasiun, hujan vs PM2.5, energi terpakai vs surya, alert per jenis |
| **2. Detak Stasiun** | Detail satu stasiun (drill-through) | Profil stasiun, gauge kemandirian surya, tren PM2.5 harian, scatter lalu lintas vs kebisingan, tabel ringkasan bulanan dengan sparkline |
| **3. Jejak Insiden** | Analisis peringatan sensor | Heatmap jam × hari, waktu penanganan (MTTR) per severity, alert per bulan, tabel detail insiden |

![Halaman 2 - Detak Stasiun](images/halaman-2-detak-stasiun.png)
![Halaman 3 - Jejak Insiden](images/halaman-3-jejak-insiden.png)

**Cara memakai:** pada Halaman 1, klik kanan satu stasiun (misalnya batang "Gedebage") → *Drill through* → *Detak Stasiun*. Gunakan tombol **← Kembali** untuk pulang ke halaman awal.

## Temuan dari Data Simulasi

Berikut contoh insight yang bisa dibaca dari dashboard (semua berasal dari data simulasi):

- Rata-rata PM2.5 seluruh stasiun sekitar **28,6 µg/m³**, dan sekitar **83,7%** hari-stasiun melewati batas pedoman WHO (15 µg/m³).
- Stasiun industri **Kopo (40,2)** dan **Gedebage (40,1)** paling tinggi, sekitar **3 kali** Tahura Juanda (12,4) yang paling bersih.
- **Musim kemarau (Juni–September)** konsisten lebih berpolusi: rata-rata PM2.5 sekitar 38 dibanding sekitar 23 di bulan lain.
- Dua episode **kabut asap** (Agustus 2025 dan September 2026) terlihat jelas di kalender heatmap.
- Panel surya menutup sekitar **78%** kebutuhan listrik stasiun secara keseluruhan (Gedebage 66%, Tahura Juanda sekitar 110%).
- Dari **762 alert**, jenis terbanyak adalah *PM2.5 Tinggi* (342, sekitar 45%). Alert paling padat terjadi pada jam sibuk pagi dan sore (jam 7 dan jam 19).
- Rata-rata penanganan alert **125 menit**, dengan alert *Kritis* butuh sekitar 273 menit dan *Rendah* sekitar 48 menit.

## Dataset

File: `data/Bandung_Urban_Pulse_Dataset.xlsx`

| Tabel | Baris | Isi |
|---|---|---|
| `Fact_Harian` | 8.028 | Data harian per stasiun: PM2.5, PM10, suhu, kelembaban, kebisingan, jumlah kendaraan, energi terpakai & surya, curah hujan, uptime |
| `Fact_Alert` | 762 | Log peringatan: tanggal, jam, jenis, severity, durasi penanganan, status |
| `Dim_Stasiun` | 12 | Nama, kecamatan, tipe zona, koordinat, kapasitas surya, batas kebisingan |
| `Dim_Tanggal` | 669 | Kalender: tahun, bulan, kuartal, hari, akhir pekan, musim |

Catatan data:
- Data dibangkitkan secara sintetis dengan pola musiman (kemarau lebih berpolusi), efek hujan, pengaruh akhir pekan, dan dua episode kabut asap.
- Nilai PM2.5 dan PM10 sengaja **kosong** saat uptime sensor < 90% (sensor offline), untuk meniru data sensor nyata. Nilai kosong ini tidak diisi 0.
- Koordinat stasiun hanya perkiraan.
- Acuan ambang: pedoman WHO 2021 (PM2.5 24 jam = 15 µg/m³) dan batas ISPU kategori Sedang (55,4 µg/m³).

### Model data (skema bintang)

```
Dim_Stasiun (1) ──< Fact_Harian (*)
Dim_Stasiun (1) ──< Fact_Alert  (*)
Dim_Tanggal (1) ──< Fact_Harian (*)
Dim_Tanggal (1) ──< Fact_Alert  (*)
```

`Dim_Tanggal` ditandai sebagai *date table*. Kolom `Bulan` diurutkan berdasarkan `Bulan_No`, dan `Hari` berdasarkan `Hari_No`.

## Measure DAX Utama

```DAX
Avg PM2.5 = AVERAGE ( Fact_Harian[PM25_ugm3] )

YoY PM2.5 % =
DIVIDE ( [Avg PM2.5] - [Avg PM2.5 LY], [Avg PM2.5 LY] )

% Hari Melampaui WHO =
DIVIDE ( [Hari Melampaui WHO], COUNT ( Fact_Harian[PM25_ugm3] ) )

Skor Udara Bersih = ROUND ( 100 * ( 1 - [% Hari Melampaui WHO] ), 0 )

Kemandirian Energi % = DIVIDE ( [Energi Surya], [Energi Terpakai] )

Avg Uptime % = DIVIDE ( AVERAGE ( Fact_Harian[Uptime_Pct] ), 100 )

MTTR (menit) = AVERAGE ( Fact_Alert[Durasi_Penanganan_Menit] )
```

Measure `Warna Kategori` dipakai untuk mewarnai bubble pada peta berdasarkan kategori kualitas udara (Baik, Sedang, Waspada, Tidak Sehat).

## Cara Membuka Proyek

1. Buka `dashboard/Bandung_Urban_Pulse.pbix` di **Power BI Desktop**.
2. Kalau Power BI meminta lokasi data, arahkan ke `data/Bandung_Urban_Pulse_Dataset.xlsx` lewat *Transform data → Data source settings*.
3. (Opsional) Untuk membuat ulang dari nol: impor 4 tabel dari file Excel, atur relasi seperti skema di atas, lalu terapkan tema `theme/Urban_Pulse_Theme.json` (*View → Themes → Browse for themes*).

Ukuran halaman: **1280 × 720** px.

## Struktur Repositori

```
bandung-urban-pulse/
├── README.md
├── data/
│   └── Bandung_Urban_Pulse_Dataset.xlsx
├── dashboard/
│   └── Bandung_Urban_Pulse.pbix
├── theme/
│   └── Urban_Pulse_Theme.json
└── images/
    ├── halaman-1-denyut-kota.png
    ├── halaman-2-detak-stasiun.png
    └── halaman-3-jejak-insiden.png
```

## Tools

- **Microsoft Excel**: sumber data (4 tabel)
- **Power Query**: pembersihan dan tipe data
- **Power BI Desktop**: pemodelan, DAX, dan visualisasi
- **JSON theme** kustom untuk tampilan gelap (aksen teal, amber, dan coral)

## Keterbatasan

- Data bersifat simulasi, jadi tidak ada kesimpulan nyata tentang kualitas udara Bandung.
- Cakupan data berhenti pada 31 Oktober 2026, sehingga November–Desember 2026 kosong.
- Perbandingan tahun ke tahun (YoY) hanya bermakna untuk Januari–Oktober karena 2025 adalah satu-satunya tahun pembanding.
