# VAM Reduksi Berbobot (VRB)

**Kelompok 8 — Institut Teknologi Sepuluh Nopember**

| Nama | NRP |
|---|---|
| Mochammad Irfan Sandy | 5025241127 |
| Adrian Afzal Zaidana | 5025241151 |
| Muhammad Naufal Hadaya Setiawan | 5025241181 |
| Mirza Rifai Dhiaurrahman | 5025241205 |
| Akmal Yusuf | 5025241212 |

Program C ini mencari solusi awal (*Initial Basic Feasible Solution* / IBFS) untuk masalah transportasi menggunakan metode **VAM Reduksi Berbobot (VRB)**. Masukkan biaya pengiriman, pasokan, dan permintaan, lalu program menampilkan jumlah yang dikirim melalui setiap rute serta total biayanya. Opsi `--trace` menampilkan keputusan pada setiap iterasi.

## Menjalankan program

Kompilasi program dengan GCC atau Clang dalam mode C11.

```sh
mkdir -p build
cc -std=c11 -O2 -Isrc/vrb -Isrc/helper \
  src/vrb/main.c src/vrb/vrb.c src/helper/input.c src/helper/trace.c \
  -o build/vrb
./build/vrb dataset/69/G3_N33.txt
./build/vrb dataset/69/G3_N33.txt --trace
```

Untuk mengetik data di terminal, jalankan `./build/vrb --cli --trace`, lalu masukkan angka berikut. Untuk masukan dari pipa, gunakan `--stdin`.

```text
3 3
3 3 5
6 5 4
6 10 7
90 80 100
70 120 80
```

Dua angka pertama adalah jumlah baris dan kolom. Tiga baris berikutnya adalah biaya per unit. Baris terakhir berisi permintaan, dan baris sebelumnya berisi pasokan. Format file sama dengan format terminal. Jika sebuah file dataset memiliki label optimum setelah permintaan, solver tidak membacanya. Untuk keluaran yang bisa dibaca program lain, tambahkan `--json`.

## Model transportasi dan IBFS

Misalkan ada $m$ sumber dan $n$ tujuan. Sebuah sel $(i,j)$ mewakili rute dari sumber $i$ ke tujuan $j$. Pada ilustrasi tabel transportasi dengan reduksi, setiap sel ditulis dalam format `Cij (Rij) [Xij]`.

| Simbol | Arti |
|---|---|
| $C(i,j)$ atau $C_{ij}$ | Biaya asli untuk mengirim satu unit pada sel $(i,j)$ |
| $R(i,j)$ atau $R_{ij}$ | Nilai biaya setelah reduksi baris dan kolom pada sel $(i,j)$ |
| $X(i,j)$ atau $X_{ij}$ | Jumlah yang dialokasikan pada sel $(i,j)$ |
| $s_i$, $d_j$ | Pasokan sumber $i$ dan permintaan tujuan $j$ pada awal masalah yang sudah seimbang |
| $s_i^{(t)}$, $d_j^{(t)}$ | Pasokan dan permintaan yang masih tersisa sebelum iterasi $t$ |
| $I_t$, $J_t$ | Himpunan baris dan kolom yang sisanya masih positif pada iterasi $t$ |

Tujuannya meminimalkan biaya total dengan memenuhi seluruh pasokan dan permintaan:

```math
\begin{aligned}
Z(X) &= \sum_{i=1}^{m}\sum_{j=1}^{n} C(i,j)X(i,j),\\
\sum_{j=1}^{n}X(i,j) &= s_i &&\text{untuk setiap sumber } i,\\
\sum_{i=1}^{m}X(i,j) &= d_j &&\text{untuk setiap tujuan } j,\\
X(i,j) &\ge 0 &&\text{untuk setiap sel } (i,j).
\end{aligned}
```

Jika total pasokan melebihi permintaan, tambahkan kolom dummy dengan permintaan sebesar selisihnya. Jika permintaan lebih besar, tambahkan baris dummy dengan pasokan sebesar selisihnya. Kedua jenis dummy memakai biaya nol.

IBFS adalah *Initial Basic Feasible Solution*, atau solusi basis layak awal. Pada masalah transportasi seimbang, basis memiliki $m+n-1$ sel independen. Jika alokasi positifnya lebih sedikit, sebagian sel basis bernilai nol. Program menampilkan alokasi tanpa menandai sel basis nol untuk simpleks lanjutan.

## Cara kerja VAM

*Vogel's Approximation Method* (VAM) menghitung penalti dari biaya **asli**. Untuk suatu garis $\ell$, yaitu satu baris atau kolom aktif, urutkan biaya sel aktif dari yang terkecil. Jika $c_{\ell,(1)}$ dan $c_{\ell,(2)}$ adalah biaya pada dua posisi pertama, penaltinya adalah:

```math
P_\ell^{\mathrm{VAM}}=
\begin{cases}
c_{\ell,(2)}-c_{\ell,(1)}, & \text{jika ada minimal dua sel aktif},\\
c_{\ell,(1)}, & \text{jika hanya ada satu sel aktif pada kode pembanding}.
\end{cases}
```

Selisih itu mengukur kenaikan biaya per unit jika pilihan termurah pada garis tersebut tidak bisa dipakai dan pilihan termurah kedua yang diambil. Biaya yang berulang tetap dihitung. Sebagai contoh, biaya $[3,3,5]$ memberi penalti $3-3=0$ karena ada dua pilihan seharga 3.

Pilih garis dengan penalti terbesar, lalu pilih sel berbiaya terkecil di garis itu. [Kode VAM pembanding](src/benchmark/methods/VAM.c) mempertahankan sel minimum pertama dan garis pertama yang ditemukan jika nilainya sama. Kode memeriksa baris dahulu, lalu kolom, masing-masing menurut urutan indeks. Alokasikan sebanyak mungkin ke sel terpilih:

