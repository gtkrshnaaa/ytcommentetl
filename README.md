# Dokumentasi Resmi: ETL Komentar YouTube (Unstructured ➝ CSV ➝ MariaDB)

## Tujuan

Melakukan proses **ETL (Extract, Transform, Load)** terhadap komentar YouTube yang awalnya berbentuk **unstructured text**, lalu mengubahnya ke dalam format CSV terstruktur dan memuatnya ke dalam database MariaDB.
Seluruh proses dilakukan menggunakan alat baris perintah (CLI) di Ubuntu, tanpa GUI dan tanpa ketergantungan pada Snap.

---

## Instalasi Tools

Lakukan instalasi berikut di terminal Ubuntu:

1. Install youtube-comment-downloader
```bash
pip install youtube-comment-downloader
```

2. Pastikan awk tersedia (biasanya sudah default)
```bash
awk --version
```

3. (Opsional) Install Miller untuk transformasi lanjutan
```bash
sudo apt update
```
```bash
sudo apt install miller
```

4. Install MariaDB server
```bash
sudo apt install mariadb-server
```

---

## Tools yang Digunakan

| Tool                       | Peran dalam ETL      | Fungsi                             |
| -------------------------- | -------------------- | ---------------------------------- |
| youtube-comment-downloader | Extract              | Mengambil komentar dari YouTube    |
| awk                        | Transform            | Parsing teks mentah ke CSV         |
| miller                     | Transform (opsional) | Filter, validasi, normalisasi data |
| MariaDB                    | Load                 | Menyimpan hasil akhir ke database  |

---

## Proses ETL

### Extract – Ambil Komentar Unstructured

**Tujuan**: Mengambil komentar dari video YouTube dan menyimpannya sebagai teks mentah (comments.txt).

**Perintah**:

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.txt
```

**Contoh output `comments.txt`**:

```
[0:00] user1: This video is amazing!
[0:03] user2: Thanks for the explanation.
[0:06] user3: I learned a lot.
```

---

### Transform – Ubah ke Format CSV

**Tujuan**: Mengubah teks mentah menjadi data terstruktur dalam format CSV.

**Parsing menggunakan awk**:

```bash
awk 'BEGIN { FS="[][:]" ; OFS=","; print "timestamp,user,comment" } {
    time=$2;
    user=$3;
    comment=substr($0, index($0,$4));
    print time, user, comment
}' comments.txt > comments.csv
```

**Contoh output `comments.csv`**:

```csv
timestamp,user,comment
0:00,user1,This video is amazing!
0:03,user2,Thanks for the explanation.
0:06,user3,I learned a lot.
```

**Opsional (gunakan miller untuk normalisasi atau validasi):**

```bash
mlr --csv put '$comment = tolower($comment)' comments.csv > comments_clean.csv
```

---

### Load – Masukkan ke MariaDB

**Tujuan**: Memasukkan file CSV ke dalam tabel database.

**Buat database dan tabel**:

Masuk ke MariaDB:

```bash
sudo mysql
```

Lalu jalankan:

```sql
CREATE DATABASE etl_example;
USE etl_example;

CREATE TABLE comments (
    timestamp VARCHAR(10),
    user VARCHAR(100),
    comment TEXT
);
```

**Load CSV ke database**:

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

**Verifikasi data**:

```bash
sudo mysql -e "SELECT * FROM etl_example.comments LIMIT 5;"
```

---

## Ringkasan Pipeline

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.txt

# Transform
awk 'BEGIN { FS="[][:]" ; OFS=","; print "timestamp,user,comment" } {
    time=$2; user=$3; comment=substr($0, index($0,$4));
    print time, user, comment
}' comments.txt > comments.csv

# Load
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

