# Modul 4 — OpenRouter & Model AI: Token, Context, dan Dunia LLM

> Bagian dari: Kelas Agentic AI Automation — Kelas Otomesyen  
> Dikerjakan oleh: Rizal Wahyu Pratama  
> Tanggal: 30 Agustus 2026  
> Durasi pengerjaan: ~1.5 jam  
> Prasyarat: Modul 3 (HTTP Requests)

---

## Daftar Isi

- [Ringkasan Modul](#ringkasan-modul)
- [Konsep Dasar](#konsep-dasar)
  - [OpenRouter](#openrouter)
  - [Token](#token)
  - [Context Window](#context-window)
  - [Temperature & Parameter](#temperature--parameter)
- [Setup OpenRouter](#setup-openrouter)
- [Hasil Praktik](#hasil-praktik)
  - [PR 1 — First API Call](#pr-1--first-api-call)
  - [PR 2 — Perbandingan 3 Model](#pr-2--perbandingan-3-model)
  - [PR 3 — Hitung Token](#pr-3--hitung-token)
  - [PR 4 — Chatbot Multi-Turn](#pr-4--chatbot-multi-turn)
  - [PR 5 — Experiment Temperature](#pr-5--experiment-temperature)
- [Catatan Masalah & Solusi](#catatan-masalah--solusi)
- [Kesimpulan](#kesimpulan)

---

## Ringkasan Modul

Modul ini membahas cara berkomunikasi langsung dengan model AI melalui HTTP Request ke OpenRouter. Tujuannya adalah memahami apa yang sebenarnya terjadi di balik layar sebelum menggunakan AI Agent node di n8n.

Yang dipelajari:
- Cara kerja OpenRouter sebagai aggregator model AI
- Konsep token, context window, dan pricing
- Parameter seperti temperature dan max_tokens
- Cara call API OpenRouter via HTTP Request node di n8n
- Simulasi chatbot multi-turn dengan history percakapan

---

## Konsep Dasar

### OpenRouter

OpenRouter adalah **aggregator** — satu API key untuk mengakses ratusan model AI dari berbagai provider (OpenAI, Anthropic, Google, Meta, Mistral, dll).

```
Tanpa OpenRouter:  daftar OpenAI + daftar Anthropic + daftar Google + ...
Dengan OpenRouter: 1 akun, 1 API key, akses semua
```

**Endpoint utama:**
```
POST https://openrouter.ai/api/v1/chat/completions
```

**Header yang wajib ada:**
```
Authorization: Bearer sk-or-v1-xxxxxxxxxx
Content-Type: application/json
```

**Model gratis yang direkomendasikan (per Agustus 2026):**

| Model | Kategori | Catatan |
|---|---|---|
| `inclusionai/ling-3.0-flash-fin:free` | General / Finance | 262K context, stabil |
| `poolside/laguna-xs-2.1:free` | General | Cepat, gratis |
| `dots-studio/dots-3-note-preview:free` | General | Ringan |
| `qwen/qwen3-14b:free` | General / Fast | Dari Alibaba |
| `deepseek/deepseek-chat-v3-0324:free` | Code / Chat | Populer |

> Catatan: daftar model berubah sewaktu-waktu. Cek https://openrouter.ai/models untuk update terbaru. Model yang di modul seperti `google/gemma-3-27b-it:free` sudah tidak tersedia gratis saat praktik ini dilakukan.

---

### Token

Token adalah unit dasar cara AI membaca dan menghasilkan teks. Bukan per kata, tapi per **chunk karakter**.

```
Kalimat: "Halo, saya suka makan nasi goreng"
Token:   ["Halo", ",", " saya", " suka", " makan", " nasi", " gore", "ng"]
         = 8 tokens
```

**Perbandingan efisiensi bahasa:**

| Bahasa | Estimasi |
|---|---|
| English | ~1 kata = 1.3 token |
| Indonesia | ~1 kata = 2 token |
| Kode program | bervariasi, cenderung boros |

**Input vs Output tokens:**
- Input tokens = semua yang kita kirim (system prompt + history + pesan baru)
- Output tokens = response AI
- Output biasanya lebih mahal (contoh GPT-4o: input $2.50/1M, output $10/1M)

**Kenapa penting:**
1. Biaya dihitung per token
2. Setiap model punya batas total token (context window)
3. Lebih banyak token = response lebih lambat

---

### Context Window

Context window adalah jumlah maksimum token yang bisa diproses model dalam satu request. Analoginya seperti RAM — kalau penuh, yang paling lama dihapus.

```
Yang masuk context window:
  system prompt + semua history chat + pesan user sekarang + response AI
```

**Contoh limit per model:**

| Model | Context Window |
|---|---|
| Llama 3 8B | 8K tokens (~6 hal. A4) |
| GPT-4o-mini | 128K tokens (~100 hal. A4) |
| Gemini 2.0 Flash | 1M tokens (~750 hal. A4) |
| Claude 3.5 Sonnet | 200K tokens (~150 hal. A4) |

> System prompt ikut mengurangi context window. Jangan buat system prompt terlalu panjang kalau tidak perlu.

---

### Temperature & Parameter

**Temperature** mengontrol tingkat kreativitas response:

| Value | Perilaku | Cocok untuk |
|---|---|---|
| 0 | Deterministik, selalu sama | Ekstraksi data, klasifikasi, coding |
| 0.3–0.5 | Sedikit variasi | Q&A, summarization, translation |
| 0.7 | Balanced (default) | Chat umum, brainstorming |
| 1.0–1.5 | Sangat kreatif, bisa tidak konsisten | Creative writing, ide bebas |

**max_tokens** — batas panjang response. Berguna untuk:
- Hemat biaya (response pendek = token sedikit)
- Paksa format output yang ringkas
- Cegah AI berkata terlalu panjang

**Parameter lain (biarkan default untuk pemula):**
- `top_p` — nucleus sampling
- `frequency_penalty` — kurangi pengulangan kata
- `presence_penalty` — dorong AI bahas topik baru

---

## Setup OpenRouter

### 1. Buat Akun

Buka https://openrouter.ai dan klik **Sign In**. Bisa pakai akun Google atau GitHub.

### 2. Buat API Key

Akses langsung ke: `https://openrouter.ai/settings/keys`

Klik **+ New Key**, isi:
- **Name**: `kelas-otomesyen` (atau nama bebas)
- **Expiration**: No expiration
- **Credit limit**: kosongkan (unlimited)

Klik **Create**. Copy API key yang muncul dan simpan — hanya ditampilkan sekali.

Format API key: `sk-or-v1-xxxxxxxxxxxxxxxxxxxxxxxxxx`

### 3. Simpan API Key di n8n

Di n8n, simpan key sebagai credential:
- Type: **Generic Credential Type** → **Header Auth**
- Name: `Authorization`
- Value: `Bearer sk-or-v1-xxxxxxxxxx`

> Jangan hardcode API key langsung di JSON body. Selalu simpan sebagai credential.

---

## Hasil Praktik

### PR 1 — First API Call

**Tujuan:** Berhasil call minimal 1 model gratis via HTTP Request node di n8n.

**Workflow:** `Modul 4 - PR1 & PR2 Model Comparison`

**Konfigurasi HTTP Request node:**

```
Method : POST
URL    : https://openrouter.ai/api/v1/chat/completions
Auth   : Generic Credential Type → Header Auth (credential yang sudah dibuat)
Body   : JSON (Using JSON)
```

**Body JSON yang dikirim:**

```json
{
  "model": "inclusionai/ling-3.0-flash-fin:free",
  "messages": [
    {
      "role": "system",
      "content": "Kamu adalah asisten yang menjawab singkat dan padat dalam bahasa Indonesia."
    },
    {
      "role": "user",
      "content": "Apa itu API testing?"
    }
  ],
  "temperature": 0.3,
  "max_tokens": 200
}
```

**Response yang diterima:**

```json
{
  "usage": {
    "prompt_tokens": 47,
    "completion_tokens": 200,
    "total_tokens": 247,
    "cost": 0
  },
  "model": "inclusionai/ling-3.0-flash-fin:free"
}
```

**Expression untuk ekstrak teks response di n8n:**
```
{{ $json.choices[0].message.content }}
```

**Hasil:** Berhasil. Total 247 token, cost $0 (gratis).

---

### PR 2 — Perbandingan 3 Model

**Tujuan:** Kirim prompt yang sama ke 3 model berbeda, bandingkan kualitas dan kecepatan.

**Struktur workflow:** 3 node HTTP Request paralel dari 1 Manual Trigger.

```
Manual Trigger --> HTTP Request  (Model 1)
               --> HTTP Request2 (Model 2)
               --> HTTP Request1 (Model 3)
```

**Prompt yang digunakan:** `"Apa itu API testing?"`

**Hasil perbandingan:**

| Model | Prompt Tokens | Completion Tokens | Total | Cost |
|---|---|---|---|---|
| `inclusionai/ling-3.0-flash-fin:free` | 47 | 200 | 247 | $0 |
| `poolside/laguna-xs-2.1:free` | 41 | 200 | 241 | $0 |
| `dots-studio/dots-3-note-preview:free` | 32 | 200 | 232 | $0 |

**Analisis:**
- Semua model kena batas `max_tokens: 200` sehingga completion tokens sama
- Perbedaan di prompt tokens menunjukkan tiap model punya cara tokenisasi yang sedikit berbeda
- Model dengan prompt tokens lebih sedikit berarti tokenizer-nya lebih efisien untuk teks yang dikirim

---

### PR 3 — Hitung Token

**Tujuan:** Estimasi token dari artikel Bahasa Indonesia, lalu verifikasi dengan API.

**Teks yang diuji:** Artikel tentang pertanian presisi (~150 kata Bahasa Indonesia)

**Hasil dari API:**
```
prompt_tokens    : 329
completion_tokens: 200
total_tokens     : 529
```

**Analisis:**
- 150 kata Bahasa Indonesia menghasilkan 329 prompt tokens (termasuk system prompt)
- Rasio: ~2 token per kata untuk Bahasa Indonesia
- Ini sesuai teori di modul — Bahasa Indonesia lebih boros token dibanding English karena banyak kata yang dipecah menjadi lebih banyak chunk

**Rumus estimasi kasar:**
```
Bahasa Indonesia: jumlah kata × 2 = estimasi token
English         : jumlah kata × 1.3 = estimasi token
```

---

### PR 4 — Chatbot Multi-Turn

**Tujuan:** Workflow yang bisa menerima input user dan mengirim history percakapan ke AI.

**Workflow:** `Modul 4 - PR4 Chatbot Multi-Turn`

**Konsep multi-turn:** Setiap request ke API harus menyertakan seluruh history percakapan. AI tidak menyimpan memori sendiri — kita yang mengirim ulang semua konteks setiap request.

**Body JSON multi-turn:**

```json
{
  "model": "inclusionai/ling-3.0-flash-fin:free",
  "messages": [
    {
      "role": "system",
      "content": "Kamu adalah asisten yang menjawab singkat dan padat dalam bahasa Indonesia."
    },
    {
      "role": "user",
      "content": "Apa itu machine learning?"
    },
    {
      "role": "assistant",
      "content": "Machine learning adalah cabang AI yang memungkinkan komputer belajar dari data tanpa diprogram secara eksplisit."
    },
    {
      "role": "user",
      "content": "Kasih contoh penggunaannya di kehidupan sehari-hari."
    }
  ],
  "temperature": 0.3,
  "max_tokens": 200
}
```

**Hasil:**
```
prompt_tokens    : 105
completion_tokens: 200
total_tokens     : 305
cost             : 0
```

**Catatan penting:** Prompt tokens 105 karena mencakup system prompt + 2 pesan user + 1 history assistant. Semakin panjang percakapan, semakin banyak token yang dikonsumsi per request karena semua history dikirim ulang.

---

### PR 5 — Experiment Temperature

**Tujuan:** Kirim prompt yang sama dengan temperature 0, 0.5, dan 1.0. Observe perbedaannya.

**Model:** `inclusionai/ling-3.0-flash-fin:free`

**Prompt:**
```
"Ceritakan tentang masa depan teknologi AI dalam 3 kalimat."
```

**Hasil:**

| Temperature | Prompt Tokens | Completion Tokens | Karakteristik Response |
|---|---|---|---|
| 0 | 52 | 200 | Deterministik, jawaban cenderung sama jika diulang |
| 0.5 | 52 | 200 | Sedikit variasi, masih fokus dan relevan |
| 1.0 | 52 | 200 | Lebih kreatif, diksi lebih bervariasi |

**Catatan:** Model reasoning seperti `ling-3.0-flash-fin` menggunakan internal chain-of-thought. Perbedaan temperature lebih terasa di konten teks response daripada di statistik token usage.

**Kesimpulan kapan pakai masing-masing:**
- Temperature 0: untuk automation, ekstraksi data, klasifikasi — butuh hasil konsisten
- Temperature 0.5: untuk Q&A, ringkasan, terjemahan — perlu sedikit fleksibilitas
- Temperature 1.0: untuk creative writing, brainstorming — butuh variasi dan kreativitas

---

## Catatan Masalah & Solusi

### Model tidak tersedia / sudah tidak gratis

**Error:**
```
The resource you are requesting could not be found
This model is unavailable for free.
```

**Penyebab:** Model `google/gemma-3-27b-it:free` yang ada di modul sudah tidak tersedia versi gratisnya.

**Solusi:** Ganti ke model gratis lain. Cek daftar model aktif di https://openrouter.ai/models dan filter by "Free".

---

### Rate limit

**Error:**
```
The service is receiving too many requests from you
Provider returned error (429)
```

**Penyebab:** Model yang dipilih sedang overload atau kita terlalu sering request dalam waktu singkat.

**Solusi:** Ganti ke model lain yang lebih stabil, atau tunggu beberapa detik lalu coba lagi.

---

### Body JSON masuk sebagai field, bukan raw JSON

**Gejala:** Di n8n, body yang diisi masuk ke "Value" dari "Body Field 1", bukan sebagai raw JSON.

**Penyebab:** "Specify Body" masih di mode "Using Fields Below".

**Solusi:** Ubah "Specify Body" dari "Using Fields Below" ke **"Using JSON"**, baru paste JSON-nya.

---

### Node HTTP Request tersambung serial, bukan paralel

**Gejala:** Node baru tersambung ke node sebelumnya, bukan langsung ke Manual Trigger.

**Penyebab:** Klik `+` di node HTTP Request (bukan di Manual Trigger).

**Solusi:** Untuk membuat node paralel, klik `+` yang ada di node **Manual Trigger**, bukan di node HTTP Request yang sudah ada.

---

## Kesimpulan

Dari modul ini, hal paling penting yang dipahami:

1. **OpenRouter** menyederhanakan akses ke ratusan model AI dengan satu API key. Untuk development dan belajar, selalu pakai model `:free` dulu.

2. **Token bukan kata.** Bahasa Indonesia lebih boros token (~2x dibanding English). Ini penting untuk estimasi biaya di production.

3. **AI tidak punya memori.** Setiap request harus menyertakan seluruh history percakapan. Ini yang membuat context window cepat penuh di chat panjang.

4. **Temperature** menentukan seberapa "kreatif" AI menjawab. Untuk automation pakai 0–0.3. Untuk creative tasks pakai 0.7–1.0.

5. **Raw HTTP Request sebelum AI Agent node** adalah cara terbaik untuk benar-benar memahami apa yang terjadi. Kalau nanti ada error di AI Agent node, kamu tahu harus debug di mana.

---

## Referensi

- OpenRouter: https://openrouter.ai
- Daftar model gratis: https://openrouter.ai/models (filter: Free)
- Pricing per model: klik model di halaman Models → lihat tab Pricing
- n8n instance: https://n8n.wican.my.id
