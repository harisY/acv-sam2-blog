---
layout: post
title: "Dari SAM 2 ke SAM 3: Review Paper dan Tutorial Segmentasi Gambar dan Video"
author: "Haris"
date: 2026-10-21
description: "Review paper SAM 2 (ICLR 2025) dan SAM 3 (2025), ditambah tutorial SAM 2.1 di Google Colab, plus ide riset untuk tesis."
---

> Catatan mata kuliah **Advanced Computer Vision**, Magister Teknologi Informasi.
> Ditulis untuk teman-teman mahasiswa S2 yang butuh gambaran cepat tentang *foundation model* segmentasi dan ingin langsung mencobanya.

## Daftar isi

1. [Kenapa topik ini penting](#1-kenapa-topik-ini-penting)
2. [SAM 2: apa yang baru](#2-sam-2-apa-yang-baru)
3. [Arsitektur SAM 2](#3-arsitektur-sam-2)
4. [Data engine dan dataset SA-V](#4-data-engine-dan-dataset-sa-v)
5. [Hasil utama dan kelemahan](#5-hasil-utama-dan-kelemahan)
6. [SAM 3: segmentasi berbasis konsep](#6-sam-3-segmentasi-berbasis-konsep)
7. [Tutorial: SAM 2.1 di Google Colab](#7-tutorial-sam-21-di-google-colab)
8. [Ide riset untuk tesis](#8-ide-riset-untuk-tesis)
9. [Penutup](#9-penutup)
10. [Daftar pustaka](#daftar-pustaka)

---

## 1. Kenapa topik ini penting

Segmentasi berarti menentukan piksel mana yang membentuk sebuah objek. Hasilnya dipakai di banyak bidang:
analisis citra medis, pemetaan lahan dari citra satelit, kendaraan otonom, sampai penyuntingan video.
Masalahnya, model segmentasi klasik harus dilatih ulang untuk tiap domain, dan anotasi piksel sangat mahal.

*Segment Anything Model* (SAM) dari Meta mengubah pendekatan itu. Alih-alih melatih model untuk kelas tertentu,
SAM dilatih sekali pada data yang sangat besar lalu **dipandu lewat prompt** (klik, kotak, atau mask)
untuk segmentasi objek apa saja. Idenya mirip dengan cara LLM dipandu lewat teks.

Tulisan ini mengikuti evolusinya:

- **SAM 2** (2024) memperluas SAM ke **video**, dengan memori antarframe.
- **SAM 3** (2025) memperluasnya lagi ke **konsep**: cukup ketik "bus sekolah kuning" dan model menemukan *semua* instans objek itu.

Paper SAM 2 dipresentasikan di konferensi **ICLR 2025** (sesi oral). Paper SAM 3 saat ini saya rujuk sebagai
*preprint* di arXiv. Saya tidak menemukan informasi venue publikasinya, jadi tidak saya tulis.

## 2. SAM 2: apa yang baru

Kontribusi utama Ravi dkk. (2025) ada tiga:

1. **Tugas baru: *Promptable Visual Segmentation* (PVS).** Pengguna memberi prompt (klik positif/negatif, kotak, atau mask)
   di frame mana pun. Model menyebarkannya menjadi *masklet*, yaitu mask objek yang konsisten sepanjang waktu.
   Prompt tambahan di frame lain bisa dipakai untuk memperbaiki hasil. PVS mencakup tugas segmentasi gambar (SAM)
   dan *semi-supervised video object segmentation* (mask hanya di frame pertama) sebagai kasus khusus.
2. **Model dengan *streaming memory*.** Video diproses frame demi frame sambil membawa memori objek,
   sehingga cocok untuk pemrosesan real-time.
3. **Data engine dan dataset SA-V.** Dataset video segmentasi terbesar pada masanya, dibangun dengan model-in-the-loop.

Penulis melaporkan SAM 2 lebih akurat dengan interaksi sekitar **3 kali lebih sedikit** dibanding pendekatan sebelumnya pada video,
dan pada gambar **lebih akurat sekaligus sekitar 6 kali lebih cepat** daripada SAM pertama.

## 3. Arsitektur SAM 2

![Diagram sederhana arsitektur SAM 2]({{ '/assets/img/sam2-arsitektur.svg' | relative_url }})

Alur untuk satu frame, berdasarkan deskripsi di paper:

| Komponen | Fungsi |
|---|---|
| **Image encoder** | Hiera yang dilatih awal dengan MAE. Struktur hierarkis menghasilkan fitur multi-skala yang berguna untuk dekoding mask. |
| **Memory attention** | Beberapa lapis transformer (default 4) yang membuat fitur frame saat ini "melihat" memori masa lalu lewat *self-attention* dan *cross-attention*. |
| **Prompt encoder** | Sama seperti SAM: mengubah klik, kotak, atau mask menjadi embedding. |
| **Mask decoder** | Memprediksi mask dari fitur dan prompt. Ditambah *head* yang memprediksi apakah objek **sedang tidak terlihat** (oklusi). |
| **Memory encoder** | Menggabungkan fitur frame dengan mask prediksi menjadi memori. |
| **Memory bank** | Antrean FIFO untuk beberapa frame terakhir (default 6) dan frame yang diberi prompt, ditambah *object pointer* yang membawa informasi semantik objek. |

Kuncinya ada di **memory attention**. Itu yang membuat model bisa melacak objek yang sempat tertutup lalu muncul lagi,
sesuatu yang sulit bila setiap frame disegmentasi sendiri-sendiri seperti SAM pertama.

## 4. Data engine dan dataset SA-V

Data dibangun dalam tiga fase. Di setiap fase, model yang sudah ada dipakai untuk mempercepat anotator manusia.
Waktu anotasi per frame menurun drastis:

| Fase | Model di dalam loop | Waktu per frame | Percepatan vs fase 1 |
|---|---|---|---|
| 1 | SAM per frame | 37,8 detik | 1x |
| 2 | SAM + SAM 2 Mask | 7,4 detik | sekitar 5,1x |
| 3 | SAM 2 | 4,5 detik | sekitar 8,4x |

*Sumber: Ravi dkk. (2025), tabel data engine.*

Hasilnya adalah **SA-V**: 50,9 ribu video, 642,6 ribu masklet (190,9 ribu manual, 451,7 ribu otomatis),
dan 35,5 juta mask. Video direkam di 47 negara. Penulis menyebut jumlah mask-nya 53 kali lipat dibanding dataset VOS yang ada.

**Pelajaran untuk riset:** pola *model-in-the-loop* ini bisa ditiru untuk membangun dataset anotasi di domain Anda sendiri,
misalnya dataset medis atau citra drone, dengan biaya anotasi jauh lebih kecil.

## 5. Hasil utama dan kelemahan

**Hasil yang dilaporkan di paper** (semuanya klaim penulis, diambil dari Ravi dkk., 2025):

| Aspek | Hasil |
|---|---|
| Kecepatan gambar (A100, batch 10 gambar) | SAM 21,7 FPS vs SAM 2 130,1 FPS (sekitar 6x); mIoU 1 klik pada 23 dataset: 58,1 vs 58,9 |
| Video, prompt 3 klik di frame pertama (17 dataset zero-shot) | SAM 2 mencapai 75,3 J&F; baseline SAM + Cutie 70,1; SAM + XMem++ 68,4 |
| Kecepatan video (A100, batch 1) | 43,8 FPS (varian B+) dan 30,2 FPS (varian L) |

**Kelemahan yang diakui penulis** (Lampiran C):

- Bisa gagal saat terjadi **pergantian shot** (potongan kamera).
- Objek bisa hilang atau tertukar pada **adegan ramai**, **oklusi panjang**, atau **video sangat panjang**.
- Kesulitan pada **detail tipis** yang bergerak cepat.
- Dapat tertukar antara objek serupa yang berdekatan.
- Tiap objek diproses **terpisah**, tanpa komunikasi antarobjek.

Poin-poin ini langsung menjadi peluang riset (lihat bagian 8).

### Memilih ukuran model: SAM 2.1

Setelah paper terbit, Meta merilis **SAM 2.1** (30 September 2024) dengan checkpoint yang diperbarui.
Berikut tabel dari README repositori resmi. Angka kecepatan diukur di A100 dengan PyTorch 2.5.1 dan CUDA 12.4.

| Model | Parameter (juta) | FPS | J&F pada SA-V test |
|---|---|---|---|
| `hiera_tiny` | 38,9 | 91,2 | 76,5 |
| `hiera_small` | 46 | 84,8 | 76,6 |
| `hiera_base_plus` | 80,8 | 64,1 | 78,2 |
| `hiera_large` | 224,4 | 39,5 | 79,5 |

*Sumber: README [facebookresearch/sam2](https://github.com/facebookresearch/sam2). Angka FPS di sini diukur dengan pengaturan berbeda dari FPS di paper, jadi jangan dibandingkan langsung dengan tabel sebelumnya.*

Bagi mahasiswa yang hanya punya GPU gratis di Colab, `tiny` atau `small` sudah cukup untuk belajar dan sangat cepat.

## 6. SAM 3: segmentasi berbasis konsep

SAM 3 ("Segment Anything with Concepts", Carion dkk., 2025) menjawab keterbatasan lain: pada SAM 1/2 Anda harus
**menunjuk satu objek** lewat klik. Padahal sering kali kita ingin "semua objek bertipe X".

**Gagasan inti.** SAM 3 menyelesaikan tugas *Promptable Concept Segmentation* (PCS): diberi **frasa kata benda pendek**
(misalnya "mobil merah") atau **contoh gambar**, model mendeteksi, mensegmentasi, dan melacak **semua instans** konsep itu
di gambar atau video, lengkap dengan identitas unik tiap instans.

**Arsitektur singkat** (menurut paper dan README resmi):

- Sebuah **detektor** (berbasis DETR) untuk gambar dan sebuah **tracker** berbasis memori yang mewarisi arsitektur SAM 2.
  Keduanya memakai satu *vision encoder* bersama.
- Ada **presence head** yang memisahkan pertanyaan "apakah konsep ini ada di gambar?" dari "di mana lokasinya?".
  Menurut penulis, pemisahan ini meningkatkan akurasi deteksi.
- Ukuran model sekitar **848 juta parameter**.

**Data dan benchmark.** Penulis membangun data engine dengan sekitar 4 juta label konsep unik (termasuk *hard negative*)
dan merilis benchmark **SA-Co**. Mereka mengklaim akurasi SAM 3 **dua kali lipat** sistem yang ada pada tugas PCS.
README resmi menyebut SAM 3 mencapai 75-80% performa manusia pada SA-Co. Keduanya adalah klaim penulis dan belum saya verifikasi
secara independen.

**Pembaruan lanjutan.** Pada 27 Maret 2026 Meta merilis **SAM 3.1 (Object Multiplex)**. Ini memakai memori bersama
untuk melacak banyak objek sekaligus, dan menurut README lebih cepat tanpa mengorbankan akurasi.
Menurut pembacaan saya, ini menyasar kelemahan SAM 2 yang memproses tiap objek sendiri-sendiri.

### SAM 2 vs SAM 3 dalam sekilas

| Aspek | SAM 2 / 2.1 | SAM 3 |
|---|---|---|
| Bentuk prompt | Klik, kotak, mask (visual) | Frasa teks, contoh gambar, plus prompt visual |
| Keluaran | Satu objek per prompt | Semua instans dari satu konsep |
| Fokus tugas | Segmentasi dan pelacakan objek yang ditunjuk | Deteksi, segmentasi, pelacakan berbasis konsep |
| Ukuran | 38,9 juta sampai 224,4 juta parameter | sekitar 848 juta parameter |
| Lisensi (menurut repositori) | Apache 2.0 | SAM License |
| Akses checkpoint | Terbuka | Dibatasi: harus meminta akses di Hugging Face |
| Kebutuhan lingkungan (README) | Python 3.10+, PyTorch 2.5.1+ | Python 3.12+, PyTorch 2.7+, CUDA 12.6+ |

### Kenapa tutorial di bawah memakai SAM 2.1?

Pilihan yang disengaja. SAM 3 lebih baru, tetapi checkpoint-nya perlu persetujuan akses di Hugging Face
dan lingkungannya lebih berat. Untuk tutorial yang bisa dijalankan siapa saja langsung dari Colab gratis,
SAM 2.1 lebih praktis. Arsitektur tracker SAM 3 mewarisi SAM 2, jadi yang Anda pelajari di sini tetap relevan.
Kalau Anda sudah punya akses SAM 3, cukup ikuti README resminya untuk eksperimen lanjutan.

## 7. Tutorial: SAM 2.1 di Google Colab

Notebook lengkap ada di repositori ini: **[notebook/sam2_tutorial.ipynb](https://github.com/harisY/acv-sam2-blog/blob/main/notebook/sam2_tutorial.ipynb)**
dan bisa dibuka langsung di Colab:

[![Buka di Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/harisY/acv-sam2-blog/blob/main/notebook/sam2_tutorial.ipynb)

**Prasyarat:** akun Google, lalu pilih *Runtime → Change runtime type → T4 GPU*.

### Langkah 1: instal

```python
!pip install -q "git+https://github.com/facebookresearch/sam2.git"
!wget -q -P checkpoints https://dl.fbaipublicfiles.com/segment_anything_2/092824/sam2.1_hiera_tiny.pt
```

### Langkah 2: muat model

```python
import torch
from sam2.build_sam import build_sam2
from sam2.sam2_image_predictor import SAM2ImagePredictor

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
checkpoint = "checkpoints/sam2.1_hiera_tiny.pt"
model_cfg = "configs/sam2.1/sam2.1_hiera_t.yaml"

predictor = SAM2ImagePredictor(build_sam2(model_cfg, checkpoint, device=device))
```

### Langkah 3: segmentasi gambar dengan satu klik

```python
import numpy as np
from PIL import Image

image = np.array(Image.open("gambar.jpg").convert("RGB"))
predictor.set_image(image)

masks, scores, _ = predictor.predict(
    point_coords=np.array([[500, 375]]),   # (x, y) piksel yang diklik
    point_labels=np.array([1]),            # 1 = objek, 0 = latar
    multimask_output=True,                 # tiga kandidat mask
)
```

`multimask_output=True` menghasilkan tiga kandidat dengan skor kepercayaan. Pilih yang skornya tertinggi,
atau tambahkan klik negatif (label 0) untuk menyingkirkan area yang salah.

### Langkah 4: segmentasi dan pelacakan video

```python
from sam2.build_sam import build_sam2_video_predictor

video_predictor = build_sam2_video_predictor(model_cfg, checkpoint, device=device)
state = video_predictor.init_state(video_path="bedroom")    # folder berisi 00000.jpg, 00001.jpg, ...

video_predictor.add_new_points_or_box(
    inference_state=state, frame_idx=0, obj_id=1,
    box=np.array([340, 10, 540, 470], dtype=np.float32),   # (x1, y1, x2, y2) di sekeliling anak perempuan
)

hasil = {}
for frame_idx, obj_ids, mask_logits in video_predictor.propagate_in_video(state):
    hasil[frame_idx] = (mask_logits[0] > 0.0).cpu().numpy()
```

Notebook memakai klip contoh `bedroom` dari aset resmi SAM 2, jadi bisa langsung dijalankan. Untuk video Anda sendiri,
ubah dulu menjadi kumpulan frame JPEG dan sesuaikan koordinat kotak, misalnya dengan
`ffmpeg -i video.mp4 -q:v 2 -start_number 0 frames/%05d.jpg`.

### Hasil percobaan

Saya menjalankan kode di atas pada mesin tanpa GPU (hanya CPU, PyTorch 2.14, checkpoint `sam2.1_hiera_tiny`), dengan gambar `truck.jpg` dan klip `bedroom` dari repositori resmi SAM 2.

**Gambar: satu klik, tiga kandidat mask.**

![Tiga kandidat mask SAM 2.1 untuk satu klik pada jendela truk]({{ '/assets/img/hasil-gambar.png' | relative_url }})

Satu klik di kaca jendela truk menghasilkan tiga kandidat dengan skor 0,952, 0,493, dan 0,042. Ketiganya sah secara visual:
kandidat pertama hanya sebagian kaca, kandidat kedua seluruh jendela, dan kandidat ketiga hampir seluruh badan truk.
Ini memperlihatkan **ambiguitas prompt**: satu titik bisa berarti bagian, benda, atau benda induknya.
Menariknya, skor tertinggi tidak selalu sama dengan yang Anda maksud. Skor mencerminkan kepercayaan model terhadap kualitas mask, bukan niat pengguna.

**Video: satu kotak, 200 frame.**

![Mask SAM 2.1 mengikuti anak perempuan sepanjang klip]({{ '/assets/img/hasil-video.png' | relative_url }})

Prompt berupa satu kotak di frame pertama cukup untuk melacak anak perempuan di seluruh 200 frame (tiap panel menunjukkan satu frame).
Pada frame 120 dan 160 ia berada dekat anak laki-laki, dan mask tetap menempel pada anak perempuan, tidak berpindah ke anak laki-laki. Ini gambaran kerja memory attention, tetapi bukan pengujian oklusi yang ketat.
Mask juga agak "bocor" ke area kecil di sekitar kaki di beberapa frame, tanda bahwa model `tiny` punya batas.

**Pengamatan yang berguna:**

- **Klik vs kotak.** Pada frame pertama klip ini, satu klik di badan anak hanya menghasilkan sebagian pakaian atau bagian tubuh,
  bukan seluruh orang. Kotak langsung menghasilkan mask yang utuh. Untuk objek yang bentuknya rumit, kotak atau beberapa klik lebih andal.
- **Kecepatan.** Di CPU, melacak 200 frame dengan model `tiny` memakan sekitar 12 sampai 21 menit pada dua kali percobaan saya
  (kira-kira 3,5 sampai 6 detik per frame). Angka FPS di paper dan README (puluhan FPS) diukur di GPU A100, jadi jangan dibandingkan langsung.
  Di Colab dengan GPU T4 prosesnya jauh lebih cepat.
- **Percobaan ini terbatas:** satu gambar, satu klip, satu ukuran model, satu kali tracking per konfigurasi, tanpa metrik kuantitatif.
  Ini demonstrasi kualitatif, bukan evaluasi.

Eksperimen lanjutan yang layak Anda coba sendiri:

- Bandingkan ukuran model `tiny` dan `large`: waktu inferensi dan kualitas mask.
- Bandingkan satu klik dengan beberapa klik (positif dan negatif) pada objek yang bentuknya rumit.
- Uji objek tipis atau transparan, yang menurut paper termasuk kelemahan model.
- Pada video, beri prompt koreksi di frame tengah dan lihat apakah pelacakan membaik.

## 8. Ide riset untuk tesis

Bagian ini paling berguna bagi yang sedang mencari topik. Setiap ide diturunkan dari kelemahan di bagian 5.

1. **Adaptasi domain medis.** SAM dilatih pada citra umum. Citra CT, MRI, dan USG punya kontras serta tekstur yang berbeda.
   Pertanyaan riset: seberapa besar penurunan performa pada data medis, dan apakah *fine-tuning* ringan (misalnya LoRA atau *adapter*)
   cukup? Untuk volume 3D, setiap slice bisa diperlakukan sebagai "frame" dan memori SAM 2 dipakai untuk konsistensi antarslice.
   Cari literatur terkait dengan kata kunci "SAM 2 medical image segmentation".
2. **Citra satelit dan drone (remote sensing).** Objek kecil, padat, dan seragam menyulitkan model. Bandingkan prompt titik,
   kotak, dan teks (SAM 3) untuk pemetaan bangunan atau lahan pertanian.
3. **Efisiensi untuk perangkat tepi.** Varian `tiny` sudah ringan. Eksplorasi kuantisasi atau distilasi untuk Jetson atau ponsel,
   dengan metrik J&F terhadap FPS dan konsumsi memori.
4. **Pelacakan objek serupa dan oklusi panjang.** Kelemahan yang diakui penulis. Uji pada video lalu lintas atau olahraga,
   lalu usulkan modul pembeda identitas, misalnya fitur penampilan tambahan.
5. **Anotasi semi-otomatis.** Terapkan pola *data engine* di domain Anda: model menyarankan mask, manusia hanya mengoreksi.
   Ukur penghematan waktu seperti tabel di bagian 4.
6. **Evaluasi lintas prompt.** Bandingkan klik, kotak, mask, dan teks secara sistematis pada dataset Anda. Banyak paper hanya melaporkan satu jenis prompt.

Kalau memilih salah satu, mulailah dari *baseline* zero-shot (tanpa pelatihan ulang) untuk melihat seberapa besar celahnya.

## 9. Penutup

SAM 2 memperlihatkan bahwa satu model bisa menangani gambar dan video dengan memori sederhana, dan SAM 3 membawa arah berikutnya:
dari "tunjuk objeknya" menjadi "sebutkan konsepnya". Tren ini penting untuk riset karena membuat segmentasi lebih mudah dipakai tanpa data berlabel
dalam jumlah besar. Meski begitu, kelemahan pada oklusi panjang, objek serupa, dan pergeseran domain masih terbuka, dan di situlah ruang kontribusi tesis.

Kalau ada koreksi atau Anda sudah mencoba ide di atas, silakan beri tahu saya lewat kolom komentar atau issue di repositori.

## Daftar pustaka

1. N. Ravi, V. Gabeur, Y.-T. Hu, R. Hu, C. Ryali, T. Ma, H. Khedr, R. Rädle, C. Rolland, L. Gustafson, E. Mintun, J. Pan, K. V. Alwala, N. Carion, C.-Y. Wu, R. Girshick, P. Dollár, dan C. Feichtenhofer, "SAM 2: Segment Anything in Images and Videos," *International Conference on Learning Representations (ICLR)*, 2025. arXiv:2408.00714. <https://arxiv.org/abs/2408.00714>
2. N. Carion, L. Gustafson, dkk., "SAM 3: Segment Anything with Concepts," arXiv:2511.16719, 2025. <https://arxiv.org/abs/2511.16719>
3. Meta AI (facebookresearch), repositori resmi SAM 2. <https://github.com/facebookresearch/sam2>
4. Meta AI (facebookresearch), repositori resmi SAM 3. <https://github.com/facebookresearch/sam3>

*Semua angka di atas diambil dari sumber di daftar pustaka dan diparafrase. Gambar arsitektur dibuat sendiri oleh penulis.*
