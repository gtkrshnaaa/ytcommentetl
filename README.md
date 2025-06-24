# Dokumentasi ETL Komentar YouTube

**Unstructured ➝ NDJSON ➝ CSV ➝ MariaDB ➝ Visualisasi**

---

## 1  Tujuan

Dokumen ini menjelaskan proses lengkap **ETL (Extract, Transform, Load)** untuk komentar YouTube.
Komentar publik YouTube tergolong **unstructured data**, tidak memiliki format baku dan tidak dapat dianalisis secara tabular secara langsung.
Dengan pipeline CLI ini, data:

1. **Diekstrak** ke berkas **NDJSON** (JSON per baris).
2. **Ditranformasi** menjadi **CSV** terstruktur menggunakan `awk` dan `miller` (versi apt, ≈ 6.x).
3. **Dimuat** ke database **MariaDB**.
4. **Divisualisasikan** di **Google Colab**.

---

## 2  Karakteristik Data

| Tahap             | Format       | Keterangan                                  |
| ----------------- | ------------ | ------------------------------------------- |
| Sumber            | Unstructured | Komentar bebas di YouTube                   |
| Setelah extract   | NDJSON       | Satu objek JSON per baris (semi-structured) |
| Setelah transform | CSV          | Kolom tetap: `time`, `user`, `comment`      |
| Setelah load      | Tabel SQL    | Struktur relasional di MariaDB              |

---

## 3  Perangkat

| Tool                         | Peran ETL | Catatan                           |
| ---------------------------- | --------- | --------------------------------- |
| `youtube-comment-downloader` | Extract   | Mengunduh komentar sebagai NDJSON |
| `awk`                        | Transform | Parsing awal & konversi ke CSV    |
| `miller` (apt, v6.x)         | Transform | Pembersihan, filter, sortir CSV   |
| `mariadb-server`             | Load      | Basis data relasional             |
| Google Colab + `pandas`      | Visualize | Analisis dan grafik               |

---

## 4  Instalasi di Ubuntu

```bash
pip install youtube-comment-downloader        # extractor
sudo apt update
sudo apt install awk miller mariadb-server    # transform & database
```

---

## 5  Proses ETL Terperinci

### 5.1  Extract – Mengunduh Komentar

```bash
youtube-comment-downloader \
  --url "https://www.youtube.com/watch?v=VIDEO_ID" \
  --output comments.json
```

*Output* `comments.json` berbentuk NDJSON.

---

### 5.2  Transform – NDJSON ke CSV

#### 5.2.1  Parsing Awal (awk)

```bash
awk '
  BEGIN { print "time,user,comment" }
  {
    match($0, /"time": ?"([^"]+)"/,  t)
    match($0, /"author": ?"([^"]+)"/,u)
    match($0, /"text": ?"([^"]+)"/,  c)
    gsub(/"/, "", c[1])                     # hilangkan quote sisa
    print t[1] "," u[1] "," c[1]
  }
' comments.json > comments_raw.csv
```

Penjelasan:

| Baris     | Fungsi                                                 |
| --------- | ------------------------------------------------------ |
| `match()` | Menangkap nilai `time`, `author`, dan `text` via regex |
| `gsub()`  | Menghapus tanda kutip ganda tersisa di kolom `comment` |
| `print`   | Mencetak baris CSV `time,user,comment`                 |

#### 5.2.2  Pembersihan & Sortir (miller)

Miller versi apt (≈ 6.x) tidak memiliki `trim()` dan `split()`. Pembersihan dilakukan dengan `tolower()` dan `sub()`:

```bash
mlr --csv \
  put   '$user    = tolower($user);
         $comment = sub($comment, "^ +| +$", "");' \
  then  filter '$comment =~ "[A-Za-z]"' \
  then  sort   -f user \
  comments_raw.csv > comments.csv
```

Detail operasi miller:

| Tahap `then` | Keterangan                                                                                           |
| ------------ | ---------------------------------------------------------------------------------------------------- |
| `put`        | – `tolower($user)`: username menjadi huruf kecil<br>– `sub(...)`: hapus spasi di awal/akhir komentar |
| `filter`     | Hanya baris yang mengandung setidaknya satu huruf Latin (mengabaikan komentar emoji-only)            |
| `sort`       | Mengurutkan berdasarkan kolom `user` (case-insensitive)                                              |

Hasil akhir: `comments.csv`.

---

### 5.3  Load – Memasukkan CSV ke MariaDB

1. Masuk MariaDB:

```bash
sudo mysql
```

2. Buat skema dan tabel:

```sql
CREATE DATABASE etl_ytcomments;
USE etl_ytcomments;

CREATE TABLE comments (
  time    VARCHAR(50),
  user    VARCHAR(100),
  comment TEXT
);
```

3. Muat file CSV:

```bash
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(time, user, comment);
" etl_ytcomments
```

4. Verifikasi cepat:

```bash
sudo mysql -e "SELECT COUNT(*) AS total, user FROM etl_ytcomments.comments GROUP BY user LIMIT 10;"
```
```bash
sudo mysql -e "SELECT time, user, comment FROM etl_ytcomments.comments LIMIT 10;"
```

---

### 5.4  Visualisasi di Google Colab

1. Unggah `comments.csv` ke Google Drive.
2. Jalankan notebook:

```python
from google.colab import drive
drive.mount('/content/drive')

import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('/content/drive/MyDrive/comments.csv')

# 5 kontributor teratas
top = df['user'].value_counts().head(5)
print(top)

plt.figure(figsize=(6,4))
top.plot(kind='bar', color='steelblue')
plt.title('Lima Pengguna dengan Komentar Terbanyak')
plt.ylabel('Jumlah Komentar')
plt.tight_layout()
plt.show()
```

---

## 6  Ringkasan CLI

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.json

# Transform
awk 'BEGIN{print "time,user,comment"}{
      match($0,/"time": ?"([^"]+)"/,t);
      match($0,/"author": ?"([^"]+)"/,u);
      match($0,/"text": ?"([^"]+)"/,c);
      gsub(/"/,"",c[1]);
      print t[1] "," u[1] "," c[1];
     }' comments.json > comments_raw.csv

mlr --csv put '$user=tolower($user);
               $comment=sub($comment,"^ +| +$","");
              ' \
       filter '$comment =~ "[A-Za-z]"' \
       sort -f user \
       comments_raw.csv > comments.csv

# Load
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(time, user, comment);
" etl_example
```

---

## 7  Kesimpulan

| Tahap     | Masukan                         | Keluaran                                |
| --------- | ------------------------------- | --------------------------------------- |
| Extract   | Komentar YouTube (unstructured) | `comments.json` (NDJSON)                |
| Transform | `comments.json`                 | `comments.csv` (bersih, terstruktur)    |
| Load      | `comments.csv`                  | Tabel `etl_example.comments` di MariaDB |
| Visualize | `comments.csv`                  | Grafik analisis di Google Colab         |

Dokumentasi ini sepenuhnya dapat dijalankan pada Miller bawaan repositori Ubuntu dan telah disusun untuk workflow produksi maupun keperluan riset.
