# Dokumentasi Resmi: ETL Komentar YouTube (Unstructured ➝ NDJSON ➝ CSV ➝ MariaDB ➝ Visualisasi)

---

## 1. Tujuan

Dokumen ini menjelaskan proses lengkap **ETL (Extract, Transform, Load)** untuk komentar YouTube. Komentar publik YouTube adalah **unstructured data**—tidak memiliki format baku dan tidak bisa langsung dianalisis secara tabular.

Melalui alat CLI, data tersebut:

* **Diekstrak** ke bentuk **NDJSON** (JSON per baris)
* **Ditranformasi** menggunakan **`awk`** dan **`miller`**
* **Dimasukkan** ke database MariaDB
* **Divisualisasikan** lewat Google Colab

---

## 2. Karakteristik Data

| Tahap             | Format          | Penjelasan                                                              |
| ----------------- | --------------- | ----------------------------------------------------------------------- |
| Sumber asli       | Unstructured    | Komentar publik YouTube (bebas, tidak memiliki skema tetap)             |
| Setelah extract   | Semi-structured | JSON per baris (NDJSON), memiliki struktur field dasar                  |
| Setelah transform | Structured      | CSV dengan kolom tetap: waktu, user, komentar, jumlah kata (word count) |

---

## 3. Tools yang Digunakan

| Tool                         | Fungsi      | Peran ETL                          |
| ---------------------------- | ----------- | ---------------------------------- |
| `youtube-comment-downloader` | Downloader  | Extract dari YouTube (NDJSON)      |
| `awk`                        | Text parser | Parsing awal NDJSON ke CSV kasar   |
| `miller (mlr)`               | Transformer | Bersihkan, ubah format, enrich CSV |
| `MariaDB`                    | Database    | Load data structured ke SQL        |
| `Google Colab`               | Visualisasi | Analisis dan grafik berbasis CSV   |

---

## 4. Instalasi (Ubuntu)

```bash
pip install youtube-comment-downloader

sudo apt update
sudo apt install awk miller mariadb-server
```

---

## 5. Proses ETL

### 5.1 Extract – Ambil Komentar YouTube

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.json
```

Output berupa **file `.json`** berisi **NDJSON** (1 JSON per baris).

Contoh:
```bash
head -n 20 comments.json
```

```json
{"text": "Great video!", "author": "@user1", "time": "1 hari yang lalu", ...}
{"text": "I love this part!", "author": "@user2", "time": "2 hari yang lalu", ...}
```

---

### 5.2 Transform – JSON ➝ CSV (awk ➝ miller)

#### a. Ambil kolom penting pakai `awk`

```bash
awk '
  BEGIN { print "time,user,comment" }
  {
    match($0, /"time": ?"([^"]+)"/, time)
    match($0, /"author": ?"([^"]+)"/, user)
    match($0, /"text": ?"([^"]+)"/, comment)
    gsub(/"/, "", comment[1])
    print time[1] "," user[1] "," comment[1]
  }
' comments.json > comments_raw.csv
```

> Ini mengubah NDJSON ke CSV kasar dengan 3 kolom: `time`, `user`, `comment`.

#### b. Bersihkan dan tambah word count dengan `miller`

```bash
mlr --csv \
  put  '$user = tolower(sub($user, "^ +| +$", ""));
        $comment = sub($comment, "^ +| +$", "");
        $word_cnt = length(split($comment, " "));
       ' \
  filter 'length($comment) > 10' \
  sort -f user \
  comments_raw.csv > comments.csv
```

---

### Penjelasan:

* `sub($user, "^ +| +$", "")`: menghapus spasi di awal dan akhir string (fungsi ini bisa ganti `trim()`).
* `tolower(...)`: tetap valid, buat lowercase username.
* `split($comment, " ")`: hitung jumlah kata di komentar.
* `filter`: buang komentar pendek banget.
* `sort -f user`: urutkan berdasarkan username (abaikan kapital).

---

### 5.3 Load – Masukkan ke MariaDB

1. Masuk MariaDB dan buat tabel:

```bash
sudo mysql
```

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

2. Load data ke database:

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

---

## 6. Visualisasi di Google Colab

1. Upload `comments.csv` ke Google Drive
2. Buka Google Colab → Jalankan:

```python
from google.colab import drive
drive.mount('/content/drive')
```

3. Visualisasi:

```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv('/content/drive/MyDrive/comments.csv')

# Statistik awal
print(df.head())

# Komentar terpanjang
print("\nKomentar terpanjang:")
print(df.loc[df['word_cnt'].idxmax()])

# Histogram jumlah kata
plt.figure(figsize=(8,4))
df['word_cnt'].hist(bins=10, color='steelblue')
plt.title('Distribusi Panjang Komentar')
plt.xlabel('Jumlah Kata')
plt.ylabel('Jumlah Komentar')
plt.grid(True)
plt.tight_layout()
plt.show()
```

---

## 7. Ringkasan Perintah (Pipeline CLI)

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.json

# Transform (awk ➝ miller)
awk '
  BEGIN { print "time,user,comment" }
  {
    match($0, /"time": ?"([^"]+)"/, time)
    match($0, /"author": ?"([^"]+)"/, user)
    match($0, /"text": ?"([^"]+)"/, comment)
    gsub(/"/, "", comment[1])
    print time[1] "," user[1] "," comment[1]
  }
' comments.json > comments_raw.csv

mlr --csv \
  put  '$user = tolower(trim($user));
        $comment = trim($comment);
        $word_cnt = length(split($comment, " "));
       ' \
  filter 'length($comment) > 10' \
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

| Tahap     | Format                 | Keterangan                                |
| --------- | ---------------------- | ----------------------------------------- |
| Extract   | NDJSON (per baris)     | Komentar YouTube dibungkus dalam JSON     |
| Transform | CSV (via awk + miller) | Parsing manual + enrich kolom word count  |
| Load      | SQL Table              | Data structured dimuat ke MariaDB         |
| Visualize | CSV di Google Colab    | Statistik, filtering, distribusi komentar |

---