```math
\begin{aligned}
q_{ij}&=\min\bigl(s_i^{(t)},d_j^{(t)}\bigr),\\
X^{(t+1)}(i,j)&=X^{(t)}(i,j)+q_{ij},\\
s_i^{(t+1)}&=s_i^{(t)}-q_{ij},\qquad
d_j^{(t+1)}=d_j^{(t)}-q_{ij}.
\end{aligned}
```

Tutup baris atau kolom yang sisanya nol, lalu hitung ulang penalti pada sel yang masih aktif. VAM menyusun solusi awal dan tidak menjamin optimum. Lihat [penjelasan dan contoh VAM oleh Shore](https://onlinelibrary.wiley.com/doi/abs/10.1111/j.1540-5915.1970.tb00792.x).

**Dua iterasi pertama G3_N33.** Mulai dari biaya asli pada tabel input. Pada iterasi 1, semua baris dan kolom masih aktif.

| Garis | Biaya asli terurut | Penalti VAM |
|---|---|---:|
| Baris 1 | 3, 3, 5 | 0 |
| Baris 2 | 4, 5, 6 | 1 |
| Baris 3 | 6, 7, 10 | 1 |
| Kolom 1 | 3, 6, 6 | **3** |
| Kolom 2 | 3, 5, 10 | 2 |
| Kolom 3 | 4, 5, 7 | 1 |

Kolom 1 memiliki penalti terbesar, yaitu $6-3=3$. VAM memilih $(1,1)$ dan mengalokasikan $\min(90,70)=70$ unit dengan biaya $70\times3=210$. Setelah kolom 1 ditutup, biaya aktif baris 3 tinggal $[10,7]$. Penaltinya $10-7=3$, terbesar pada iterasi 2. VAM memilih $(3,3)$ dan mengalokasikan $\min(100,80)=80$ unit dengan biaya $80\times7=560$. Biaya kumulatifnya menjadi 770.

## Terminologi VRB

| Istilah | Definisi |
|---|---|
| **Biaya Reduksi** ($R_{ij}$) | Biaya hasil pengurangan cost minimum baris, lalu dilanjutkan pengurangan nilai minimum kolom. |
| **FLC** (*First Least Cost*) | Biaya reduksi aktif terkecil pertama pada suatu baris/kolom. |
| **SLC** (*Second Least Cost*) | Biaya reduksi aktif terkecil kedua. |
| **TLC** (*Third Least Cost*) | Biaya reduksi aktif terkecil ketiga. |
| **MAX** (*Maximum Cost*) | Biaya reduksi aktif terbesar. |
| **Penalti** | $\mathrm{SLC} - \mathrm{FLC}$ |
| **Bobot** | $\mathrm{TLC} + \mathrm{MAX} - \mathrm{FLC}$ |
| **SKOR** | $\mathrm{Penalti} \times \mathrm{Bobot} = (\mathrm{SLC} - \mathrm{FLC}) \times (\mathrm{MAX} + \mathrm{TLC} - \mathrm{FLC})$ |
| **FLC_ROW_COUNT** | Banyak sel aktif di sepanjang baris $i$ yang memiliki nilai sama dengan kandidat FLC di sel $(i,j)$. |
| **FLC_COL_COUNT** | Banyak sel aktif di sepanjang kolom $j$ yang memiliki nilai sama dengan kandidat FLC di sel $(i,j)$. |
| **MIN_FLC_COUNT** | Nilai terkecil dari `FLC_ROW_COUNT` dan `FLC_COL_COUNT`. |
| **TOTAL_FLC_COUNT** | Jumlah `FLC_ROW_COUNT` dan `FLC_COL_COUNT`. |

**Ilustrasi perhitungan `FLC_ROW_COUNT` dan `FLC_COL_COUNT`:**

| | D1 | D2 | D3 | D4 |
|---|---:|---:|---:|---:|
| **S1** | **0** | 0 | 3 | 4 |
| **S2** | 1 | 4 | 8 | 9 |
| **S3** | 0 | 2 | 2 | 5 |

Misalkan baris `S1` sedang ditinjau dalam aturan Tie-Breaker dengan kandidat FLC = `0` pada sel `C11`:
- Pada baris `S1`, sel aktif yang bernilai `0` ada 2 (`{C11, C12}`), sehingga **`FLC_ROW_COUNT = 2`**.
- Pada kolom `D1`, sel aktif yang bernilai `0` ada 2 (`{C11, C31}`), sehingga **`FLC_COL_COUNT = 2`**.

## Aturan Tie-Breaker

### Tie-Breaker dalam Baris / Kolom
Setiap baris dan kolom aktif mengajukan satu sel calon dengan urutan prioritas:
1. Ambil sel dengan biaya reduksi minimum (**FLC**).
2. Jika beberapa sel memiliki FLC sama, pilih **ALOKASI** terbesar, yaitu $\min(\text{sisa pasokan}, \text{sisa permintaan})$.
3. Jika masih seri, pilih **indeks sel terkecil**.

### Tie-Breaker Antar Baris / Kolom
Jika lebih dari satu garis memiliki SKOR terbesar, bandingkan secara berurutan hingga ditemukan satu pemenang:

| Urutan | Kriteria | Pilih |
|---:|---|---|
| 1 | `SKOR` | Terbesar |
| 2 | `FLC` calon alokasi | Terkecil |
| 3 | `MIN_FLC_COUNT` | Terkecil |
| 4 | `TOTAL_FLC_COUNT` | Terkecil |
| 5 | `ALOKASI` ($\min(S_i, D_j)$) | Terbesar |
| 6 | Indeks baris, lalu kolom sel | Terkecil |

## Langkah informal VRB

Alur berikut mengikuti metode pada presentasi kelompok. Reduksi dilakukan **satu kali di awal**, sedangkan statistik biaya dan skor dihitung ulang pada setiap iterasi berdasarkan sel yang masih aktif.

> **Catatan implementasi:** definisi hitungan FLC di bawah mengikuti presentasi, yaitu menghitung kemunculan **nilai biaya sel calon yang sama** pada baris dan kolomnya. Kode `src/vrb/vrb.c` saat revisi dokumentasi ini masih menghitung minimum masing-masing garis dan memproses sel terakhir melalui iterasi biasa. Jejak program dan hasil benchmark yang sudah ada tetap merujuk pada implementasi tersebut.

1. **Susun matriks transportasi seimbang.** Siapkan biaya asli, pasokan, dan permintaan. Contoh pada presentasi memakai matriks **3 × 4**, dengan tiga sumber pasokan dan empat tujuan permintaan. Total pasokan harus sama dengan total permintaan.
2. **Lakukan reduksi baris.** Kurangi setiap biaya pada suatu baris dengan biaya minimum baris tersebut.
3. **Lakukan reduksi kolom.** Dari hasil reduksi baris, kurangi setiap nilai pada suatu kolom dengan nilai minimum kolom tersebut. Simpan biaya asli untuk menghitung biaya total.
4. **Hitung statistik dan skor setiap garis aktif.**
   - **4.1.** Tentukan FLC, SLC, TLC, dan MAX dari biaya tereduksi pada setiap baris dan kolom aktif.
   - **4.2.** Hitung `PENALTI = SLC - FLC`, `BOBOT = TLC + MAX - FLC`, dan `SKOR = PENALTI × BOBOT`.
5. **Pilih baris atau kolom dengan skor terbesar.**
   - **5.1.** Ambil garis dengan skor terbesar. Setiap garis mengajukan satu sel calon: biaya tereduksi terkecil, lalu kapasitas alokasi terbesar, lalu indeks sel terkecil.
   - **5.2.** Jika lebih dari satu garis memiliki skor terbesar, lanjut ke Langkah 6. Jika hanya satu, langsung ke Langkah 7.
6. **Selesaikan seri dengan aturan tie-breaker.** Bandingkan FLC terkecil, MIN_FLC_COUNT terkecil, TOTAL_FLC_COUNT terkecil, kapasitas ALOKASI terbesar, lalu indeks baris dan kolom terkecil. Begitu tersisa satu sel pemenang, lanjut ke Langkah 7.
7. **Alokasikan unit ke sel terpilih.** Kirim sebanyak `min(sisa pasokan, sisa permintaan)`. Perbarui kedua sisa tersebut dan nonaktifkan setiap garis yang sisanya nol.
8. **Periksa baris dan kolom aktif.** Jika tersisa tepat satu baris dan satu kolom aktif, lanjut ke Langkah 9. Jika masih ada margin positif dan kondisinya belum terpenuhi, ulangi Langkah **4–7**, tanpa mengulangi reduksi. Jika seluruh margin sudah nol, langsung ke Langkah 10.
9. **Alokasikan seluruh sisa unit.** Isi sel pertemuan satu baris dan satu kolom terakhir, lalu perbarui sisanya menjadi nol.
10. **Hitung biaya total.** Jumlahkan hasil perkalian alokasi akhir dengan **biaya asli**, lalu periksa kesesuaian seluruh pasokan dan permintaan.

Ukuran 3 × 4 digunakan untuk contoh presentasi; aturan yang sama dapat diterapkan pada matriks berukuran $m\times n$. Untuk masukan tidak seimbang, tambahkan baris atau kolom dummy berbiaya nol sebelum Langkah 2, seperti dijelaskan pada bagian model transportasi.

## Langkah formal VRB

Gunakan $C(i,j)$ untuk biaya asli, $R(i,j)$ untuk biaya setelah reduksi, dan $X(i,j)$ untuk alokasi. Sisa pasokan dan permintaan sebelum iterasi $t$ adalah $s_i^{(t)}$ dan $d_j^{(t)}$. Baris atau kolom disebut **aktif** jika sisanya masih positif; sel aktif berada pada perpotongan baris dan kolom aktif.

### Langkah 1. Susun matriks transportasi seimbang

Siapkan matriks biaya, vektor pasokan, dan vektor permintaan yang memenuhi:

```math
\sum_{i=1}^{m}s_i=\sum_{j=1}^{n}d_j.
```

Mulai dengan $X(i,j)=0$, $s_i^{(1)}=s_i$, dan $d_j^{(1)}=d_j$. Abaikan garis dengan margin nol. Jika seluruh margin nol sejak awal, langsung ke Langkah 10.

Contoh seimbang **3 × 4** pada presentasi:

| Sumber / Tujuan | D1 | D2 | D3 | D4 | Pasokan |
|---|---:|---:|---:|---:|---:|
| S1 | 5 | 8 | 6 | 4 | 20 |
| S2 | 5 | 4 | 4 | 2 | 30 |
| S3 | 1 | 1 | 7 | 9 | 25 |
| **Permintaan** | **30** | **15** | **20** | **10** | **75** |

Total pasokan $20+30+25=75$ sama dengan total permintaan $30+15+20+10=75$.

### Langkah 2. Reduksi baris

Gunakan himpunan baris dan kolom aktif awal $I_1$ dan $J_1$. Cari minimum setiap baris aktif, lalu kurangkan dari seluruh biaya aktif pada baris tersebut.

```math
r_i=\min_{j\in J_1}C(i,j),\qquad
B(i,j)=C(i,j)-r_i.
```

Pada contoh, minimum baris adalah $(4,2,1)$, sehingga:

```math
B=\begin{bmatrix}
1&4&2&0\\
3&2&2&0\\
0&0&6&8
\end{bmatrix}.
```

### Langkah 3. Reduksi kolom

Cari minimum setiap kolom aktif dari **hasil reduksi baris**, lalu kurangkan dari seluruh nilai aktif pada kolom tersebut.

```math
k_j=\min_{i\in I_1}B(i,j),\qquad
R(i,j)=B(i,j)-k_j.
```

Pada contoh, minimum kolom adalah $(0,0,2,0)$, sehingga:

```math
R=\begin{bmatrix}
1&4&0&0\\
3&2&0&0\\
0&0&4&8
\end{bmatrix}.
```

**Nilai $R(i,j)$ ini tetap digunakan pada seluruh iterasi. Reduksi baris dan kolom tidak diulang setelah alokasi.**

### Langkah 4. Hitung statistik biaya dan skor

**4.1. Tentukan FLC, SLC, TLC, dan MAX.** Pada iterasi $t$, gunakan:

```math
I_t=\{i:s_i^{(t)}>0\},\qquad
J_t=\{j:d_j^{(t)}>0\}.
```

Untuk setiap garis aktif $\ell$, urutkan biaya tereduksi sel aktif menjadi $v_{\ell,1}\le\cdots\le v_{\ell,h}$.

```math
\begin{aligned}
\mathrm{FLC}_\ell&=v_{\ell,1},&
\mathrm{SLC}_\ell&=v_{\ell,\min(2,h)},\\
\mathrm{TLC}_\ell&=v_{\ell,\min(3,h)},&
\mathrm{MAX}_\ell&=v_{\ell,h}.
\end{aligned}
```

Biaya yang sama tetap dihitung pada posisi berbeda. Jika hanya ada dua sel aktif, TLC = SLC. Jika hanya ada satu sel aktif, FLC = SLC = TLC = MAX.

**4.2. Hitung penalti, bobot, dan skor.**

```math
\begin{aligned}
\mathrm{PENALTI}_\ell&=\mathrm{SLC}_\ell-\mathrm{FLC}_\ell,\\
\mathrm{BOBOT}_\ell&=\mathrm{TLC}_\ell+\mathrm{MAX}_\ell-\mathrm{FLC}_\ell,\\
\mathrm{SKOR}_\ell&=\mathrm{PENALTI}_\ell\times\mathrm{BOBOT}_\ell.
\end{aligned}
```

Pada iterasi pertama contoh, skor S1, S2, S3, D1, D2, D3, D4 berturut-turut adalah **0, 0, 0, 6, 16, 0, 0**.

### Langkah 5. Pilih garis dengan skor terbesar

**5.1.** Setiap baris dan kolom aktif mengajukan satu sel calon. Di dalam satu garis:

1. Pertahankan sel dengan biaya tereduksi minimum, yaitu FLC garis itu.
2. Jika ada beberapa sel minimum, pilih kapasitas alokasi terbesar:

   ```math
   q_t(i,j)=\min\bigl(s_i^{(t)},d_j^{(t)}\bigr).
   ```

3. Jika masih seri, pilih indeks sel terkecil. Untuk calon dari baris, pilih indeks kolom terkecil; untuk calon dari kolom, pilih indeks baris terkecil.

Calon membawa SKOR dari garis pengusul. Satu sel dapat diusulkan oleh baris dan kolom dengan skor berbeda.

**5.2.** Pertahankan calon dari garis dengan SKOR terbesar. Jika hanya ada satu calon, langsung ke Langkah 7. Jika ada beberapa, lanjut ke Langkah 6.

### Langkah 6. Terapkan aturan tie-breaker

Untuk sel calon $(i,j)$, tetapkan $f=R(i,j)$, yaitu nilai FLC calon tersebut. Hitung kemunculan **nilai $f$ yang sama** pada baris $i$ dan kolom $j$, hanya pada sel aktif:

```math
\begin{aligned}
\mathrm{FLC\_ROW\_COUNT}_t(i,j)
&=\left|\{k\in J_t:R(i,k)=f\}\right|,\\
\mathrm{FLC\_COL\_COUNT}_t(i,j)
&=\left|\{k\in I_t:R(k,j)=f\}\right|,\\
\mathrm{MIN\_FLC\_COUNT}_t(i,j)
&=\min\bigl(\mathrm{FLC\_ROW\_COUNT}_t(i,j),\mathrm{FLC\_COL\_COUNT}_t(i,j)\bigr),\\
\mathrm{TOTAL\_FLC\_COUNT}_t(i,j)
&=\mathrm{FLC\_ROW\_COUNT}_t(i,j)+\mathrm{FLC\_COL\_COUNT}_t(i,j).
\end{aligned}
```

Kedua hitungan memakai **nilai biaya sel calon**, termasuk ketika nilai tersebut bukan minimum pada garis yang berpotongan. Sel calon ikut dihitung pada baris dan kolomnya. Nama singkat Fb dan Fk dapat digunakan untuk FLC_ROW_COUNT dan FLC_COL_COUNT dalam penjelasan metode ini.

Contoh pembeda: jika calon bernilai 1 dan biaya aktif barisnya $[0,1,1]$, FLC_ROW_COUNT calon itu adalah **2**, karena ada dua sel bernilai 1.

Terapkan prioritas berikut secara berurutan. Pada setiap tahap, pertahankan semua calon dengan nilai terbaik; lanjut hanya jika masih seri.

| Tahap | Kriteria | Pilih |
|---|---|---|
| Langkah 5.1 | SKOR garis pengusul | Terbesar |
| 6.1 | FLC sel calon | Terkecil |
| 6.2 | MIN_FLC_COUNT | Terkecil |
| 6.3 | TOTAL_FLC_COUNT | Terkecil |
| 6.4 | ALOKASI, yaitu $q_t(i,j)$ | Terbesar |
| 6.5 | Indeks baris, kemudian indeks kolom sel | Terkecil |

Begitu satu sel pemenang ditemukan, lanjut ke Langkah 7. Jika dua pengusul menunjuk sel yang sama, keduanya menghasilkan alokasi yang sama; tidak diperlukan aturan tambahan untuk membedakan label pengusul.

### Langkah 7. Alokasikan unit dan perbarui sisa

Untuk sel pemenang $(i,j)$, gunakan $q=\min(s_i^{(t)},d_j^{(t)})$, kemudian:

```math
\begin{aligned}
X(i,j)&\leftarrow X(i,j)+q,\\
s_i^{(t+1)}&=s_i^{(t)}-q,\\
d_j^{(t+1)}&=d_j^{(t)}-q.
\end{aligned}
```

Margin lainnya tetap. Nonaktifkan baris jika sisa pasokannya nol dan kolom jika sisa permintaannya nol. Jika keduanya habis bersamaan, nonaktifkan keduanya.

### Langkah 8. Periksa kondisi untuk iterasi berikutnya

- Jika tersisa **tepat satu baris dan satu kolom aktif**, lanjut ke Langkah 9.
- Jika masih ada margin positif dan kondisi tersebut belum terpenuhi, ulangi Langkah **4–7** pada sel aktif dengan sisa terbaru.
- Jika seluruh margin sudah nol setelah Langkah 7, langsung ke Langkah 10.

Satu baris dengan beberapa kolom aktif, atau satu kolom dengan beberapa baris aktif, masih diproses melalui Langkah 4–7. Jangan mengulangi Langkah 2 dan 3.

### Langkah 9. Alokasikan sisa pada sel terakhir

Misalkan baris terakhir adalah $i$ dan kolom terakhir adalah $j$. Karena masalah seimbang, sisa pasokan dan permintaan keduanya sama. Alokasikan seluruh sisa ke $(i,j)$ dan perbarui kedua margin menjadi nol. Tidak perlu menghitung skor atau melakukan tie-breaker lagi.

### Langkah 10. Hitung dan verifikasi biaya total

Hitung total biaya menggunakan **biaya asli**, bukan biaya tereduksi:

```math
Z=\sum_{i=1}^{m}\sum_{j=1}^{n}C(i,j)X(i,j).
```

Periksa bahwa seluruh alokasi nonnegatif, jumlah alokasi setiap baris sama dengan pasokan awalnya, dan jumlah alokasi setiap kolom sama dengan permintaan awalnya.

Untuk contoh 3 × 4, urutan alokasinya adalah:

| Tahap | Sel | Unit | Biaya asli per unit | Biaya tahap |
|---|:---:|---:|---:|---:|
| Iterasi 1 | (3,2) | 15 | 1 | 15 |
| Iterasi 2 | (3,1) | 10 | 1 | 10 |
| Iterasi 3 | (1,1) | 20 | 5 | 100 |
| Iterasi 4 | (2,3) | 20 | 4 | 80 |
| Alokasi sisa, Langkah 9 | (2,4) | 10 | 2 | 20 |
| **Total** | | **75** | | **225** |

```math
X=\begin{bmatrix}
20&0&0&0\\
0&0&20&10\\
10&15&0&0
\end{bmatrix},\qquad
Z=20(5)+20(4)+10(2)+10(1)+15(1)=225.
```

Nilai 225 adalah **total biaya transportasi**. Total unit yang dialokasikan adalah 75. Metode ini menghasilkan solusi awal layak dan tidak menjamin biaya optimum pada setiap kasus.

## Contoh Kasus 3 × 4 (Sesuai Presentasi)

Menggunakan matriks biaya asli dan nilai reduksi `Cij (Rij)` dari Langkah 1–3:

| | D1 | D2 | D3 | D4 | Supply |
|---|---:|---:|---:|---:|---:|
| **S1** | 5 (1) | 8 (4) | 6 (0) | 4 (0) | 20 |
| **S2** | 5 (3) | 4 (2) | 4 (0) | 2 (0) | 30 |
| **S3** | 1 (0) | 1 (0) | 7 (4) | 9 (8) | 25 |
| **Demand** | **30** | **15** | **20** | **10** | **75** |

### Iterasi #1
- **Skor garis aktif:** S1 = 0, S2 = 0, S3 = 0, D1 = 6, **D2 = 16**, D3 = 0, D4 = 0.
- **Keputusan (Step 5 & 7):** Pilih **D2** (skor terbesar = 16). FLC berada di `C32` (nilai 0). Alokasikan $X_{32} = \min(25, 15) = 15$. Kolom **D2 terpenuhi** (*satisfied*), sisa S3 = 10.

### Iterasi #2
- **Skor garis aktif:** S1 = 0, S2 = 0, **S3 = 64**, D1 = 6, D3 = 0, D4 = 0.
- **Keputusan (Step 5 & 7):** Pilih **S3** (skor terbesar = 64). FLC berada di `C31` (nilai 0). Alokasikan $X_{31} = \min(10, 30) = 10$. Baris **S3 habis**, sisa D1 = 20.

### Iterasi #3
- **Skor garis aktif:** S1 = 0, S2 = 0, **D1 = 10**, D3 = 0, D4 = 0.
- **Keputusan (Step 5 & 7):** Pilih **D1** (skor terbesar = 10). FLC berada di `C11` (nilai 1). Alokasikan $X_{11} = \min(20, 20) = 20$. Baris **S1 habis** dan kolom **D1 terpenuhi**.

### Iterasi #4 (Tie-Breaker)
Tersisa baris **S2** (sisa 30) serta kolom **D3** (sisa 20) dan **D4** (sisa 10).
- **Skor:** S2 = 0, D3 = 0, D4 = 0 (seri $\rightarrow$ masuk **Step 6**).
- **Step 6.1 (FLC):** S2 = 0, D3 = 0, D4 = 0 (seri).
- **Step 6.2 (`MIN_FLC_COUNT`):** Semua kandidat memiliki `FLC_ROW_COUNT = 2` (`{C23, C24}`) dan `FLC_COL_COUNT = 1` $\rightarrow$ `MIN = 1` (seri).
- **Step 6.3 (`TOTAL_FLC_COUNT`):** Semua kandidat memiliki `2 + 1 = 3` (seri).
- **Step 6.4 (`ALOKASI`):** S2 = $\min(30,20)=20$, D3 = $\min(30,20)=20$, D4 = $\min(30,10)=10$ $\rightarrow$ tersisa S2 dan D3 (keduanya menunjuk sel `C23`).
- **Step 6.5 (Indeks sel terkecil):** Pemenang di sel **`C23`**.
- **Alokasi (Step 7):** $X_{23} = \min(30, 20) = 20$. Kolom **D3 terpenuhi**, sisa S2 = 10.

### Step 9 & Step 10
- **Step 9:** Tersisa satu baris (**S2**, sisa 10) dan satu kolom (**D4**, sisa 10). Langsung alokasikan $X_{24} = 10$.
- **Step 10:** Total biaya menggunakan **biaya asli** adalah $Z = (5 \times 20) + (1 \times 10) + (1 \times 15) + (4 \times 20) + (2 \times 10) = 225$.

## Flowchart VRB

Diagram pertama mengikuti Langkah 1–10, termasuk pemisahan reduksi baris dan kolom, pengulangan mulai Langkah 4, serta alokasi langsung pada sel terakhir. Diagram kedua merinci pemilihan sel calon dan urutan tie-breaker.

![Alur VRB sesuai presentasi: persiapan, reduksi, skor, alokasi, dan biaya total](img/flowchart.svg)

![Pemilihan calon dan tie-breaker dengan hitungan berdasarkan nilai FLC sel calon](img/flowchart-tiebreak.svg)


## Contoh G3_N33 dari dataset 69

VRB mencapai biaya optimum **1.250**, sedangkan VAM menghasilkan **1.430**. Penghematan terhadap biaya VAM adalah `(1.430 - 1.250) / 1.430 × 100% = 12,59%`. Gap VAM terhadap optimum adalah `180 / 1.250 × 100% = 14,40%`.

Pada iterasi pertama VRB, baris 3 dan kolom 2 sama-sama memiliki SKOR 8, FLC 0, minimum hitungan FLC 1, dan total hitungan 3. Alokasi 90 pada sel `(1, 2)` mengalahkan 70 pada sel `(3, 1)`. VAM memilih `(1, 1)` karena penalti kolom 1 paling besar pada biaya asli. Pada iterasi ketiga, VAM kemudian mengirim 20 unit melalui sel `(3, 2)` dengan biaya 10 per unit. VRB menghindari rute tersebut. Pada contoh ini, hitungan kelangkaan tidak memutus seri, sehingga hasilnya menunjukkan perbedaan antara VRB dan VAM sebagai metode lengkap.

| Iterasi | Sel VRB | Unit VRB | Biaya tahap VRB | Sel VAM | Unit VAM | Biaya tahap VAM |
|---:|:---:|---:|---:|:---:|---:|---:|
| 1 | (1, 2) | 90 | 270 | (1, 1) | 70 | 210 |
| 2 | (2, 2) | 30 | 150 | (3, 3) | 80 | 560 |
| 3 | (3, 1) | 70 | 420 | (3, 2) | 20 | 200 |
| 4 | (2, 3) | 50 | 200 | (2, 2) | 80 | 400 |
| 5 | (3, 3) | 30 | 210 | (1, 2) | 20 | 60 |
| **Total** | | **270** | **1.250** | | **270** | **1.430** |

### Persiapan contoh dan jejak lima iterasi

Jalankan `./build/vrb dataset/69/G3_N33.txt --trace` untuk mengikuti gambar di bawah. B dan K pada tabel terminal masing-masing berarti baris dan kolom.

**Tabel awal.** G3_N33 mempunyai pasokan $(90,80,100)$ dan permintaan $(70,120,80)$. Keduanya berjumlah 270 unit. Minimum baris adalah $(3,4,6)$ dan minimum kolom setelah reduksi baris adalah $(0,0,0)$.

```math
C=
\begin{bmatrix}
3&3&5\\
6&5&4\\
6&10&7
\end{bmatrix}
\qquad
C'=
\begin{bmatrix}
0&0&2\\
2&1&0\\
0&4&1
\end{bmatrix}.
```

#### Iterasi 1. Pilih kolom 2 lewat alokasi terbesar

Kolom 2 memiliki biaya aktif terurut $[0,1,4]$. Penalti $1-0=1$, bobot $4+(4-0)=8$, dan skor $1\times8=8$. Kolom 2 mengusulkan $(1,2)$. Baris 3 juga mendapat skor 8 dari biaya terurut $[0,1,4]$ dan mengusulkan $(3,1)$.

| Pengusul | Sel | Skor | FLC | Fb | Fk | Min | Jumlah | Alokasi |
|---|:---:|---:|---:|---:|---:|---:|---:|---:|
| Baris 3 | (3,1) | 8 | 0 | 1 | 2 | 1 | 3 | 70 |
| Kolom 2 | (1,2) | 8 | 0 | 2 | 1 | 1 | 3 | 90 |

Skor, FLC, Min, dan Jumlah seri. Pilih $(1,2)$ karena $\min(90,120)=90$ lebih besar dari 70. Biayanya $90\,C(1,2)=90\times3=270$. Tutup baris 1. Sisa pasokan $(0,80,100)$ dan sisa permintaan $(70,30,80)$.

![Jejak persiapan dan iterasi 1 VRB, kolom 2 mengalokasikan 90 unit ke sel (1,2)](img/g3-n33-iterasi-1.png)

#### Iterasi 2. Pilih kolom 2 dengan skor 21

Dengan baris 1 tertutup, kolom 2 memiliki biaya $[1,4]$. TLC sama dengan SLC, yaitu 4. Penalti $4-1=3$, bobot $4+(4-1)=7$, dan skor $3\times7=21$. Skor ini mengungguli 8 dari calon terbaik lainnya. Alokasikan $\min(80,30)=30$ ke $(2,2)$. Biayanya $30\,C(2,2)=30\times5=150$, sehingga biaya kumulatif menjadi 420. Tutup kolom 2. Sisa pasokan $(0,50,100)$ dan sisa permintaan $(70,0,80)$.

![Jejak iterasi 2 VRB, skor 21 dan alokasi 30 unit di sel (2,2)](img/g3-n33-iterasi-2.png)

#### Iterasi 3. Pilih kolom 1 lewat alokasi terbesar

Kolom 1 dan baris 2 sama-sama memiliki biaya terurut $[0,2]$. Penalti 2 dan bobot 4 memberi skor 8. Kedua calon juga seri pada FLC 0, Min 1, dan Jumlah 2. Kolom 1 mengusulkan $(3,1)$ dengan $\min(100,70)=70$ unit, sedangkan baris 2 mengusulkan $(2,3)$ dengan 50 unit. Pilih $(3,1)$. Biayanya $70\,C(3,1)=70\times6=420$, sehingga biaya kumulatif menjadi 840. Tutup kolom 1. Sisa pasokan $(0,50,30)$ dan sisa permintaan $(0,0,80)$.

![Jejak iterasi 3 VRB, alokasi 70 unit menang atas 50 unit setelah seri skor dan kelangkaan](img/g3-n33-iterasi-3.png)

#### Iterasi 4. Pilih kolom 3 dengan skor 2

Kolom 3 memiliki biaya terurut $[0,1]$. Penalti $1-0=1$, bobot $1+(1-0)=2$, dan skor 2. Setiap baris aktif hanya punya satu sel sehingga skornya nol. Alokasikan $\min(50,80)=50$ ke $(2,3)$. Biayanya $50\,C(2,3)=50\times4=200$, sehingga biaya kumulatif menjadi 1.040. Tutup baris 2. Sisa pasokan dan permintaan masing-masing $(0,0,30)$.

![Jejak iterasi 4 VRB, 50 unit dikirim ke sel (2,3)](img/g3-n33-iterasi-4.png)

#### Iterasi 5. Selesaikan sisa pasokan dan permintaan

Hanya $(3,3)$ yang aktif. FLC, SLC, TLC, dan MAX semuanya 1, sehingga penalti 0, bobot 1, dan skor 0. Calon dari baris 3 dan kolom 3 menunjuk sel yang sama. Alokasikan $\min(30,30)=30$ unit. Biayanya $30\,C(3,3)=30\times7=210$. Tutup baris 3 dan kolom 3 karena kedua margin menjadi nol. Biaya akhir dari matriks asli adalah

```math
Z=90\times3+30\times5+70\times6+50\times4+30\times7=1250.
```

![Jejak iterasi 5 VRB dan matriks alokasi akhir dengan biaya 1250](img/g3-n33-iterasi-5.png)

Untuk contoh ini, konstanta reduksi adalah $3(90)+4(80)+6(100)=1190$. Biaya tereduksi alokasi VRB adalah $30(1)+30(1)=60$, sehingga biaya asli $1190+60=1250$. Alokasi VAM memiliki biaya tereduksi 240 dan biaya asli $1190+240=1430$. Selisih 180 sama pada kedua matriks.

## Pengujian dataset 69

Pembanding memakai kode yang tersedia pada proyek. NWCM mengikuti urutan sel dari sudut kiri atas, LCM memilih biaya langsung terkecil, dan [Cost Supply Method](https://etasr.com/index.php/ETASR/article/view/16468) menggabungkan biaya dengan pasokan. SAM dan RBSM menyesuaikan alokasi ketika pasokan suatu baris belum sesuai. Fractional memakai rasio dua biaya minimum sebagai penalti. Pembanding lain pada tabel memakai skor atau aturan alokasi masing-masing. Varian SAM terkoreksi, LCM2 koreksi batas, dan CSM2 int64 ditampilkan terpisah dari kode aslinya.

### Lima pembanding dengan gap terendah

Kedua grafik memakai kelompok yang sama. Lima pembanding dipilih berdasarkan gap rata-rata terendah di antara metode yang menghasilkan alokasi layak pada seluruh 69 kasus: VRB lama, TOCM-TDM1, SAM terkoreksi, RBSM, dan VAM. VRB tie-break menjadi acuan. Urutan ini memakai gap rata-rata karena jumlah optimum tidak menunjukkan besar selisih biaya pada kasus yang belum optimal.

![Jumlah solusi optimal dari 69 kasus untuk VRB tie-break dan lima pembanding](img/benchmark-optimal.svg)

VRB tie-break dan VRB lama sama-sama mencapai 55 solusi optimal dari 69 kasus. SAM terkoreksi lebih sering mencapai optimum daripada TOCM-TDM1, meskipun gap rata-ratanya sedikit lebih besar. Kedua ukuran itu menjawab pertanyaan yang berbeda.

![Median, rata-rata, dan rentang runtime program lengkap VRB serta lima pembanding](img/benchmark-runtime.svg)

Median runtime VRB tie-break adalah 0,8298 ms per kasus, lebih tinggi daripada kelima pembanding pada grafik. Runtime diukur dari program mulai sampai selesai, termasuk membaca input dan menyiapkan keluaran. Tabel di bawah tetap menampilkan semua metode, termasuk CSM dan Fractional.

### Kualitas solusi

Untuk biaya solusi $Z$ dan biaya optimum $Z^\star$, gap per kasus adalah:

```math
\mathrm{Gap}(\%)=100\,\frac{Z-Z^\star}{Z^\star}.
```

Rata-rata gap memakai kasus layak dengan optimum positif. Kolom **Optimal** menghitung hasil layak yang biayanya sama dengan optimum tersertifikasi. M/S/K berarti menang, seri, dan kalah dari sudut pandang VRB. Biaya yang lebih rendah berarti menang. Kasus dengan salah satu hasil tidak layak dikeluarkan dari perbandingan, sehingga jumlah pasangan bisa kurang dari 69.

| Metode | Layak | Gap rata-rata (%) | Optimal | M/S/K VRB | Pasangan |
|---|---:|---:|---:|---:|---:|
| VRB tie-break | 69/69 | 0,789262 | 55 | - | - |
| VRB lama | 69/69 | 0,789262 | 55 | 0/69/0 | 69 |
| VAM | 69/69 | 3,565986 | 36 | 26/38/5 | 69 |
| SAM asli | 67/69 | 1,792396 | 48 | 14/51/2 | 67 |
| SAM terkoreksi | 69/69 | 2,115392 | 48 | 16/51/2 | 69 |
| RBSM | 69/69 | 2,473487 | 45 | 24/41/4 | 69 |
| CSM | 69/69 | 3,783236 | 42 | 26/39/4 | 69 |
| Fractional | 69/69 | 3,983091 | 28 | 36/30/3 | 69 |
| NWCM_FIX | 69/69 | 44,041184 | 3 | 66/3/0 | 69 |
| LCM2 | 69/69 | 6,795846 | 14 | 51/16/2 | 69 |
| LCM2 koreksi batas | 69/69 | 6,795846 | 14 | 51/16/2 | 69 |
| CSM2 asli | 69/69 | 3,789324 | 43 | 26/39/4 | 69 |
| CSM2 int64 | 69/69 | 3,789324 | 43 | 26/39/4 | 69 |
| RCWMCAM | 69/69 | 7,039858 | 21 | 46/23/0 | 69 |
| IHW | 69/69 | 4,441102 | 40 | 26/34/9 | 69 |
| EXP-ICHW | 69/69 | 4,398083 | 34 | 31/37/1 | 69 |
| TOCM-TDM1 | 69/69 | 2,112915 | 42 | 24/38/7 | 69 |
| BCE | 62/69 | 2,705478 | 41 | 19/39/4 | 62 |
| PowerRank | 69/69 | 3,780388 | 30 | 33/33/3 | 69 |
| JHM_langsung | 67/69 | 2,916807 | 40 | 23/40/4 | 67 |

VRB dan VRB lama sama pada seluruh 69 kasus. Keduanya mencapai 55 optimum dengan gap rata-rata 0,789262%. Terhadap VAM, VRB menang pada 26 kasus, seri pada 38, dan kalah pada 5. SAM asli, BCE, dan JHM memiliki hasil tidak layak pada sebagian kasus. Gap rata-rata ketiganya memakai jumlah kasus yang berbeda.

### Runtime program lengkap

Setiap kasus dijalankan satu kali untuk pemanasan dan validasi, lalu tiga kali untuk pengukuran. Ambil median tiga pengulangan untuk tiap kasus. Tabel berikut merangkum median, rata-rata, minimum, dan maksimum dari median kasus yang layak. M/S/K runtime juga dari sudut pandang VRB. Waktu lebih singkat berarti menang. Seri berarti median dalam nanodetik persis sama.

| Metode | Median (ms) | Rata-rata (ms) | Min (ms) | Maks (ms) | M/S/K VRB | Pasangan |
|---|---:|---:|---:|---:|---:|---:|
| VRB tie-break | 0,8298 | 0,8438 | 0,7668 | 1,0787 | - | - |
| VRB lama | 0,6816 | 0,7156 | 0,6412 | 1,0973 | 2/0/67 | 69 |
| VAM | 0,5899 | 0,6130 | 0,5572 | 0,9968 | 0/0/69 | 69 |
| SAM asli | 0,5866 | 0,5959 | 0,5451 | 0,8263 | 0/0/67 | 67 |
| SAM terkoreksi | 0,5883 | 0,6010 | 0,5523 | 0,7452 | 0/0/69 | 69 |
| RBSM | 0,6056 | 0,6183 | 0,5622 | 0,8477 | 0/0/69 | 69 |
| CSM | 0,5967 | 0,6147 | 0,5582 | 0,9012 | 0/0/69 | 69 |
| Fractional | 0,9252 | 0,9495 | 0,8364 | 1,2689 | 64/0/5 | 69 |
| NWCM_FIX | 0,5932 | 0,6029 | 0,5500 | 0,8217 | 0/0/69 | 69 |
| LCM2 | 0,6390 | 0,6530 | 0,6020 | 0,9864 | 2/0/67 | 69 |
| LCM2 koreksi batas | 0,6419 | 0,6551 | 0,5947 | 0,8191 | 0/0/69 | 69 |
| CSM2 asli | 0,5927 | 0,6050 | 0,5571 | 0,7702 | 0/0/69 | 69 |
| CSM2 int64 | 0,5927 | 0,6083 | 0,5558 | 0,8422 | 0/0/69 | 69 |
| RCWMCAM | 0,6743 | 0,7011 | 0,6291 | 1,0138 | 0/0/69 | 69 |
| IHW | 0,7822 | 0,8039 | 0,7243 | 1,1663 | 11/0/58 | 69 |
| EXP-ICHW | 0,6934 | 0,7153 | 0,6421 | 1,0131 | 1/0/68 | 69 |
| TOCM-TDM1 | 0,5970 | 0,6085 | 0,5571 | 0,7754 | 0/0/69 | 69 |
| BCE | 0,5910 | 0,5981 | 0,5385 | 0,7667 | 0/0/62 | 62 |
| PowerRank | 0,6064 | 0,6359 | 0,5619 | 0,9267 | 0/0/69 | 69 |
| JHM_langsung | 0,5950 | 0,6064 | 0,5577 | 0,7565 | 0/0/67 | 67 |

Pengukuran memakai AMD Ryzen 7 8845HS, satu CPU, dan kompilasi `-O2`. Waktu mencakup proses mulai, membaca input, menghitung alokasi, menyiapkan keluaran, dan proses selesai. Format keluaran setiap implementasi dapat berbeda. Angka ini mengukur program lengkap pada lingkungan pengujian. VRB lebih cepat daripada Fractional pada 64 dari 69 kasus dan lebih lambat daripada VAM pada seluruh 69 kasus.

Untuk mengulang benchmark dengan Python 3, GCC, G++, dan `make`, jalankan:

```sh
python3 src/benchmark/run.py --output results/69-ulangan
```

[Data per kasus](results/69/cases.csv) dan [protokol pengukuran](results/69/protocol.json) mendukung tabel di atas. Hasil dataset lain tersedia di [results](results).
