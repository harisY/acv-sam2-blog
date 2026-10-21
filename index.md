---
layout: post
title: "Dari “Klik Objeknya” ke “Sebut Namanya”: Perjalanan SAM 2 menuju SAM 3"
author: "Abdul Haris"
date: 2026-10-21
description: "Review paper SAM 2 (ICLR 2025) dan SAM 3, lengkap dengan tutorial yang bisa kamu jalankan sendiri dan ide riset untuk tesis."
---

*Review paper, tutorial praktik, dan ide riset untuk teman-teman S2 yang sedang mencari topik. Ditulis oleh **Abdul Haris**, mahasiswa S2 TI UNPAM, untuk mata kuliah Advanced Computer Vision.*

---

**Kalau kamu cuma punya 30 detik:**

- **SAM 2** membuat segmentasi “klik apa saja” yang dulu hanya untuk gambar bisa bekerja di **video**, berkat memori antarframe.
- **SAM 3** melangkah lebih jauh: kamu tidak perlu menunjuk objek. Cukup **sebut namanya**, dan model menemukan semua yang cocok.
- Ada **tutorial yang bisa kamu jalankan sendiri** di bawah, plus **enam ide riset** untuk tesis.

---

## Bayangkan kamu harus menandai 10.000 gambar

Misalnya tesismu tentang citra medis. Kamu butuh mask yang menandai persis di mana tumor berada, piksel demi piksel. Satu gambar saja sudah memakan waktu, dan itu baru satu dari ribuan.

Inilah alasan model segmentasi klasik terasa berat. Setiap domain butuh data berlabel sendiri, dan melabelinya mahal.

Lalu Meta merilis **Segment Anything Model (SAM)**, dengan gagasan yang sederhana: latih sekali pada data yang sangat besar, lalu **pandu lewat prompt**. Kamu cukup mengklik, menggambar kotak, atau memberi sketsa mask, dan model menyegmentasi objek apa pun yang kamu maksud. Mirip cara kita mengarahkan LLM dengan teks.

Tulisan ini mengikuti kelanjutannya:

> **SAM 2 (2024)** menambahkan dimensi *waktu*. **SAM 3 (2025)** menambahkan *bahasa*.

Paper SAM 2 dipresentasikan di **ICLR 2025** (sesi oral). Untuk SAM 3 saya merujuk preprint di arXiv, karena saya tidak menemukan informasi venue publikasinya.

## SAM 2: ketika segmentasi belajar mengingat

SAM pertama bekerja pada satu gambar. Kalau kamu ingin melacak sebuah objek di video, kamu harus mengulang prosesnya di setiap frame. Prosesnya bisa lambat, dan hasilnya mudah tidak konsisten dari frame ke frame.

Tim Meta (Ravi dkk., 2025) menjawabnya dengan tiga kontribusi.

**1. Tugas baru bernama *Promptable Visual Segmentation* (PVS).** Kamu memberi prompt di frame mana pun (klik positif atau negatif, kotak, atau mask), dan model menyebarkannya menjadi *masklet*: mask objek yang konsisten sepanjang video. Kalau hasilnya meleset, beri koreksi di frame lain dan model memperbaiki sisanya.

**2. Model dengan memori.** Video diproses frame demi frame sambil membawa ingatan tentang objek, jadi cocok untuk pemrosesan real-time.

**3. Data engine dan dataset SA-V.** Mereka membangun dataset video segmentasi terbesar pada masanya, dengan cara yang cerdik. Saya jelaskan di bawah.

### Cara kerja, tanpa jargon berlebihan

![Diagram sederhana arsitektur SAM 2]({{ '/assets/img/sam2-arsitektur.svg' | relative_url }})

Untuk setiap frame, alurnya begini:

1. **Image encoder** (Hiera, dilatih awal dengan MAE) mengubah frame menjadi fitur.
2. **Memory attention** membuat fitur itu “melihat” memori dari frame-frame sebelumnya. Ini bagian terpentingnya.
3. **Mask decoder** menggabungkan fitur dan prompt, lalu menghasilkan mask. Ada tambahan *head* yang memprediksi apakah objek **sedang tidak terlihat** (tertutup).
4. **Memory encoder** mengemas mask yang baru saja dihasilkan menjadi memori, lalu disimpan di **memory bank**: antrean beberapa frame terakhir, frame yang pernah diberi prompt, dan *object pointer* yang membawa informasi semantik objek.

