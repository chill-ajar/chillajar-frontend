# Case Study: Sistem Paket Berbasis Waktu & Fitur "Tidak Terbatas"

**Tanggal:** 2026-05-15  
**Project:** chillAjar Frontend  
**Status:** Proposed (belum diimplementasi)

---

## 1. Latar Belakang & Problem Statement

### Bagaimana sistem paket saat ini bekerja

Setiap paket memiliki dua field opsional untuk periode promo:

| Field | Tipe | Keterangan |
|---|---|---|
| `tanggal_mulai` | `string \| null` | Tanggal mulai paket tampil ke user |
| `tanggal_berakhir` | `string \| null` | Tanggal paket tidak lagi tampil ke user |

### Logika filter di sisi user

File: `src/components/Student/CoursePackageSelectionModal.jsx` (baris 44–60)

```js
const activePackages = visiblePackages.filter((pkg) => {
  const now = new Date();

  if (pkg.tanggal_mulai) {
    const startDate = new Date(pkg.tanggal_mulai);
    if (startDate > now) return false; // Belum mulai → tidak tampil
  }

  if (pkg.tanggal_berakhir) {
    const endDate = new Date(pkg.tanggal_berakhir);
    if (endDate < now) return false; // Sudah expired → tidak tampil
  }

  return true; // Aktif → tampil
});
```

**Kesimpulan:** Jika `tanggal_berakhir` sudah lewat, paket **otomatis hilang dari pilihan user** — tidak ada konfirmasi atau notifikasi ke admin.

### Logika status di sisi admin

File: `src/pages/admin/manage-packages/AdminPackagesPage.jsx` (baris 154–190)

| Kondisi | Status yang ditampilkan | Warna |
|---|---|---|
| `tanggal_mulai` & `tanggal_berakhir` = null | **Tidak Terbatas** | Abu-abu |
| Sekarang < `tanggal_mulai` | **Akan Datang** | Biru |
| `tanggal_mulai` ≤ Sekarang ≤ `tanggal_berakhir` | **Aktif** | Hijau |
| Sekarang > `tanggal_berakhir` | **Berakhir** | Merah |

---

## 2. Problem yang Diidentifikasi

### Problem Utama: Admin Harus Update Manual Secara Berkala

Ketika sebuah paket dibuat dengan periode waktu tertentu (misalnya promo Lebaran, promo semester), setelah tanggal berakhir:

1. Paket **langsung hilang** dari tampilan user tanpa peringatan
2. Admin **tidak mendapat notifikasi** bahwa paket sudah expired
3. Admin harus **masuk ke halaman Paket**, cek satu per satu status "Berakhir"
4. Admin harus **edit paket** dan memperbarui `tanggal_berakhir` ke tanggal baru
5. Proses ini harus dilakukan **berulang** setiap kali periode promo habis

### Dampak

- **Bagi user:** Pilihan paket tiba-tiba hilang tanpa penjelasan, bisa menyebabkan kebingungan saat checkout
- **Bagi admin:** Beban operasional tinggi, rentan lupa update → paket lama tidak tersedia padahal masih relevan
- **Bagi bisnis:** Potensi kehilangan penjualan karena paket tidak tampil saat user ingin membeli

---

## 3. Solusi yang Diusulkan

### Fitur Toggle "Tidak Terbatas" pada Form Paket

Tambahkan sebuah checkbox/toggle di `FormPackagePage` dengan label **"Paket Tidak Terbatas (Tanpa Batas Waktu)"**.

#### Behaviour yang diharapkan:

| Kondisi Toggle | Behaviour |
|---|---|
| Toggle **aktif** (Tidak Terbatas) | Field tanggal disembunyikan, `tanggal_mulai` dan `tanggal_berakhir` dikirim sebagai `null` |
| Toggle **tidak aktif** (Berbatas Waktu) | Field tanggal mulai & berakhir muncul dan wajib diisi keduanya |

#### Kenapa ini solusi yang tepat:

1. **Backend sudah support** — field `tanggal_mulai: null` dan `tanggal_berakhir: null` sudah dihandle di kedua sisi (admin status menampilkan "Tidak Terbatas", user filter melewatkan pengecekan tanggal)
2. **Tidak butuh perubahan API** — payload sudah mendukung `null` untuk kedua field (lihat `FormPackagePage.jsx` baris 204–205)
3. **UX lebih eksplisit** — admin secara sadar memilih apakah paket punya batas waktu atau tidak, menghindari kesalahan lupa isi tanggal
4. **Mengurangi beban operasional** — paket reguler (non-promo) cukup dibuat sekali dengan toggle "Tidak Terbatas"

---

## 4. Rencana Implementasi

### File yang perlu diubah

Hanya satu file: `src/pages/admin/manage-packages/FormPackagePage.jsx`

### Perubahan State

```js
// Tambah state baru
const [isTidakTerbatas, setIsTidakTerbatas] = useState(false);
```

Saat load data edit mode, set `isTidakTerbatas = true` jika kedua tanggal null:

```js
// Di dalam useEffect fetch package (baris ~89)
const tidakTerbatas = !pkg.tanggal_mulai && !pkg.tanggal_berakhir;
setIsTidakTerbatas(tidakTerbatas);
```

### Perubahan UI — Bagian Form Periode Promo

Ganti bagian form tanggal yang sekarang (sekitar baris 330–360) dengan:

