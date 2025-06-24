# Dokumentasi Resmi: ETL Komentar YouTube (Unstructured ➝ CSV ➝ MariaDB)

## Tujuan

Melakukan proses **ETL (Extract, Transform, Load)** terhadap komentar YouTube yang awalnya berbentuk teks bebas, mengubahnya menjadi CSV terstruktur, lalu memuatnya ke dalam MariaDB—seluruhnya lewat CLI di Ubuntu.

---

## Instalasi Tools
1. downloader komentar
```bash
pip install youtube-comment-downloader
```

2. awk bawaan; pastikan tersedia
```bash
awk --version
```

3. miller untuk transform lanjut
```bash
sudo apt update
```
```bash
sudo apt install miller
```

4. mariaDB
```bash
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

---

## Proses ETL

### 1. Extract – ambil komentar

```bash
youtube-comment-downloader --url "https://www.youtube.com/watch?v=VIDEO_ID" --output comments.txt
```

Contoh `comments.txt`

```
[0:00] user1: This video is amazing!
[0:03] user2: Thanks for the explanation.
[0:06] user3: I learned a lot.
```

### 2. Transform

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

Contoh pipeline `miller` yang lebih bermakna:

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

Penjelasan singkat:

| Operasi                | Fungsi                                |
| ---------------------- | ------------------------------------- |
| `tolower(trim($user))` | user ke huruf kecil, hapus spasi tepi |
| `trim($comment)`       | hapus spasi tepi komentar             |
| `word_cnt`             | kolom baru, hitung jumlah kata        |
| `filter length>10`     | buang komentar spam/pendek            |
| `sort -f user`         | urutkan berdasar kolom user           |

Hasil cuplikan `comments.csv`

```csv
timestamp,user,comment,word_cnt
0:00,user1,this video is amazing!,4
0:03,user2,thanks for the explanation.,4
0:06,user3,i learned a lot.,4
```

> Jika diperlukan konversi format (misal ke JSON) cukup:
> `mlr --icsv --ojson cat comments.csv > comments.json`

### 3. Load – masukkan ke MariaDB

1. Masuk MariaDB dan buat skema

```bash
sudo mysql
```

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

2. Muat file CSV

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

3. Verifikasi

```bash
sudo mysql -e "SELECT * FROM etl_example.comments LIMIT 5;"
```

---

## Ringkasan Pipeline

```bash
# Extract
youtube-comment-downloader --url "YOUTUBE_URL" --output comments.txt

# Transform: awk ➝ miller
awk 'BEGIN {FS="[][:]"; OFS=","; print "timestamp,user,comment"}{
      time=$2; user=$3; comment=substr($0,index($0,$4));
      print time,user,comment
     }' comments.txt \
| mlr --csv put  '$user=tolower(trim($user));
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

## Hasil Akhir

| Tahap     | Input           | Output                               |
| --------- | --------------- | ------------------------------------ |
| Extract   | Halaman YouTube | `comments.txt` (unstructured)        |
| Transform | `comments.txt`  | `comments.csv` (terstruktur, bersih) |
| Load      | `comments.csv`  | Tabel `comments` di MariaDB          |

---

