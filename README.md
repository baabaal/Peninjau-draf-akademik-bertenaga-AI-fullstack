# Peninjau-draf-akademik-bertenaga-AI-fullstack

# Scholar Review — Peninjau Draf Akademik Bertenaga AI

Aplikasi web fullstack (MERN) untuk meninjau draf tesis/disertasi (PDF/DOCX) secara otomatis menggunakan Claude. Hasilnya berupa **Skor Kesiapan (0–100)**, rincian empat kriteria akademik, dan **tabel ulasan per bagian**, lengkap dengan ekspor laporan **DOCX/PDF**.

Tumpukan: **React (Vite) · Node.js + Express · MongoDB (Mongoose) · LangChain + Anthropic**.

---

## 1. Prasyarat (CachyOS / Arch)

```bash
# Node.js 20 + npm
sudo pacman -S nodejs npm

# MongoDB (dari AUR; prebuilt binary)
paru -S mongodb-bin
sudo systemctl enable --now mongodb

# git (bila belum ada)
sudo pacman -S git
```

> **Alternatif database:** pakai **MongoDB Atlas** (free tier) dan tempel connection string-nya ke `MONGODB_URI`. Lewati instalasi `mongodb-bin`.

Anda juga butuh **kunci API Anthropic** dari https://console.anthropic.com (menu *API Keys*).

---

## 2. Menjalankan secara lokal

### Backend

```bash
cd server
cp .env.example .env        # lalu isi ANTHROPIC_API_KEY
npm install
npm run dev                 # API: http://localhost:5000
```

Isi minimal `server/.env`:

```bash
PORT=5000
MONGODB_URI=mongodb://127.0.0.1:27017/scholar_review
ANTHROPIC_API_KEY=sk-ant-xxxxxxxxxxxx
ANTHROPIC_MODEL=claude-sonnet-4-6
CLIENT_URL=http://localhost:5173
MAX_FILE_MB=25
NODE_ENV=development
```

### Frontend (terminal baru)

```bash
cd client
npm install
npm run dev                 # UI: http://localhost:5173
```

Buka **http://localhost:5173**, unggah PDF/DOCX, tunggu hasilnya, lalu ekspor laporan.

> Vite sudah mem-*proxy* `/api` ke `http://localhost:5000`, jadi tidak ada masalah CORS saat pengembangan lokal.

---

## 3. Struktur proyek

```
scholar-review/
├── client/                    # React (Vite) + Tailwind + Framer Motion
│   ├── index.html
│   ├── vite.config.js         # proxy /api -> :5000
│   ├── tailwind.config.js     # aksen biru-slate (ganti ke emerald utk sage)
│   └── src/
│       ├── App.jsx            # routing: / dan /hasil/:id
│       ├── lib/api.js         # axios (baseURL: /api)
│       ├── components/        # ScoreRing, ReviewTable, LoadingOverlay
│       └── pages/             # UploadPage, ResultsPage
│
└── server/                    # Node.js + Express
    ├── .env.example
    └── src/
        ├── server.js          # entry
        ├── config/db.js
        ├── models/Review.js
        ├── middleware/upload.js        # Multer (PDF/DOCX, maks 25MB)
        ├── services/
        │   ├── extractText.js          # pdf-parse + mammoth
        │   ├── chunk.js                # pemecah teks
        │   ├── reviewService.js        # LangChain + Claude + Zod (JSON ketat)
        │   └── reportService.js        # ekspor DOCX & PDF
        ├── controllers/reviewController.js
        └── routes/reviewRoutes.js
```

### Endpoint API

| Metode | Rute | Fungsi |
|---|---|---|
| `POST` | `/api/reviews/analyze` | Unggah file (`field: file`) → kembalikan ulasan JSON |
| `GET`  | `/api/reviews/:id` | Ambil ulasan tersimpan |
| `GET`  | `/api/reviews/:id/export?format=docx\|pdf` | Unduh laporan |
| `GET`  | `/api/health` | Health check |

---

## 4. Cara kerja AI

`reviewService.js` memanggil Claude lewat LangChain dengan `withStructuredOutput(ReviewSchema)` (skema Zod), sehingga **keluaran dijamin berupa JSON valid** yang langsung dipetakan ke kartu skor & tabel — tanpa *parsing* string yang rapuh.

- **Model default:** `claude-sonnet-4-6` (seimbang biaya/kualitas). Ganti via `ANTHROPIC_MODEL`.
- **Telaah paling tajam:** set `ANTHROPIC_MODEL=claude-opus-4-8`.
- **Dokumen panjang:** bila melewati ambang token, dokumen dipecah lalu diringkas (`claude-haiku-4-5-20251001`) sebelum telaah final (*map-reduce*) untuk menekan biaya.

Ingin pakai **OpenAI**? Pasang `@langchain/openai`, lalu ganti inisialisasi model di `reviewService.js` dengan `ChatOpenAI` — `withStructuredOutput` tetap sama.

---

## 5. Menjalankan via Docker (opsional)

```bash
export ANTHROPIC_API_KEY=sk-ant-xxxx
docker compose up -d --build
# Aplikasi: http://localhost:8080
```

Compose menyalakan tiga layanan: `mongo`, `server`, dan `client` (Nginx yang menyajikan build React dan mem-proxy `/api` ke backend).

---

## 6. Pemecahan masalah

- **`npm install` gagal di `pdf-parse`** — jalankan `npm install --ignore-scripts` (kode sudah mengimpor `pdf-parse/lib/pdf-parse.js` agar tidak memicu mode debug yang error).
- **`MongoNetworkError` / koneksi gagal** — pastikan `sudo systemctl status mongodb` *active*, atau periksa `MONGODB_URI` (Atlas: pastikan IP Anda diizinkan).
- **`401`/`authentication_error` dari Claude** — `ANTHROPIC_API_KEY` salah/kosong di `server/.env`.
- **Unggahan ditolak** — hanya PDF & DOCX, maks. 25 MB (ubah `MAX_FILE_MB`).
- **CORS saat produksi** — set `CLIENT_URL` di backend agar cocok dengan domain frontend.

---

## 7. Lisensi & catatan

Proyek contoh untuk keperluan internal/edukasi. Biaya API Anthropic sebanding dengan panjang dokumen; mulai dari Sonnet dan naikkan model hanya bila perlu. Jangan pernah meng-*commit* file `.env`.
