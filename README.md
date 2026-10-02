# Deteksi Tanda Tangan (SIGNATURE PRESENT / ABSENT)

**Nama:** Muhammad Yani
**NIM:** F1G124069

## Pipeline

1. **Crop ROI** koordinat tetap pada citra tegak 1600×1132 (Rektor: `x 230–630, y 815–925`).
2. **Grayscale** (`cv2.cvtColor`).
3. **Praproses:** Gaussian blur 5×5, lalu normalisasi latar (citra dibagi estimasi latar) agar warna kertas tiap scan seragam.
4. **Thresholding** (3 metode dibandingkan):
   * Global: T = 190 (pada citra ternormalisasi)
   * Otsu: T otomatis, dibatasi maksimum 200 (*contrast guard*). Tanpa batas ini Otsu pada kertas kosong membelah tekstur kertas menjadi foreground palsu.
   * Adaptive Gaussian: blok 31×31, C = 10
5. **Morphology:** opening (elips 3×3) untuk membuang bintik noise → closing (elips 5×5) untuk menyambung goresan putus.
6. **Fitur area:** `fg_pixels`, `fg_ratio`, jumlah *connected component*, luas komponen terbesar, dan lebar komponen terbesar / lebar ROI.
7. **Aturan keputusan:**

```
SIGNATURE PRESENT  jika fg_ratio >= 1%  DAN  lebar komponen terbesar >= 25% lebar ROI
SIGNATURE ABSENT   jika selain itu
```

Alasan aturan: kertas kosong hampir tanpa foreground; teks cetak punya foreground cukup banyak tetapi terpecah menjadi huruf-huruf kecil (komponen terbesar sempit); tanda tangan berupa goresan menyambung yang membentang lebar.

## Hasil pengujian

Total 72 sampel (24 PRESENT, 48 ABSENT).

| Metode | TP | TN | FP | FN | Akurasi |
|---|---|---|---|---|---|
| Global (T=190) | 23 | 48 | 0 | 1 | 98,6% |
| **Otsu + contrast guard** | 24 | 48 | 0 | 0 | **100%** |
| Adaptive Gaussian | 24 | 48 | 0 | 0 | **100%** |

Statistik fitur (Otsu + morfologi, rentang min–maks dari 8 scan):

| Jenis ROI | fg_pixels | fg_ratio | komponen | lebar komp. terbesar |
|---|---|---|---|---|
| Rektor | 2049–3145 | 4,7–7,1% | 1–5 | 0,57–0,93 |
| Dekan | 4399–7952 | 6,7–12,1% | 8–19 | 0,28–0,71 |
| Pemilik | 599–886 | 11,5–17,0% | 1–2 | 0,61–0,82 |
| Kertas kosong | 0–8 | ~0% | 0–1 | 0–0,005 |
| Rektor dihapus | 0 | 0% | 0 | 0 |
| Teks cetak | 315–2759 | 2,0–13,1% | 10–30 | 0,03–0,23 |

### Perbandingan metode

* **Global** sederhana, tetapi sensitif terhadap kondisi tiap scan. Satu kegagalan terjadi pada ROI Dekan di scan yang tintanya terang (goresan putus sehingga komponen terbesar sempit).
* **Otsu** menentukan T otomatis tiap citra sehingga paling konsisten, asalkan diberi batas kontras agar tidak salah pada kertas kosong.
* **Adaptive** juga bagus untuk pencahayaan tidak merata, tetapi pada teks cetak menghasilkan foreground lebih banyak (kontur huruf menebal), sehingga bergantung pada aturan lebar komponen untuk tetap benar.
* **Morfologi** (opening) membuang bintik kecil; closing menyambung goresan tipis sehingga lebar komponen terbesar lebih stabil.

### Keterbatasan

* Margin pemisah pada aturan lebar komponen cukup tipis: teks cetak maksimum 0,23 sedangkan tanda tangan Dekan terendah 0,28 (ambang 0,25). Tanda tangan yang kecil/sangat terputus atau teks dengan huruf bersambung dapat salah klasifikasi.
* Pengujian hanya pada satu dokumen (8 scan) dengan ROI tetap; belum diuji pada dokumen/penandatangan lain, coretan acak, atau posisi tanda tangan yang bergeser. Akurasi 100% **tidak boleh** dianggap berlaku umum.
* Ambang (1%, 0,25, T=190, batas Otsu 200) dipilih dengan melihat data yang sama dengan data uji, jadi hasilnya cenderung optimistis.

## Jawaban analisis

### 1. Mengapa thresholding diperlukan sebelum analisis keberadaan tanda tangan?

Citra grayscale hanya berisi nilai keabuan 0–255; belum ada pemisahan mana piksel tinta dan mana kertas. Thresholding memisahkan *foreground* (goresan tinta) dari *background* (kertas) menjadi citra biner. Dari citra biner inilah kita dapat menghitung jumlah piksel foreground, jumlah komponen, dan lebar goresan secara objektif, yang menjadi dasar aturan PRESENT/ABSENT. Tanpa segmentasi, fitur-fitur tersebut tidak dapat dihitung. Thresholding juga menyederhanakan data dan, bila dikombinasikan dengan normalisasi latar, menekan perbedaan warna kertas antar scan.

### 2. Apa masalah yang terjadi jika threshold terlalu tinggi atau terlalu rendah?

* **Terlalu rendah:** hanya piksel tergelap yang lolos. Pada scan 1 (tinta paling terang, piksel tergelap bernilai 109), T = 60 dan T = 100 menghasilkan **0 piksel foreground** dan T = 150 hanya 273 piksel terpecah-pecah, padahal tanda tangan jelas ada. Akibatnya tanda tangan asli dianggap tidak ada (*false negative*), goresan tipis putus, dan fitur area tidak representatif.
* **Terlalu tinggi:** piksel kertas ikut menjadi foreground. Pada T = 240 goresan membengkak (4655 px dibanding 2315 px pada T = 200) dan bentuknya menebal. Mendekati nilai kertas (T ≥ ~245) seluruh ROI menjadi foreground (100%, lihat `hasil/06_grafik_sensitivitas_T.png`), baik pada ROI tanda tangan maupun kertas kosong, sehingga kertas kosong bisa dianggap bertanda tangan (*false positive*) dan area/bentuk tanda tangan tidak lagi bisa dibedakan dari latar.
* Karena kecerahan tinta dan kertas berbeda pada tiap scan, threshold tetap kurang robust. Otsu atau adaptive (dengan normalisasi latar) lebih aman karena menyesuaikan diri dengan setiap citra.
