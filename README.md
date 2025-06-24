# Dokumentasi Resmi: ETL Komentar YouTube (Unstructured ➝ CSV ➝ MariaDB)

---

## Tujuan

Melakukan proses **ETL (Extract, Transform, Load)** terhadap komentar YouTube yang awalnya berbentuk **unstructured text**, dan menyimpannya ke **database MariaDB**, seluruhnya menggunakan alat baris perintah (CLI) di Ubuntu.

---

## Tools yang Digunakan

| Tool                         | Peran dalam ETL      | Fungsi                             |
| ---------------------------- | -------------------- | ---------------------------------- |
| `youtube-comment-downloader` | Extract              | Mengambil komentar dari YouTube    |
| `awk`                        | Transform            | Parsing teks mentah ke CSV         |
| `miller`                     | Transform (opsional) | Filter, validasi, normalisasi data |
| `MariaDB`                    | Load                 | Menyimpan hasil akhir ke database  |

---

## Proses ETL Step-by-Step

---

### **\[E] EXTRACT – Ambil Komentar Unstructured**

#### Tujuan:

Mengambil komentar dari YouTube dan menyimpannya dalam bentuk **teks mentah** (`.txt`) tanpa struktur data.

#### Langkah:

Install downloader:

```bash
pip install youtube-comment-downloader
```

Ambil komentar:

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.txt
```

#### Output file (`comments.txt`):

```
[0:00] user1: This video is amazing!
[0:03] user2: Thanks for the explanation.
[0:06] user3: I learned a lot.
```

---

### **\[T] TRANSFORM – Ubah ke CSV**

#### Tujuan:

Mengubah teks mentah ke format **structured CSV**, agar bisa diproses oleh database.

#### Parsing dengan `awk`:

```bash
awk 'BEGIN { FS="[][:]" ; OFS=","; print "timestamp,user,comment" } {
    time=$2;
    user=$3;
    comment=substr($0, index($0,$4));
    print time, user, comment
}' comments.txt > comments.csv
```

#### Output file (`comments.csv`):

```csv
timestamp,user,comment
0:00,user1,This video is amazing!
0:03,user2,Thanks for the explanation.
0:06,user3,I learned a lot.
```

#### (Opsional) Validasi dengan `miller`:

```bash
mlr --csv put '$comment = toupper($comment)' comments.csv > comments_clean.csv
```

---

### **\[L] LOAD – Masukkan ke MariaDB**

#### Tujuan:

Memasukkan data komentar terstruktur ke dalam **tabel database** untuk analisis atau penyimpanan jangka panjang.

#### Buat database dan tabel:

```sql
CREATE DATABASE etl_example;
USE etl_example;

CREATE TABLE comments (
    timestamp VARCHAR(10),
    user VARCHAR(100),
    comment TEXT
);
```

#### Load CSV ke database:

```bash
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(timestamp, user, comment);
" etl_example
```

#### Verifikasi:

```bash
sudo mysql -e "SELECT * FROM etl_example.comments LIMIT 5;"
```

---

## Ringkasan Pipeline ETL

```bash
# [E] Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.txt

# [T] Transform
awk 'BEGIN { FS="[][:]" ; OFS=","; print "timestamp,user,comment" } {
    time=$2; user=$3; comment=substr($0, index($0,$4));
    print time, user, comment
}' comments.txt > comments.csv

# [L] Load
sudo mysql --local-infile=1 -e "
LOAD DATA LOCAL INFILE 'comments.csv'
INTO TABLE comments
FIELDS TERMINATED BY ','
LINES TERMINATED BY '\n'
IGNORE 1 ROWS
(timestamp, user, comment);
" etl_example
```

---

## Hasil Akhir

| Tahap     | Input               | Output                             |
| --------- | ------------------- | ---------------------------------- |
| Extract   | YouTube page        | File `comments.txt` (unstructured) |
| Transform | File `comments.txt` | File `comments.csv` (structured)   |
| Load      | File `comments.csv` | Tabel `comments` di MariaDB        |

---

