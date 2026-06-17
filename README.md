# Econo-Shield

Econo-Shield adalah dashboard peringatan risiko finansial untuk individu dan UMKM. Aplikasi menggunakan Next.js, API Route, Drizzle ORM, SQLite/libSQL, Better Auth, serta dataset BI Rate, inflasi M-to-M, suku bunga kredit, dan UMP.

## Library

- `Next.js`: frontend dan backend Route Handler dalam satu aplikasi.
- `React`: state dan interaksi dashboard.
- `TypeScript`: pemeriksaan tipe input, hasil model, API, dan database.
- `Tailwind CSS`: styling responsif.
- `lucide-react`: ikon UI.
- `better-auth`: autentikasi email/password, session cookie, dan hashing password.
- `@better-auth/drizzle-adapter`: menghubungkan Better Auth dengan schema Drizzle.
- `drizzle-orm`: query database bertipe aman.
- `drizzle-kit`: membuat dan menjalankan migrasi schema.
- `@libsql/client`: driver SQLite lokal dan database libSQL/Turso untuk deployment.
- `csv-parse`: parsing CSV dengan jumlah kolom yang dapat berbeda.
- `zod`: validasi body API dan pembatasan nilai finansial.
- `tsx`: menjalankan script TypeScript untuk seed dan pengujian model.

## Menjalankan Lokal

```bash
npm install
copy .env.example .env.local
npm run db:setup
npm run test:model
npm run dev
```

Buka `http://localhost:3000`.

Login demo:

- Nama boleh diisi bebas.
- Sandi: `wealth`.

Nama diubah menjadi email internal deterministik untuk Better Auth. Contoh `Toko Maju` menjadi `toko.maju@econo-shield.app`. Password tetap disimpan dalam bentuk hash oleh Better Auth.

> Sandi bersama `wealth` hanya cocok untuk demonstrasi. Produksi sebaiknya meminta password unik minimal 8-12 karakter dan verifikasi email.

## Environment