Karena ada memori, objek yang sempat tertutup lalu muncul lagi bisa tetap dikenali. SAM pertama tidak punya kemampuan itu.

### Cerdiknya data engine

Dataset video segmentasi itu mahal, jadi mereka membuatnya bertahap, dengan model yang ikut membantu anotator manusia. Waktu anotasi per frame turun drastis:

- **Fase 1** (SAM per frame): 37,8 detik per frame
- **Fase 2** (SAM + SAM 2 Mask): 7,4 detik per frame, sekitar 5,1 kali lebih cepat
- **Fase 3** (SAM 2 penuh): 4,5 detik per frame, sekitar 8,4 kali lebih cepat

Hasil akhirnya, **SA-V**, berisi 50,9 ribu video, 642,6 ribu masklet, dan 35,5 juta mask, direkam di 47 negara. Penulis menyebut jumlah mask-nya 53 kali lipat dataset VOS yang ada sebelumnya.

> **Pelajaran untuk risetmu:** pola *model-in-the-loop* ini bisa kamu tiru. Latih model awal, biarkan ia menyarankan label, lalu manusia cukup mengoreksi. Biaya anotasi di domainmu sendiri bisa turun jauh.

### Seberapa bagus hasilnya?

Menurut penulis (semua angka berikut diambil dari paper):

- **Gambar:** SAM 2 sekitar **6 kali lebih cepat** daripada SAM (130,1 vs 21,7 FPS di A100, batch 10 gambar), dengan akurasi sedikit lebih baik (mIoU 1 klik pada 23 dataset, dengan data latih yang sama, SA-1B: 58,9 vs 58,1).
- **Video:** dengan prompt 3 klik di frame pertama pada 17 dataset zero-shot, SAM 2 (varian B+) mencapai **75,3 J&F**, mengungguli baseline SAM + Cutie (70,1) dan SAM + XMem++ (68,4). Secara umum penulis mengklaim akurasi lebih baik dengan interaksi sekitar **3 kali lebih sedikit**.
- **Kecepatan video:** 43,8 FPS (varian B+) dan 30,2 FPS (varian L) di satu GPU A100.

### Yang tidak dipamerkan di abstrak

Saya suka bahwa penulisnya jujur soal kelemahan (Lampiran C). Model bisa gagal ketika:

- terjadi **pergantian shot** (potongan kamera),
- adegan **ramai**, ada **oklusi panjang**, atau video **sangat panjang**,
- objeknya **tipis dan bergerak cepat**,
- ada **objek serupa yang berdekatan** (misalnya bola juggling yang identik),
- ada banyak objek, karena **tiap objek diproses sendiri-sendiri** tanpa saling berkomunikasi.

Catat daftar ini. Setiap poinnya adalah calon topik tesis. Kita kembali ke sini di bagian akhir.

### Catatan soal SAM 2.1

Setelah paper terbit, Meta merilis **SAM 2.1** (30 September 2024) dengan checkpoint yang diperbarui. Berikut ukuran-ukurannya, dari README resmi (kecepatan diukur di A100, PyTorch 2.5.1):

- `hiera_tiny`: 38,9 juta parameter, 91,2 FPS, 76,5 J&F di SA-V test
- `hiera_small`: 46 juta, 84,8 FPS, 76,6 J&F
- `hiera_base_plus`: 80,8 juta, 64,1 FPS, 78,2 J&F
- `hiera_large`: 224,4 juta, 39,5 FPS, 79,5 J&F

Angka FPS ini diukur dengan pengaturan berbeda dari FPS di paper, jadi jangan dibandingkan langsung. Untuk belajar dan bereksperimen, `tiny` sudah cukup dan paling ringan.

## SAM 3: dari “ini objeknya” menjadi “cari semua mobil merah”

