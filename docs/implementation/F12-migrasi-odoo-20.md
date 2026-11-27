# F12. Migrasi Module ke Odoo 20 (setelah rilis)

| | |
|---|---|
| **Requirement FSD** | NFR-CE-04 |
| **Waktu** | Setelah Odoo 20 dirilis dan image Docker `odoo:20.0` tersedia (±8–12 jam) |
| **Hasil** | `mrp_keripik` ter-install & lulus test di Odoo 20; catatan perubahan di `MIGRATION.md` |

> ⚠️ Odoo biasanya merilis versi mayor sekitar **Odoo Experience** (akhir September/Oktober). Per 5 Okt 2026, cek apakah Odoo 20 sudah dirilis. **Jangan migrasi di tengah proyek**: selesaikan F1–F11 di Odoo 19 dulu. Perubahan API yang disebut di bawah adalah **pola umum**; detail Odoo 20 harus kamu baca dari sumber resmi.

---

## 1. Konsep

### 1.1 Dua jenis migrasi

| | Migrasi **kode** module | Migrasi **data** database |
|---|---|---|
| Apa | Menyesuaikan Python/XML/JS dengan API baru | Mengubah DB 19 menjadi DB 20 |
| Alat | Manual + `odoo-module-migrator` (OCA) + log warning | Enterprise: layanan upgrade Odoo SA. **Community: OpenUpgrade (OCA)** |
| Di proyek ini | **Wajib** (inti F12) | Opsional (DB demo bisa dibuat ulang dengan `demo_data.py`) |

Padanan Frappe: `bench update` + patch (`patches.txt`). Di Odoo, migrasi data module memakai folder `migrations/<versi>/pre-migrate.py` / `post-migrate.py` / `end-migrate.py`.

### 1.2 Strategi branch
```
main (Odoo 19)  ──●──●──●──────────────
                         \
 20.0                     ●──●──● (migration)
```
Simpan versi 19 tetap berjalan (portofolio dua versi justru nilai plus).

---

## 2. Langkah

### Langkah 1 — Riset perubahan
- Release notes Odoo 20 & dokumentasi "Upgrade / Developer changelog".
- Diff source: `git diff 19.0..20.0 -- addons/mrp/models/mrp_production.py addons/mrp/models/mrp_workorder.py addons/stock/models/stock_move.py`.
- Cari method yang kamu override: `action_confirm`, `button_mark_done`, `button_start`, `button_finish`, `button_pending`, `_get_children`, field `quantity`, relasi `consume_line_ids`.
- Status OCA 20.0 (OpenUpgrade coverage, `quality_control_oca`).
- Status External API: setahuku Odoo 19 mendeprekasi XML-RPC/JSON-RPC lama demi JSON-2 API dengan rencana penghapusan di versi mendatang. Cek apakah ada dampak (proyek ini memakai REST custom, jadi kemungkinan aman).

### Langkah 2 — Lingkungan Odoo 20
```bash
git checkout -b 20.0
# docker-compose.yml: image odoo:20.0 (and postgres version required by 20)
docker compose down
docker volume rm keripik_odoo-db-data keripik_odoo-web-data   # fresh DB (or use a separate compose project name)
docker compose up -d
```
Atau jalankan berdampingan dengan nama project lain: `docker compose -p keripik20 up -d` dan port berbeda.

### Langkah 3 — Bump versi & alat otomatis
`__manifest__.py`: `"version": "20.0.1.0.0"`.

Coba OCA **odoo-module-migrator** (cek dukungan target 20.0):
```bash
pipx install odoo-module-migrator    # ⚠️ verify package name / install method in its README
odoo-module-migrate --directory odoo/addons --modules mrp_keripik --init-version-name 19.0 --target-version-name 20.0
```
Tinjau setiap perubahan yang dibuat alat ini; jangan diterima mentah.

### Langkah 4 — Install bersih + test
```bash
make install MOD=mrp_keripik 2>&1 | tee migrate-install.log
grep -E "WARNING|ERROR|DeprecationWarning" migrate-install.log
make test MOD=mrp_keripik
```
Perbaiki berurutan:
1. Error Python saat import (import path, nama field).
2. Error parse XML (atribut/elemen view, XPath view induk berubah).
3. Error ACL/grup (struktur grup berubah).
4. Test gagal (perilaku method standar berubah).
5. Warning deprecation (penting untuk Odoo 21).
6. Asset JS/OWL (import path `@web/...`, API service, versi OWL).

### Langkah 5 — Uji manual ulang
Jalankan checklist AC dari F1–F7 (MTO, QC blokir, yield, recall, layar operator, API, email, Telegram). Integrasi Go & Next.js seharusnya tidak berubah karena kontrak REST tetap. Ini bukti nyata manfaat **kontrak API sendiri**.

### Langkah 6 — Migrasi data (opsional, OpenUpgrade)
1. Backup DB 19.
2. Ikuti panduan OpenUpgrade 20.0 (jalankan server OpenUpgrade dengan `--update all` di DB salinan).
3. Untuk `mrp_keripik`, tulis script bila ada perubahan skema:
   ```
   mrp_keripik/migrations/20.0.1.0.0/pre-migrate.py
   mrp_keripik/migrations/20.0.1.0.0/post-migrate.py
   ```
   ```python
   # post-migrate.py
   def migrate(cr, version):
       """Example: recompute stored fields whose dependencies changed in 20."""
       # cr.execute("UPDATE ...")
       pass
   ```

### Langkah 7 — Dokumentasikan
`MIGRATION.md`:
| Area | Odoo 19 | Odoo 20 | Perubahan di mrp_keripik |
|---|---|---|---|
| (contoh) WO finish | `button_finish` | … | override dipindah ke … |

Tambahkan badge di README: "Supports Odoo 19 & 20".

---

## 3. Error yang Sering Terjadi

| Gejala | Penyebab | Solusi |
|---|---|---|
| `KeyError: 'field_x'` saat install | Field standar di-rename | Cari di source 20 |
| XPath tidak ditemukan | View induk direstrukturisasi | Lihat arch baru; pakai `name=` attribute yang stabil |
| `super()` signature mismatch | Method standar menambah/menghapus argumen | Gunakan `*args, **kwargs` dan teruskan |
| JS `Cannot find module @web/...` | Path modul web berubah | Cari di `addons/web/static/src` versi 20 |
| Test lulus di 19, gagal di 20 | Perilaku default berubah (misal demo data, stock rule) | Sesuaikan setup test |

---

## 4. Pertanyaan Interview
1. **Bagaimana kamu memigrasi custom module antar versi Odoo?** Riset changelog/diff, branch terpisah, alat OCA, install bersih dengan log, perbaiki per kategori, test otomatis + AC manual, script migrasi data bila skema berubah.
2. **Apa yang membuat module mudah dimigrasi?** Override minimal di titik stabil, logika di method sendiri, test yang baik, kontrak API eksternal sendiri, tidak bergantung detail internal UI.
3. **Community tanpa layanan upgrade Odoo SA, bagaimana?** OpenUpgrade (OCA) untuk data, atau migrasi data selektif/ekspor-impor untuk instalasi kecil.