```env
DATABASE_URL=file:./data/econo-shield.db
DATABASE_AUTH_TOKEN=
BETTER_AUTH_SECRET=replace-with-at-least-32-random-characters
BETTER_AUTH_URL=http://localhost:3000
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

Untuk Vercel, filesystem lokal tidak persisten. Gunakan database SQLite-compatible seperti Turso:

```env
DATABASE_URL=libsql://database-name.turso.io
DATABASE_AUTH_TOKEN=your-token
BETTER_AUTH_URL=https://your-domain.vercel.app
NEXT_PUBLIC_APP_URL=https://your-domain.vercel.app
```

## Database

Schema berada di `lib/db/schema.ts`.

Tabel utama:

- `user`, `session`, `account`, `verification`: schema Better Auth.
- `macro_indicator`: BI Rate, inflasi kota/proxy nasional, dan bunga kredit.
- `minimum_wage`: UMP per provinsi tahun 2020.
- `dataset_import`: ringkasan kualitas dan hasil import.
- `financial_record`: input dan hasil analisis milik user.
- `inventory_item`: inventaris yang terhubung ke analisis.

Perintah:

```bash
npm run db:generate
npm run db:migrate
npm run db:seed
npm run db:setup
```

## Dataset

Dataset mentah berada di `data/raw`:

- BI Rate 2010-2026.
- Inflasi M-to-M 2010-2026.
- Suku bunga kredit menurut kelompok bank 2011-2026.
- UMP per provinsi 2020.

Beberapa file sumber hanya memiliki header atau bulan kosong. Pipeline tidak menganggap angka buatan sebagai data aktual.

Aturan kualitas:

1. Nilai sumber disimpan dengan `isEstimated = false`.
2. Inflasi nasional adalah proxy rata-rata aritmetika kota tanpa bobot populasi.
3. Celah di antara dua observasi aktual dapat diisi interpolasi linear.
4. Hasil interpolasi selalu diberi `isEstimated = true`, `method = linear_interpolation`, dan `sourceFile = MODEL_GENERATED`.
5. Bulan setelah observasi terbaru tidak diekstrapolasi.
6. Seed gagal jika ada duplikat, periode tidak valid, atau nilai di luar batas wajar.

Hasil seed saat implementasi:

- 7.194 baris makro.
- 65 observasi BI Rate.
- 4.552 baris inflasi.
- 2.577 baris bunga kredit.
- 35 baris UMP.
- 144 titik inflasi dan 12 titik bunga kredit ditandai sebagai estimasi.

## API Route

- `GET /api/macro/latest?segment=msme`: data makro terbaru dan UMP.
- `GET /api/macro/history?indicator=bi_rate&from=2020-01`: histori makro.
- `POST /api/calculate`: kalkulasi server-side; membutuhkan session.
- `GET /api/financial-records`: daftar analisis user.
- `POST /api/financial-records`: hitung dan simpan analisis.
- `/api/auth/*`: endpoint Better Auth.

Body kalkulasi:

```json
{
  "input": {
    "segment": "msme",
    "monthlyIncome": 45000000,
    "fixedExpense": 16500000,
    "variableExpense": 8700000,
    "savingsAsset": 65000000,
    "physicalAsset": 120000000,
    "investmentValue": 25000000,
    "debt": 42000000,
    "monthlyDebtPayment": 3500000,
    "incomeShock": 20,
    "biRate": 5.25,
    "inflationRate": 4.2,
    "loanRate": 8
  },
  "inventoryItems": []
}
```

Nilai makro dari database menjadi default. Slider dapat mengirim override untuk stress test; response tetap menyertakan data dasar dan nilai yang benar-benar diterapkan.

## Model Matematika

Variabel:

- `I`: pendapatan bulanan.
- `S`: shock pendapatan dalam persen.
- `E_f`: pengeluaran tetap.
- `E_v`: pengeluaran variabel.
- `pi`: inflasi 12 bulan.
- `D`: total utang.
- `C`: cicilan bulanan aktual.
- `r_l`: bunga kredit tahunan.
- `A_l`: aset likuid.
- `A_p`: aset fisik.
- `A_i`: investasi.
- `Q`, `P`: kuantitas dan HPP inventaris.

Pendapatan setelah shock:

```text
I_stress = I x (1 - S / 100)
```

Pengeluaran setelah tekanan inflasi:

```text
E_inflation = (E_f + E_v) x (1 + pi / 100)
```

Estimasi bunga bulanan minimum:

```text
Interest = D x (r_l / 100 / 12)
```

Debt service memakai cicilan aktual, tetapi tidak boleh lebih rendah dari estimasi bunga:

```text
DebtService = max(C, Interest)
```

Total pengeluaran dan surplus:

```text
E_total = E_inflation + DebtService
Surplus = I_stress - E_total
```

Rasio cicilan:

```text
DSR = DebtService / I_stress
```

Ini memperbaiki rumus lama yang membandingkan total utang dengan pendapatan tahunan. Ambang 30% dan 50% seharusnya menggunakan beban cicilan bulanan.

Inventaris, aset, dan leverage:

```text
Inventory = sum(Q x P)
A_total = A_l + A_p + A_i + Inventory
NetWorth = A_total - D
Leverage = D / A_total
```

Cash runway:

```text
CashRunway = (A_l + A_i) / E_total
```

Inflasi 12 bulan menggunakan compounding, bukan penjumlahan:

```text
Inflation12M = product(1 + inflation_mtm / 100) - 1
```

## Skor dan Klasifikasi

Skor 0-100 terdiri dari:

- Arus kas: maksimal 30 poin.
- Cash runway: maksimal 25 poin.
- Rasio cicilan: maksimal 25 poin.
- Solvabilitas: maksimal 15 poin.
- Tekanan makro: maksimal 5 poin.

Klasifikasi:

- `Bankrupt`: surplus negatif, utang melebihi aset likuid, dan runway di bawah 1 bulan.
- `Near Crisis`: surplus tidak positif, runway maksimal 3 bulan, DSR minimal 50%, utang melebihi total aset, atau skor di bawah 55.
- `Resilient`: DSR di bawah 30%, runway di atas 6 bulan, dan skor minimal 70.

Model adalah decision-support, bukan diagnosis hukum kebangkrutan atau nasihat investasi.

## Validasi

```bash
npm run test:model
npm run build
```

`test:model` memeriksa:

- Rumus `sum(Q x P)`.
- DSR memakai cicilan bulanan.
- Skenario sehat menjadi `Resilient`.
- Skenario arus kas negatif menjadi `Bankrupt`.
- Seluruh hasil numerik finite, bukan `NaN` atau `Infinity`.

## Struktur

- `app/page.tsx`: UI login dan dashboard.
- `app/api`: route auth, makro, kalkulasi, dan penyimpanan.
- `lib/auth.ts`: konfigurasi Better Auth.
- `lib/db`: koneksi dan schema Drizzle.
- `lib/calculations.ts`: model finansial murni.
- `lib/macro-data.ts`: agregasi data makro.
- `lib/validation.ts`: validasi API.
- `scripts/seed-datasets.ts`: normalisasi dan seed CSV.
- `scripts/validate-model.ts`: pengujian matematis.
- `drizzle`: migrasi SQL.
