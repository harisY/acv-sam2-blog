# Panduan Publish ke GitHub Pages

Isi folder ini sudah siap dipublikasikan sebagai situs Jekyll. Tidak perlu build lokal.

## 0. Status isian

Placeholder sudah terisi: username `harisY`, repo `acv-sam2-blog`, penulis "Haris".
**Ganti nama penulis di `index.md` (baris `author:`) dengan nama lengkap dan NIM Anda** bila diminta dosen.

## 1. Opsional: jalankan di Colab

Hasil di blog berasal dari run lokal (CPU). Untuk memastikan notebook juga jalan di Colab:
buka lewat badge "Buka di Colab" di blog, pilih Runtime → T4 GPU, lalu Run all.

## 2. Buat repo publik dan push

```powershell
git init
git add .
git commit -m "docs: tambah blog review SAM 2 dan SAM 3"
git branch -M main
gh repo create acv-sam2-blog --public --source . --push
```

Repo harus **public**, kalau tidak, dosen tidak bisa membuka tautannya.

## 3. Aktifkan GitHub Pages

Repo → **Settings → Pages → Build and deployment** → Source: *Deploy from a branch* →
Branch: `main`, folder `/ (root)` → Save.

Tunggu 1-3 menit. Tautan akan berbentuk:

```
https://usernameAnda.github.io/acv-sam2-blog/
```

## 4. Cek sebelum dikumpulkan

- [ ] Buka tautan di **jendela incognito** (tanpa login), halaman tampil normal.
- [ ] Diagram arsitektur muncul, tidak ada gambar rusak.
- [ ] Klik setiap tautan di Daftar isi; semuanya harus melompat ke bagian yang benar.
- [ ] Badge "Buka di Colab" membuka notebook dengan benar.
- [ ] Baca ulang seluruh tulisan dengan kata-kata Anda sendiri; ubah kalimat yang terasa bukan gaya Anda.
- [ ] Kumpulkan tautan GitHub Pages (bukan tautan repo).