```jsx
{/* Toggle Tidak Terbatas */}
<div className="flex items-center gap-3 mb-4">
  <input
    type="checkbox"
    id="tidakTerbatas"
    checked={isTidakTerbatas}
    onChange={(e) => {
      setIsTidakTerbatas(e.target.checked);
      if (e.target.checked) {
        setFormData((prev) => ({
          ...prev,
          tanggal_mulai: "",
          tanggal_berakhir: "",
        }));
      }
    }}
    className="w-4 h-4 accent-chill-blue cursor-pointer"
  />
  <label htmlFor="tidakTerbatas" className="text-sm font-medium text-gray-700 cursor-pointer">
    Paket Tidak Terbatas (Tanpa Batas Waktu)
  </label>
</div>

{/* Field tanggal hanya tampil jika toggle TIDAK aktif */}
{!isTidakTerbatas && (
  <div className="grid grid-cols-2 gap-4">
    <div>
      <label className="block text-sm font-medium text-gray-700 mb-1">
        Tanggal Mulai
      </label>
      <input
        type="date"
        value={formData.tanggal_mulai}
        onChange={(e) =>
          setFormData((prev) => ({ ...prev, tanggal_mulai: e.target.value }))
        }
        className="w-full border border-gray-300 rounded-lg px-3 py-2 text-sm"
      />
    </div>
    <div>
      <label className="block text-sm font-medium text-gray-700 mb-1">
        Tanggal Berakhir
      </label>
      <input
        type="date"
        value={formData.tanggal_berakhir}
        onChange={(e) =>
          setFormData((prev) => ({ ...prev, tanggal_berakhir: e.target.value }))
        }
        className="w-full border border-gray-300 rounded-lg px-3 py-2 text-sm"
      />
    </div>
  </div>
)}
```

### Perubahan Payload Submit

Tidak perlu perubahan — payload sudah mengirim `null` jika field kosong (baris 204–205):

```js
tanggal_mulai: formData.tanggal_mulai || null,
tanggal_berakhir: formData.tanggal_berakhir || null,
```

### Perubahan Validasi

Tambahkan bypass validasi tanggal jika toggle aktif:

```js
// Di handleSubmit, sebelum validasi tanggal
if (!isTidakTerbatas) {
  if (formData.tanggal_mulai && formData.tanggal_berakhir) {
    const startDate = new Date(formData.tanggal_mulai);
    const endDate = new Date(formData.tanggal_berakhir);
    if (endDate <= startDate) {
      throw new Error("Tanggal berakhir harus setelah tanggal mulai");
    }
  }
  if (
    (formData.tanggal_mulai && !formData.tanggal_berakhir) ||
    (!formData.tanggal_mulai && formData.tanggal_berakhir)
  ) {
    throw new Error("Jika mengatur periode promo, kedua tanggal harus diisi");
  }
}
```

---

## 5. Tidak Ada Perubahan yang Diperlukan di Sisi Lain

| File | Perlu Diubah? | Alasan |
|---|---|---|
| `CoursePackageSelectionModal.jsx` | ❌ Tidak | Logika filter sudah handle `null` dengan benar — paket tanpa tanggal selalu lolos filter |
| `CoursePackageCard.jsx` | ❌ Tidak | Countdown timer sudah handle `null` — tidak menampilkan badge timer jika `tanggal_berakhir` null |
| `AdminPackagesPage.jsx` | ❌ Tidak | Status "Tidak Terbatas" sudah tampil jika kedua tanggal null |
| Backend API | ❌ Tidak | `null` untuk kedua field sudah diterima dan disimpan |

---

## 6. Pertanyaan Terbuka

### Apakah sistem waktu ini akan dipakai untuk auto-hidden di masa depan?

**Jawaban:** Secara teknis, **sudah berfungsi sebagai auto-hidden sekarang**. Paket yang `tanggal_berakhir`-nya lewat otomatis tidak tampil ke user tanpa perlu aksi manual admin dari sisi frontend.

Yang belum ada adalah:
- **Notifikasi ke admin** ketika sebuah paket mendekati atau melewati tanggal berakhir
- **Auto-reaktivasi** — saat ini admin harus manual update tanggal
- **Scheduled job** di backend untuk mengubah status paket secara otomatis (jika butuh flag di DB level)

### Rekomendasi ke depan

Jika sistem promo berbasis waktu akan dipakai intensif, pertimbangkan:

1. **Fitur notifikasi** — email/notif in-app ke admin H-3 sebelum paket expired
2. **Filter di admin panel** — tombol "Tampilkan yang Berakhir" agar mudah ditemukan
3. **Bulk extend** — kemampuan memperbarui tanggal berakhir banyak paket sekaligus

---

## 7. Ringkasan

```
Masalah:   Paket dengan tanggal berakhir → hilang otomatis dari user view
           Admin harus update manual secara berkala → beban operasional tinggi

Solusi:    Tambah toggle "Tidak Terbatas" di FormPackagePage
           → Jika aktif: tanggal_mulai & tanggal_berakhir = null
           → Paket selalu tampil ke user tanpa perlu diupdate

Dampak:    Minimal — hanya 1 file diubah (FormPackagePage.jsx)
           Tidak ada perubahan API, tidak ada perubahan komponen lain
           Backend & frontend sudah support null sejak awal
```
