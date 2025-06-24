# Dokumentasi Resmi: ETL Komentar YouTube

**(Unstructured ➝ NDJSON ➝ CSV ➝ MariaDB ➝ Visualisasi)**

---

## 1. Tujuan

Dokumen ini menjelaskan proses lengkap **ETL (Extract, Transform, Load)** pada komentar publik YouTube.
Komentar YouTube secara alami bersifat **unstructured** — bebas format, tanpa skema, dan sulit diolah secara langsung.
Melalui pipeline ini, komentar tersebut:

* **Diekstrak** menjadi **NDJSON** (semi-structured)
* **Ditranformasi** ke dalam **CSV** (structured)
* **Dimasukkan** ke **database MariaDB**
* **Divisualisasikan** di Google Colab

---

## 2. Alur Transformasi Data

```text
[YouTube Comments]
     (unstructured)
          ↓ Extract
  [NDJSON file: comments.txt]
     (semi-structured)
          ↓ Transform
     [CSV file: comments.csv]
       (structured)
          ↓ Load
   [Tabel: MariaDB - comments]
          ↓ Visualisasi
     [Grafik via Google Colab]
```

---

## 3. Tools yang Digunakan

| Tool                       | Peran ETL | Fungsi                                             |
| -------------------------- | --------- | -------------------------------------------------- |
| youtube-comment-downloader | Extract   | Ambil komentar YouTube, output NDJSON per baris    |
| miller (`mlr`)             | Transform | Parsing JSON, bersihkan kolom, hitung metrik, dsb. |
| MariaDB                    | Load      | Menyimpan hasil akhir dalam format relasional      |
| Google Colab               | Visualize | Visualisasi data berbasis file CSV                 |

---

## 4. Instalasi di Ubuntu

```bash
# 1. Extractor
pip install youtube-comment-downloader

# 2. Transform tools
sudo apt update
sudo apt install miller

# 3. Database
sudo apt install mariadb-server
```

---

## 5. Step-by-Step ETL

### 🔹 5.1 Extract – Ambil Komentar dari YouTube

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.txt
```

File `comments.txt` berisi data dalam format NDJSON (newline-delimited JSON).
Contoh:

```json
{"cid":"xxx","text":"Great video!","author":"@user1","time":"1 hari yang lalu",...}
{"cid":"yyy","text":"Insane quality!","author":"@user2","time":"2 hari yang lalu",...}
```

> Pada tahap ini, komentar YouTube yang **unstructured** telah dikemas menjadi **semi-structured JSON**.

---

### 🔹 5.2 Transform – JSON ➝ CSV

#### a. Ubah NDJSON ke CSV:

```bash
mlr --ijson --ocsv cat comments.txt > comments_raw.csv
```

#### b. Bersihkan dan Enrich:

```bash
mlr --csv \
  put '$user = tolower($author);
       $comment = $text;
       $word_cnt = length(split($text," "));
      ' \
  filter 'length($text) > 10' \
  cut -f time,user,comment,word_cnt \
  sort -f user \
  comments_raw.csv > comments.csv
```

Penjelasan:

* Kolom `author` diganti menjadi `user` (huruf kecil)
* Hitung jumlah kata (`word_cnt`)
* Filter komentar terlalu pendek

---

### 🔹 5.3 Load – Masukkan ke MariaDB

1. Masuk ke MariaDB:

```bash
sudo mysql
```

2. Buat database dan tabel:

```sql
CREATE DATABASE etl_example;
USE etl_example;

CREATE TABLE comments (
    time VARCHAR(50),
    user VARCHAR(100),
    comment TEXT,
    word_cnt INT
);
```

3. Muat CSV:

```bash
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(time, user, comment, word_cnt);
" etl_example
```

4. Verifikasi:

```bash
sudo mysql -e "SELECT * FROM etl_example.comments LIMIT 5;"
```

---

## 6. Visualisasi Data di Google Colab

1. Upload `comments.csv` ke Google Drive
2. Buka [Google Colab](https://colab.research.google.com)
3. Jalankan:

```python
from google.colab import drive
drive.mount('/content/drive')
```

4. Analisis dan grafik:

```python
import pandas as pd
import matplotlib.pyplot as plt

# Ganti path sesuai lokasi file
df = pd.read_csv('/content/drive/MyDrive/comments.csv')

# Statistik awal
print(df.head())

# Komentar terpanjang
print("\nKomentar terpanjang:")
print(df.loc[df['word_cnt'].idxmax()])

# Distribusi jumlah kata
plt.figure(figsize=(8,4))
df['word_cnt'].hist(bins=10, color='teal')
plt.title('Distribusi Panjang Komentar')
plt.xlabel('Jumlah Kata')
plt.ylabel('Jumlah Komentar')
plt.grid(True)
plt.tight_layout()
plt.show()
```

---

## 7. Ringkasan CLI Pipeline

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.txt

# Transform
mlr --ijson --ocsv cat comments.txt > comments_raw.csv

mlr --csv put '$user=tolower($author);
                $comment=$text;
                $word_cnt=length(split($text," "));
               ' \
    filter 'length($text) > 10' \
    cut -f time,user,comment,word_cnt \
    sort -f user \
    comments_raw.csv > comments.csv

# Load
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(time, user, comment, word_cnt);
" etl_example
```

---

## 8. Kesimpulan

| Tahap     | Format                | Keterangan                                     |
| --------- | --------------------- | ---------------------------------------------- |
| Extract   | Unstructured ➝ NDJSON | Komentar mentah dibungkus jadi JSON per baris  |
| Transform | NDJSON ➝ CSV          | Parsing, filter, enrich (jumlah kata)          |
| Load      | CSV ➝ MariaDB         | Data dimasukkan ke tabel relasional            |
| Visualize | CSV ➝ Grafik          | Analisis dan statistik visual via Google Colab |

---

