# Dokumentasi Resmi: ETL Komentar YouTube (Unstructured ➝ CSV ➝ MariaDB)

## Tujuan

Melakukan proses **ETL (Extract, Transform, Load)** terhadap komentar YouTube yang awalnya berbentuk teks bebas, mengubahnya menjadi CSV terstruktur, lalu memuatnya ke dalam MariaDB.
Seluruh proses dilakukan lewat **Command Line Interface (CLI)** di Ubuntu, dan dilengkapi dengan **visualisasi hasil** menggunakan Google Colab.

---

## Instalasi Tools

```bash
# 1. downloader komentar
pip install youtube-comment-downloader

# 2. pastikan awk tersedia
awk --version

# 3. install miller
sudo apt update
sudo apt install miller

# 4. install MariaDB
sudo apt install mariadb-server
```

---

## Tools yang Digunakan

| Tool                       | Peran ETL | Ringkasan Fungsi                                  |
| -------------------------- | --------- | ------------------------------------------------- |
| youtube-comment-downloader | Extract   | Menyimpan komentar YouTube ke file teks           |
| awk                        | Transform | Parsing awal teks mentah ke CSV                   |
| miller                     | Transform | Pembersihan, validasi, penambahan kolom, konversi |
| MariaDB                    | Load      | Menyimpan hasil akhir ke database                 |
| Google Colab               | Visualize | Membaca dan menganalisis CSV di cloud environment |

---

## Proses ETL

### 1. Extract – ambil komentar

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.txt
```

Contoh isi `comments.txt`:

```
[0:00] user1: This video is amazing!
[0:03] user2: Thanks for the explanation.
[0:06] user3: I learned a lot.
```

---

### 2. Transform – parsing dan pembersihan

#### 2.1 Parsing awal dengan awk

```bash
awk 'BEGIN { FS="[][:]" ; OFS=","; print "timestamp,user,comment" } {
    time=$2;
    user=$3;
    comment=substr($0, index($0,$4));
    print time, user, comment
}' comments.txt > comments_raw.csv
```

#### 2.2 Pembersihan dan enrich dengan miller

```bash
mlr --csv \
    put  '$user      = tolower(trim($user));
          $comment   = trim($comment);
          $word_cnt  = length(split($comment," "));
         ' \
    filter 'length($comment) > 10' \
    sort   -f user \
    comments_raw.csv > comments.csv
```

---

### 3. Load – masukkan ke MariaDB

1. Masuk MariaDB:

```bash
sudo mysql
```

2. Buat database dan tabel:

```sql
CREATE DATABASE etl_example;
USE etl_example;

CREATE TABLE comments (
    timestamp VARCHAR(10),
    user      VARCHAR(100),
    comment   TEXT,
    word_cnt  INT
);
```

3. Load CSV:

```bash
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(timestamp, user, comment, word_cnt);
" etl_example
```

4. Verifikasi isi tabel:

```bash
sudo mysql -e "SELECT * FROM etl_example.comments LIMIT 5;"
```

---

## Ringkasan CLI Pipeline

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.txt

# Transform
awk 'BEGIN {FS="[][:]"; OFS=","; print "timestamp,user,comment"}{
    time=$2; user=$3; comment=substr($0,index($0,$4));
    print time,user,comment
}' comments.txt \
| mlr --csv put '$user=tolower(trim($user));
                 $comment=trim($comment);
                 $word_cnt=length(split($comment," "));
                ' \
      filter 'length($comment) > 10' \
      sort -f user \
> comments.csv

# Load
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(timestamp, user, comment, word_cnt);
" etl_example
```

---

## Visualisasi CSV dengan Google Colab

**Langkah-langkah visualisasi data hasil transformasi:**

1. Upload `comments.csv` ke Google Drive.
2. Buka Google Colab: [https://colab.research.google.com](https://colab.research.google.com)
3. Jalankan kode berikut:

```python
# Mount Google Drive
from google.colab import drive
drive.mount('/content/drive')
```

4. Baca CSV dan tampilkan analisis dasar:

```python
import pandas as pd
import matplotlib.pyplot as plt

# Ganti path sesuai lokasi file di Google Drive
df = pd.read_csv('/content/drive/MyDrive/comments.csv')

# Tampilkan beberapa baris
print(df.head())

# Statistik jumlah kata per komentar
plt.figure(figsize=(8,4))
df['word_cnt'].hist(bins=10)
plt.title('Distribusi Jumlah Kata per Komentar')
plt.xlabel('Jumlah Kata')
plt.ylabel('Jumlah Komentar')
plt.grid(True)
plt.show()
```

---

## Hasil Akhir

| Tahap     | Input           | Output                             |
| --------- | --------------- | ---------------------------------- |
| Extract   | Halaman YouTube | File `comments.txt` (unstructured) |
| Transform | comments.txt    | File `comments.csv` (structured)   |
| Load      | comments.csv    | Tabel `comments` di MariaDB        |
| Visualize | comments.csv    | Histogram & statistik via Colab    |

---

