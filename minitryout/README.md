# Prastancy CBT — Panduan Setup

Aplikasi ujian online single-file (`index.html`) berbasis Vanilla JS + Bootstrap 5 + Firebase (Auth + Firestore). Siap deploy statis ke Vercel via GitHub.

## 1. Buat Project Firebase

1. Buka [console.firebase.google.com](https://console.firebase.google.com) → **Add project**.
2. Di **Project settings → General → Your apps**, tambahkan **Web app** → salin objek `firebaseConfig`.
3. Tempel objek tersebut ke bagian atas `<script>` di `index.html`, menggantikan placeholder `GANTI_DENGAN_...`.

## 2. Aktifkan Authentication (untuk Admin)

1. Di sidebar Firebase Console → **Build → Authentication → Get started**.
2. Tab **Sign-in method** → aktifkan **Email/Password**.
3. Tab **Users → Add user** → buat 1 akun admin manual (email + password). Akun inilah yang dipakai login di `#admin`.

> Tidak ada form "daftar admin" di aplikasi — akun admin memang sengaja hanya dibuat manual dari Console agar tidak ada pendaftaran publik ke sisi admin.

## 3. Buat Cloud Firestore

1. Sidebar → **Build → Firestore Database → Create database** → pilih mode **Production**.
2. Buka tab **Rules**, ganti isinya dengan aturan berikut, lalu **Publish**:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /tryouts/{tryoutId} {
      // Siapa saja (peserta) boleh MEMBACA data try out (kode, durasi, link CSV)
      allow read: if true;

      // Hanya user yang sudah login (admin) yang boleh menulis/menghapus
      allow write: if request.auth != null;
    }
  }
}
```

Penjelasan singkat:
- `allow read: if true` — peserta tidak perlu login untuk mengambil info try out saat menekan "Mulai Ujian".
- `allow write: if request.auth != null` — hanya request yang membawa sesi Firebase Auth (yaitu admin yang sudah login) yang boleh membuat/mengubah/menghapus dokumen try out. Tanpa login, tulisan akan ditolak oleh Firestore meskipun seseorang mencoba lewat console browser.

Setiap dokumen di collection `tryouts` berbentuk:

```
tryouts/{KODE}  (contoh id dokumen: "UTBK-1")
  - durasiMenit: number
  - csvLink: string
  - updatedAt: timestamp
```

## 4. Siapkan Google Sheets sebagai Bank Soal

Kolom minimal (urutan bebas, nama kolom fleksibel — sistem mendeteksi otomatis):

| No | Teks_Soal | Opsi_A | Opsi_B | Opsi_C | Opsi_D | Opsi_E | Kunci_Jawaban |
|----|-----------|--------|--------|--------|--------|--------|----------------|
| 1  | 2 + 2 = ? | 3      | 4      | 5      | 6      |        | B              |

Cara publikasikan sebagai CSV:
1. **File → Share → Publish to web**.
2. Pilih sheet yang benar, format **Comma-separated values (.csv)** → **Publish**.
3. Salin link yang dihasilkan (formatnya diakhiri `/pub?output=csv`) — inilah yang dimasukkan admin ke field "Link CSV Google Sheets".

## 5. Deploy ke GitHub + Vercel

1. Buat repository GitHub baru, push `index.html` (dan file ini) ke branch `main`.
2. Buka [vercel.com](https://vercel.com) → **Add New → Project** → import repo tersebut.
3. Framework preset: pilih **Other** (karena ini static HTML tanpa build step). Build command & output directory boleh dikosongkan.
4. **Deploy**. Setiap `git push` berikutnya ke `main` otomatis re-deploy.
5. (Opsional) Hubungkan domain `prastancy.fun` di tab **Domains**, arahkan path `/minitryout` sesuai kebutuhan hosting Anda.

## 6. Cara Pakai

- **Peserta**: buka domain utama (`/`), isi Nama + Kode Try Out, klik "Mulai Ujian".
- **Admin**: buka `domain/#admin`, login dengan akun yang dibuat di langkah 2, lalu tambahkan Try Out baru (Kode, Durasi, Link CSV) — tersimpan instan tanpa proses fetch berat.

## Catatan Teknis

- Jawaban & waktu tersisa peserta disimpan otomatis di `localStorage` browser peserta — refresh/tertutup tidak sengaja tidak menghilangkan progres (selama masih di device/browser yang sama).
- Admin **tidak pernah** memuat isi Google Sheets saat menyimpan Try Out — hanya menyimpan link & durasi, sehingga tidak ada risiko hang/timeout di sisi admin.
- Validasi CSV dilakukan di sisi peserta saat "Mulai Ujian", lengkap dengan pesan error yang jelas jika link salah, sheet tidak dipublikasikan, atau koneksi gagal.