Ada satu hal yang mengganjal di SAM 1 dan 2: kamu harus **menunjuk satu objek**. Padahal sering kali kebutuhan kita lain: “tandai semua bus sekolah di video ini.”

**SAM 3** (*Segment Anything with Concepts*, Carion dkk., 2025) menjawabnya dengan tugas baru, *Promptable Concept Segmentation*. Kamu memberi **frasa kata benda pendek** atau **contoh gambar**, dan model mendeteksi, menyegmentasi, dan melacak **semua instans** konsep itu, lengkap dengan identitas unik tiap instans.

Beberapa hal teknis yang menarik, menurut paper dan README resminya:

- Ada **detektor** berbasis DETR untuk gambar dan **tracker** berbasis memori yang mewarisi arsitektur SAM 2. Keduanya berbagi satu *vision encoder*.
- Ada **presence head** yang memisahkan pertanyaan “apakah konsep ini ada di gambar?” dari “di mana tepatnya?”. Menurut penulis, pemisahan ini meningkatkan akurasi deteksi.
- Ukuran modelnya sekitar **848 juta parameter**.
- Datanya berasal dari data engine dengan sekitar 4 juta label konsep unik (termasuk *hard negative*), disertai benchmark baru **SA-Co**.

Klaim penulis: akurasi SAM 3 **dua kali lipat** sistem yang ada pada tugas ini, dan README menyebut 75 sampai 80 persen performa manusia di SA-Co. Keduanya klaim penulis dan belum saya verifikasi sendiri.

Pada 27 Maret 2026 Meta juga merilis **SAM 3.1 (Object Multiplex)**, yang memakai memori bersama untuk melacak banyak objek sekaligus dan menurut README lebih cepat tanpa mengorbankan akurasi. Dari pembacaan saya, ini menyasar salah satu kelemahan SAM 2 tadi: objek yang diproses sendiri-sendiri.

### SAM 2 atau SAM 3, mana yang dipakai?

**SAM 2.1** terbuka (lisensi Apache 2.0), butuh Python 3.10+ dan PyTorch 2.5.1+, dan paling kecil hanya puluhan juta parameter. **SAM 3** punya lisensi SAM License, butuh Python 3.12+, PyTorch 2.7+, dan CUDA 12.6+, dan **checkpoint-nya harus diminta aksesnya** di Hugging Face.

Karena itu tutorial di bawah memakai **SAM 2.1**: tanpa permintaan akses, dan cukup ringan untuk GPU Colab gratis. Tracker SAM 3 mewarisi arsitektur SAM 2, jadi yang kamu pelajari tetap berguna. Kalau kamu sudah punya akses SAM 3, ikuti README resminya untuk lanjut.

## Tutorial: SAM 2.1 dalam lima langkah

Notebook lengkapnya ada di repositori ini, dan bisa dibuka langsung di Colab (pilih *Runtime → Change runtime type → T4 GPU*):

