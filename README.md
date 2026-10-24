# Dokumentasi Penambahan Proses Bisnis: Modul Sanggahan E-TLE

**Identitas Mahasiswa**
* **Nama:** Muhammad Andy Dwi Subandi
* **NIM:** 264311076
* **Kelas:** 1C
* **Program Studi:** Teknologi Rekayasa Perangkat Lunak

---

## Latar Belakang

Keputusan untuk mengangkat fitur **Sanggahan (Konfirmasi Pelanggaran)** merupakan langkah arsitektural yang sangat strategis dan mencerminkan penyelesaian masalah operasional nyata pada sistem E-TLE (Electronic Traffic Law Enforcement). Penambahan ini secara langsung menutup celah *business flow* di mana pemilik kendaraan memerlukan ruang hukum untuk mengajukan keberatan valid (misalnya: kendaraan telah berpindah tangan atau hilang) sebelum status pelanggaran dibekukan menjadi denda tetap.

Berikut adalah rumusan solusi teknis yang disusun berdasarkan tiga tahapan pemodelan data terstruktur:

## 1. Pemodelan Konseptual: Pemetaan Entity-Relationship Diagram (ERD)

Pada tahap abstraksi ini, didefinisikan entitas transaksional baru beserta relasi absolutnya terhadap ekosistem E-TLE yang sudah ada.

* **Entitas:** `SANGGAHAN`

* **Atribut Inti:** `id_sanggahan` (Primary Key), `tgl_sanggah`, `kategori_alasan`, `keterangan_detail`, `bukti_dokumen`, `status_sanggahan`.

* **Definisi Relasi & Kardinalitas:**

  * **`SURAT_TILANG` – memiliki – `SANGGAHAN` (1:1):** Aturan bisnis menetapkan bahwa satu Nomor Surat Konfirmasi Tilang hanya memiliki hak satu kali pengajuan sanggahan. Kardinalitas 1:1 ini mutlak untuk mencegah eksploitasi sistem (pengajuan banding berulang pada satu kasus).

  * **`PETUGAS` – memverifikasi – `SANGGAHAN` (1:N):** Satu petugas (dengan hak akses verifikator) memiliki kewenangan untuk memproses banyak berkas sanggahan, namun satu berkas sanggahan hanya dieksekusi oleh satu petugas yang ditugaskan secara spesifik.

## 2. Pemodelan Logis: Dekomposisi Skema dan Normalisasi (1NF - 3NF)

Untuk memastikan arsitektur basis data bebas dari anomali pembaruan (*update anomaly*) dan redundansi ruang penyimpanan, skema direduksi melalui tahapan normalisasi ketat:

* **Bentuk Normal Pertama (1NF):** Seluruh atribut bernilai atomik. Atribut `bukti_dokumen` tidak diizinkan menyimpan *array* atau nilai ganda; sistem hanya akan menyimpan satu tautan/URL terpadu (*single file path*) untuk dokumen bukti.

* **Bentuk Normal Kedua (2NF):** Karena `id_sanggahan` adalah *Primary Key* bernilai tunggal, seluruh atribut transaksional (seperti `tgl_sanggah` dan `alasan`) sepenuhnya bergantung secara utuh pada kunci utama ini, tanpa adanya ketergantungan parsial.

* **Bentuk Normal Ketiga (3NF):** Sistem tidak menyimpan ulang data "Nama Pemilik" atau "Detail Pelanggaran" ke dalam tabel ini karena hal tersebut akan memicu ketergantungan transitif. Informasi tersebut diakses murni melalui pengikatan (*referencing*) ke entitas asalnya menggunakan *Foreign Key*.

**Skema Relasional Final (3NF):**
`SANGGAHAN (id_sanggahan PK, no_surat_konfirmasi FK, nrp_petugas_verif FK, tgl_sanggah, kategori_alasan, keterangan_detail, bukti_dokumen, status_sanggahan)`

## 3. Pemodelan Fisik: Eksekusi SQL DDL dan Integritas Referensial

Skrip DDL berikut dirancang dengan batasan (*constraints*) yang bertindak sebagai penjaga gerbang utama untuk menjamin integritas referensial dan mencegah munculnya *orphan data* (data yatim piatu) di level *database engine*.

```sql
CREATE TABLE sanggahan (
    id_sanggahan INT AUTO_INCREMENT PRIMARY KEY,
    
    -- Constraint UNIQUE adalah kunci penegakan relasi 1:1 secara fisik di level database
    no_surat_konfirmasi VARCHAR(50) NOT NULL UNIQUE, 
    
    -- Diizinkan NULL pada saat awal insert karena verifikator belum di-assign oleh sistem
    nrp_petugas_verif VARCHAR(20) NULL, 
    
    tgl_sanggah DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    -- Penerapan domain ketat menghindari masuknya tipe data sampah
    kategori_alasan ENUM('Kendaraan Dijual', 'Kendaraan Hilang', 'Bukan Pengemudi', 'Lainnya') NOT NULL,
    keterangan_detail TEXT NOT NULL,
    bukti_dokumen VARCHAR(255) NOT NULL,
    status_sanggahan ENUM('Menunggu', 'Proses Verifikasi', 'Diterima', 'Ditolak') DEFAULT 'Menunggu',

    -- ON DELETE RESTRICT memblokir penghapusan surat tilang jika sedang dalam proses hukum (sanggah)
    CONSTRAINT fk_sanggahan_surat
        FOREIGN KEY (no_surat_konfirmasi)
        REFERENCES surat_tilang(no_surat_konfirmasi)
        ON UPDATE CASCADE
        ON DELETE RESTRICT,

    -- ON DELETE SET NULL memastikan data sanggahan riwayat tidak hilang meski akun petugas dihapus (pensiun/mutasi)
    CONSTRAINT fk_sanggahan_petugas
        FOREIGN KEY (nrp_petugas_verif)
        REFERENCES petugas(nrp_petugas)
        ON UPDATE CASCADE
        ON DELETE SET NULL
);
```
