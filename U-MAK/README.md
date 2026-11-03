# U-MAK

Aplikasi penilaian kesehatan organisasi UMKM (6 faktor OH-Index). Hasil rombakan dari NexaUMKM:
tampilan klasik-minimalis, backend **Supabase** (PostgreSQL + Auth), tanpa PHP dan tanpa build step.

```
U-MAK/
├── index.html
├── css/style.css
├── js/config.js        ← isi URL & anon key Supabase di sini
├── js/app.js
└── supabase/
    ├── 01_schema.sql   ← tabel, keamanan (RLS), fungsi, view
    └── 02_seed_survei.sql  ← 428 responden dari data_asli.csv
```

## Tutorial Supabase (±10 menit)

### 1. Buat project
1. Daftar/masuk di <https://supabase.com>, klik **New project**.
2. Isi nama (mis. `u-mak`), buat **Database Password** (simpan), pilih region **Southeast Asia (Singapore)**.
3. Tunggu project selesai dibuat.

### 2. Buat tabel dan data
1. Menu kiri **SQL Editor → New query**.
2. Salin seluruh isi `supabase/01_schema.sql`, tempel, klik **Run**. Harus muncul "Success".
3. Buat query baru, tempel isi `supabase/02_seed_survei.sql`, klik **Run**. Ini mengisi 428 responden.
4. Cek di **Table Editor → survey_responses**: harus ada 428 baris.

> Jalankan `01_schema.sql` hanya sekali. Jika perlu mengulang dari nol, hapus tabel lebih dulu
> (`drop table public.submissions, public.profiles, public.survey_responses, public.factors cascade;`
> dan `drop view public.v_scores;`).

### 3. Atur autentikasi
Menu **Authentication → Sign In / Providers → Email**:
- Untuk **uji coba**, matikan **Confirm email** agar akun langsung aktif setelah daftar.
- Untuk **produksi**, biarkan menyala, lalu isi **Authentication → URL Configuration → Site URL**
  dengan alamat website Anda agar tautan konfirmasi mengarah ke tempat yang benar.

### 4. Hubungkan website
1. **Project Settings → API** (atau tombol **Connect**). Salin **Project URL** dan **anon / publishable key**.
2. Buka `js/config.js`, ganti kedua nilai.

```js
window.UMAK_CONFIG = {
  SUPABASE_URL: 'https://abcdefgh.supabase.co',
  SUPABASE_ANON_KEY: 'eyJhbGciOi...',
};
```

Kunci anon memang boleh tampil di browser karena data dilindungi Row Level Security.
**Jangan pernah** memakai kunci `service_role` di sini.

### 5. Jalankan
Buka folder di terminal lalu jalankan salah satu:

```bash
python3 -m http.server 8000      # lalu buka http://localhost:8000
# atau: npx serve .
# atau: ekstensi "Live Server" di VS Code
```

### 6. Uji alurnya
1. **Daftar sebagai Pemilik** (mis. UMKM "Warung Contoh"), lalu isi **Penilaian** (16 pertanyaan).
2. Keluar, **daftar sebagai Pegawai** dengan **nama UMKM yang sama persis**, lalu isi penilaian (19 pertanyaan).
3. Buka **Ringkasan** / **Data UMKM**: "Warung Contoh" muncul dengan tanda *terdaftar*, bersama data survei.
4. Cek di Supabase **Table Editor → profiles / submissions** untuk melihat data yang masuk.

### 7. Publikasikan (opsional)
Karena situs ini statis, cukup unggah foldernya ke Netlify, Vercel, Cloudflare Pages, atau GitHub Pages.
Setelah punya alamat, masukkan ke **Authentication → URL Configuration → Site URL**.

## Cara kerja
| Hal | Di NexaUMKM (lama) | Di U-MAK |
|---|---|---|
| Database | MySQL (InfinityFree) | Supabase PostgreSQL |
| Login | JWT buatan sendiri + PHP | Supabase Auth |
| Penyimpanan skor | Tabel `submissions`, skor ditulis ulang saat pegawai mengisi | Hanya jawaban mentah disimpan; skor dihitung oleh view `v_scores` sehingga selalu konsisten |
| Validasi kuesioner | `validator.php` | Fungsi `submit_assessment()` di database (lengkap, rentang 1–N, sekali isi, pegawai menunggu pemilik) |
| Data survei 428 responden | `data_asli.csv` dibaca browser, nama/lokasi/omzet dikarang acak | Tabel `survey_responses`, tampil anonim ("Responden 001") |

Skor total = rata-rata tertimbang (INS 0,20; OV 0,18; LDI 0,17; WEQ, OPS, ECT 0,15), skala 0–5.
Kategori: Sangat Baik ≥ 4, Baik > 3, Cukup ≥ 2, Buruk < 2.

## Keamanan
- Tabel dilindungi RLS: pengguna hanya bisa membaca profil dan jawabannya sendiri. Penulisan hanya lewat `submit_assessment()`.
- Semua pengguna login dapat melihat skor gabungan (nama UMKM terdaftar, sektor, lokasi, skor), bukan jawaban individual atau email.
- **Ganti kredensial lama.** File `env` di NexaUMKM berisi kredensial database dan `JWT_SECRET` yang ikut terunggah.
  Anggap sudah bocor: ganti password MySQL, dan jangan ikutkan file itu ke repositori baru.

## Catatan data
- Kolom `INS` pada data survei asli berskala 1–3 (belum dinormalisasi), sedangkan penilaian baru di U-MAK
  menormalkannya ke 0–5 seperti logika PHP lama. Skor INS UMKM terdaftar karena itu tidak sepenuhnya sebanding
  dengan data survei.
- Hanya skor per faktor yang diimpor; 35 kolom jawaban mentah (OH1–OH35) tidak dimasukkan ke database.