[![Buka di Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/harisY/acv-sam2-blog/blob/main/notebook/sam2_tutorial.ipynb)

**Langkah 1: instal, lalu unduh checkpoint dan contoh data** (gambar truk dan klip video `bedroom` dari repositori resmi).

```python
!pip install -q "git+https://github.com/facebookresearch/sam2.git"
!mkdir -p checkpoints
!wget -q -P checkpoints https://dl.fbaipublicfiles.com/segment_anything_2/092824/sam2.1_hiera_tiny.pt
!wget -q -O truck.jpg https://raw.githubusercontent.com/facebookresearch/sam2/main/notebooks/images/truck.jpg
!wget -q -O bedroom.zip https://dl.fbaipublicfiles.com/segment_anything_2/assets/bedroom.zip
!unzip -q -o bedroom.zip
```

**Langkah 2: muat model.**

```python
import torch
from sam2.build_sam import build_sam2
from sam2.sam2_image_predictor import SAM2ImagePredictor

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
checkpoint = "checkpoints/sam2.1_hiera_tiny.pt"
model_cfg = "configs/sam2.1/sam2.1_hiera_t.yaml"

predictor = SAM2ImagePredictor(build_sam2(model_cfg, checkpoint, device=device))
```

**Langkah 3: segmentasi gambar dengan satu klik.**

```python
import numpy as np
from PIL import Image

image = np.array(Image.open("truck.jpg").convert("RGB"))
predictor.set_image(image)

masks, scores, _ = predictor.predict(
    point_coords=np.array([[500, 375]]),   # (x, y) piksel yang diklik (kaca jendela truk)
    point_labels=np.array([1]),            # 1 = objek, 0 = latar
    multimask_output=True,                 # tiga kandidat mask
)
```

**Langkah 4: lacak objek di video.** Kode ini memakai folder frame JPEG `bedroom` dari langkah 1. Untuk videomu sendiri, ubah dulu menjadi frame dengan `ffmpeg -i video.mp4 -q:v 2 -start_number 0 frames/%05d.jpg`, lalu ganti `video_path` dan koordinat kotaknya.

```python
from sam2.build_sam import build_sam2_video_predictor

video_predictor = build_sam2_video_predictor(model_cfg, checkpoint, device=device)
state = video_predictor.init_state(video_path="bedroom")

video_predictor.add_new_points_or_box(
    inference_state=state, frame_idx=0, obj_id=1,
    box=np.array([340, 10, 540, 470], dtype=np.float32),   # (x1, y1, x2, y2) di sekeliling anak perempuan
)

hasil = {}
for frame_idx, obj_ids, mask_logits in video_predictor.propagate_in_video(state):
    hasil[frame_idx] = (mask_logits[0] > 0.0).cpu().numpy()
```

**Langkah 5: bereksperimen.** Notebook di repositori sudah menyatukan semua langkah di atas, lengkap dengan visualisasi hasilnya.

### Apa yang saya temukan saat menjalankannya

Saya menjalankan kode ini di komputer lokal tanpa GPU (CPU saja, PyTorch 2.14, model `tiny`), bukan di Colab, pada gambar `truck.jpg` dan klip `bedroom` dari repositori resmi. Karena itu angka waktu di bawah tidak mewakili Colab.

**Satu klik, tiga kemungkinan jawaban.**

![Tiga kandidat mask SAM 2.1 untuk satu klik pada jendela truk]({{ '/assets/img/hasil-gambar.png' | relative_url }})

Saya mengklik kaca jendela truk. Model mengembalikan tiga kandidat dengan skor 0,952, 0,493, dan 0,042: sebagian jendela, seluruh jendela, dan hampir seluruh badan truk. Ketiganya masuk akal, karena satu titik memang **ambigu**: kamu menunjuk bagian, benda, atau benda induknya?

Yang menarik, skor tertinggi tidak selalu sama dengan yang kamu maksud. Skor mencerminkan keyakinan model terhadap kualitas mask, bukan niatmu.

**Satu kotak, dua ratus frame.**

![Mask SAM 2.1 mengikuti anak perempuan sepanjang klip]({{ '/assets/img/hasil-video.png' | relative_url }})

Satu kotak di frame pertama cukup untuk melacak anak perempuan di seluruh 200 frame. Di frame 120 dan 160 ia berada dekat anak laki-laki, dan mask tetap menempel padanya, tidak berpindah. Ini gambaran kerja memory attention, tapi bukan uji oklusi yang ketat. Di beberapa frame mask juga sedikit bocor ke area sekitar kaki, tanda bahwa model `tiny` punya batas.

**Dua catatan praktis:**

- **Klik vs kotak.** Pada beberapa titik klik yang saya coba di frame pertama klip ini, hasilnya hanya sebagian pakaian atau tubuh, atau malah meluas ke area besar di latar. Satu kotak langsung menghasilkan mask yang utuh. Untuk objek berbentuk rumit, pakai kotak atau beberapa klik.
- **Kecepatan.** Di CPU, melacak 200 frame membutuhkan sekitar 12 sampai 21 menit pada dua kali percobaan saya (sekitar 3,5 sampai 6,5 detik per frame). Angka puluhan FPS di paper diukur di GPU A100, jadi jangan dibandingkan langsung. Dengan GPU, waktunya umumnya jauh lebih singkat.

Satu catatan jujur: ini hanya demonstrasi kualitatif (satu gambar, satu klip, satu ukuran model), bukan evaluasi dengan metrik.

**Coba sendiri:** bandingkan `tiny` dengan `large`, satu klik dengan beberapa klik, dan uji objek tipis atau transparan. Itu titik lemah yang diakui paper.

## Enam ide riset untuk tesis

Setiap ide di bawah lahir dari kelemahan yang tadi kita catat.

1. **Adaptasi ke citra medis.** SAM dilatih di citra umum, sedangkan CT, MRI, dan USG punya kontras dan tekstur berbeda. Seberapa besar performanya turun, dan apakah *fine-tuning* ringan (LoRA atau *adapter*) cukup? Untuk volume 3D, setiap slice bisa diperlakukan sebagai “frame” sehingga memori SAM 2 menjaga konsistensi antarslice. Cari literatur dengan kata kunci *SAM 2 medical image segmentation*.
2. **Citra satelit dan drone.** Objek kecil, padat, dan seragam menyulitkan model. Bandingkan prompt titik, kotak, dan teks (SAM 3) untuk memetakan bangunan atau lahan pertanian.
3. **Efisiensi di perangkat tepi.** Varian `tiny` sudah ringan. Jajaki kuantisasi atau distilasi untuk Jetson atau ponsel, dengan metrik J&F terhadap FPS dan memori.
4. **Objek serupa dan oklusi panjang.** Kelemahan yang diakui penulis. Uji di video lalu lintas atau olahraga, lalu usulkan modul pembeda identitas, misalnya fitur penampilan tambahan.
5. **Anotasi semi-otomatis.** Terapkan pola data engine di domainmu: model menyarankan, manusia mengoreksi. Ukur penghematan waktunya, seperti angka per fase tadi.
6. **Evaluasi lintas jenis prompt.** Bandingkan klik, kotak, mask, dan teks secara sistematis di datasetmu, lalu lihat jenis prompt mana yang paling efisien untuk domain tersebut.

Apa pun yang kamu pilih, mulai dari **baseline zero-shot** (tanpa pelatihan ulang) untuk melihat seberapa besar celahnya.

## Penutup

SAM 2 menunjukkan bahwa satu model bisa menangani gambar dan video dengan memori yang sederhana. SAM 3 menunjukkan arah berikutnya: dari “tunjuk objeknya” menjadi “sebutkan konsepnya”. Arah ini penting buat kita yang sedang riset, karena segmentasi jadi lebih mudah dipakai tanpa data berlabel dalam jumlah besar.

Namun celahnya masih terbuka lebar: oklusi panjang, objek kembar, dan pergeseran domain. Di situlah ruang kontribusi tesismu.

Kalau kamu mencoba salah satu ide di atas, atau menemukan kesalahan di tulisan ini, kabari saya lewat *issue* di repositori. Selamat bereksperimen!

---

### Daftar pustaka

1. N. Ravi, V. Gabeur, Y.-T. Hu, R. Hu, C. Ryali, T. Ma, H. Khedr, R. Rädle, C. Rolland, L. Gustafson, E. Mintun, J. Pan, K. V. Alwala, N. Carion, C.-Y. Wu, R. Girshick, P. Dollár, dan C. Feichtenhofer, “SAM 2: Segment Anything in Images and Videos,” *International Conference on Learning Representations (ICLR)*, 2025. arXiv:2408.00714. <https://arxiv.org/abs/2408.00714>
2. N. Carion, L. Gustafson, dkk., “SAM 3: Segment Anything with Concepts,” arXiv:2511.16719, 2025. <https://arxiv.org/abs/2511.16719>
3. Meta AI (facebookresearch), repositori resmi SAM 2. <https://github.com/facebookresearch/sam2>
4. Meta AI (facebookresearch), repositori resmi SAM 3. <https://github.com/facebookresearch/sam3>

*Semua angka diambil dari sumber di atas dan diparafrase. Diagram arsitektur dibuat sendiri oleh penulis. Hasil percobaan berasal dari run lokal penulis.*
