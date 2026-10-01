# 📚 Panduan Lengkap Pengembangan UI Menggunakan RmlUi di Sisi Server (Pawn / open.mp)

---

## 1. Arsitektur & Konsep Dasar

RmlUi adalah **engine UI berbasis C++** yang dirancang untuk rendering UI ringan di atas OpenGL. Pada proyek ini, RmlUi di‑wrap dalam sebuah library native (`librmlui-component.so` / `rmlui-component.dll`) yang dapat dipanggil dari skrip Pawn melalui **callback RPC**.

### 1.1 Komponen Utama
- **RmlUi Core** – C++ library yang meng‑parse file `.rml` (mirip HTML) dan `.rcss` (mirip CSS).
- **Resource Manager** – Menyajikan berkas UI, gambar, font, dan asset lain melalui HTTP server internal.
- **Event Bridge (RmlEventBridge.cpp)** – Menyalurkan event UI dari client (browser) ke server melalui RPC, serta meng‑serialize argumen ke format binary yang dapat dibaca di sisi Pawn.
- **Pawn Wrapper (`rmlui.inc`)** – Definisi fungsi RPC yang dapat dipanggil dari skrip Pawn (contoh: `RmlCreateDocument`, `RmlSetData`, `RmlTriggerEvent`).

### 1.2 Alur Kerja Umum
1. **Client meminta file UI** melalui HTTP (`/rmlui/...`).
2. Server meng‑serve file `.rml`, `.rcss`, gambar, dan font.
3. Pada client, RmlUi **render** UI ke dalam texture yang dibawa oleh CEF/GLFW.
4. Event UI (klik, submit, dll.) dikirim kembali ke server lewat RPC `RPC_RmlEventTriggered`.
5. Server men‑handle event, memodifikasi data, dan **update** UI dengan `RmlSetData` atau `RmlUpdateDocument`.

---

## 2. Persiapan Lingkungan Pengembangan

| Langkah | Perintah / Catatan |
|--------|-------------------|
| **Clone repository** | `git clone https://github.com/yourorg/alyn_samp_v17_1.git` |
| **Bangun komponen C++** | `cd rmlui-component && mkdir build && cd build && cmake .. && make -j$(nproc)` |
| **Copy library ke server** | `cp build/librmlui-component.so ../test_server/plugins/` |
| **Pasang dependensi Pawn** | Pastikan `rmlui.inc` berada di folder `include/` server Pawn. |
| **Konfigurasi HTTP server** | Edit `test_server/config.json` → `"http_port": 8080` dan pastikan `"rmlui_root": "scriptfiles/rmlui"`. |
| **Bangun server** | `cd test_server && ./build_server.sh` |

---

## 3. Struktur Direktori UI
```
scriptfiles/
└── rmlui/
    ├── login/
    │   ├── login.rml          # Layout login
    │   └── login.rcss         # Style login
    ├── hud/
    │   ├── hud.rml            # HUD layout
    │   └── hud.rcss           # HUD style
    ├── list_test/
    │   ├── list.rml           # Contoh list scrollable
    │   └── list.rcss          # Style list
    └── common/
        └── styles/
            ├── reset.rcss    # Reset CSS (normalize)
            └── fonts.rcss    # Font-face definitions
```

### 3.1 File `.rml`
Mirip HTML5 tetapi **tidak support** semua tag. Tag umum:
- `<div>`, `<span>`, `<img>`, `<input>`, `<button>`, `<form>`
- Data‑binding: `data-bind="myVar"` untuk meng‑update nilai secara otomatis dari server.

### 3.2 File `.rcss`
Sintaks mirip CSS, namun **property yang didukung** terbatas pada:
- `font-family`, `font-size`, `color`, `background`, `border`, `margin`, `padding`, `width`, `height`, `display`, `position`, `overflow`.

---

## 4. API Pawn – `rmlui.inc`

```pawn
// Membuat dokumen UI baru dan mengembalikan handle (int)
native RmlCreateDocument(const name[], const data[] = "");

// Mengupdate data‑binding pada dokumen yang sudah ada
native RmlSetData(handle, const key[], const value[]);

// Men‑trigger event client‑side (misalnya klik tombol)
native RmlTriggerEvent(handle, const eventName[], const args[] = "");

// Menutup dokumen UI
native RmlDestroyDocument(handle);
```

### 4.1 Contoh Penggunaan di Skrip Pawn
```pawn
new loginDoc;

public OnGameModeInit()
{
    // Buat dokumen login, kirim data awal (mis: server name)
    loginDoc = RmlCreateDocument("login/login.rml", "{\"serverName\":\"Sunrise MP\"}");
    // Tampilkan ke pemain
    ShowRmlDocument(loginDoc, playerid);
}

public OnRmlEventTriggered(playerid, handle, const eventName[], const jsonArgs[])
{
    if (eventName == "loginSubmit") {
        new username[32], password[32];
        json_decode(jsonArgs, "{s:s,s:s}", "username", username, "password", password);
        // Validasi di sisi server
        if (IsValidUser(username, password)) {
            // Update HUD setelah login berhasil
            RmlSetData(loginDoc, "status", "LoggedIn");
        } else {
            RmlSetData(loginDoc, "error", "Invalid credentials");
        }
    }
    return 1;
}
```

---

## 5. Asset Management – `resource-manager.cpp`

- **Manifest generation**: Pada startup, `ResourceManager::GenerateManifest()` memindai folder `scriptfiles/rmlui` dan men‑generate file `manifest.json` yang berisi **hash SHA‑256** tiap asset.
- **Cache‑busting**: Client men‑request asset dengan query `?v=<hash>` sehingga browser tidak meng‑cache versi lama.
- **Security**: Library otomatis menolak path yang mengandung `..` untuk mencegah **path traversal**.

### 5.1 Menambah Asset Baru
1. Tempatkan file di dalam `scriptfiles/rmlui/...`.
2. Jalankan server ulang atau panggil RPC `RmlReloadAssets()` (exposed via `rmlui.inc`).
3. Server akan memperbarui manifest dan client otomatis mengambil asset baru.

---

## 6. Event Handling – `RmlEventBridge.cpp`

### 6.1 RPC `RPC_RmlEventTriggered`
```cpp
// Structure yang diterima di sisi server
struct RmlEvent {
    int documentHandle;
    std::string eventName;
    std::string jsonArgs; // Serialized JSON string
};
```
- **Deserialisasi**: Di Pawn, gunakan `json_decode` untuk mengambil argumen.
- **Thread‑safety**: Semua callback dijalankan di thread utama server, sehingga tidak perlu mutex tambahan.

### 6.2 Mengirim Event ke Client
```cpp
void TriggerClientEvent(int handle, const std::string& name, const std::string& jsonArgs) {
    // Serialisasi ke binary, kirim via RakNet/ENet
    RPC::Call("RPC_RmlEventTriggered", handle, name.c_str(), jsonArgs.c_str());
}
```

---

## 7. Debugging & Troubleshooting

| Symptom | Penyebab Kemungkinan | Langkah Perbaikan |
|--------|----------------------|-------------------|
| UI tidak muncul di client | Library `.so` tidak ter‑load | Periksa log server (`librmlui-component.so: cannot open shared object file`). Pastikan path library sudah ditambahkan ke `LD_LIBRARY_PATH` atau copy ke folder `plugins/`. |
| Event tidak terkirim | RPC name mismatch (`RPC_RmlEventTriggered` vs `RPC_RmlEventTrigger`) | Periksa definisi di `RmlEventBridge.cpp` dan `rmlui.inc`. |
| Asset 404 | Manifest belum di‑refresh | Jalankan `RmlReloadAssets()` atau restart server. |
| Lag / FPS drop | UI rendering terlalu berat (gambar besar) | Optimalkan gambar, gunakan **texture atlas**, dan batasi ukuran UI ≤ 512 KB. |
| Crash saat `RmlSetData` | Nilai JSON tidak valid | Selalu gunakan `json_encode` di sisi server sebelum mengirim. |

### 7.1 Log Server
- Log berada di `test_server/logs/`. Cari baris yang mengandung `Rml` untuk detail error.
- Contoh output: `[Rml] Failed to load resource "hud/hud.rml" (404)`.

---

## 8. Best Practices & Tips

- **Gunakan data‑binding** sebanyak mungkin (`data-bind`) untuk mengurangi jumlah RPC yang diperlukan.
- **Batch update**: Kirim beberapa key/value dalam satu panggilan `RmlSetData` dengan JSON berisi object besar.
- **Validasi sisi server**: Jangan pernah mempercayai nilai yang dikirim dari client melalui `data-args`. Selalu cek kembali di database atau variabel server.
- **Batas ukuran payload**: Maksimum 512 KB per berkas UI. Jika membutuhkan lebih, pecah menjadi beberapa dokumen yang dipanggil secara dinamis.
- **Gunakan font-face** dengan format **WOFF2** untuk mengurangi ukuran.
- **Cache‑busting**: Manfaatkan hash di query string (`?v=<hash>`) untuk memastikan update asset.
- **Testing**: Jalankan UI di browser lokal dengan `http://localhost:8080/rmlui/login/login.rml` untuk memeriksa tampilan sebelum di‑integrasi ke server.
- **Security**: Aktifkan **Content‑Security‑Policy** di HTTP server untuk membatasi tipe file yang dapat dimuat.

---

## 9. Deploy ke Production
1. Build release version of `rmlui-component` dengan `-DCMAKE_BUILD_TYPE=Release`.
2. Copy library dan `manifest.json` ke server produksi.
3. Pastikan **port 8080** (atau yang dikonfigurasi) terbuka di firewall.
4. Set environment variable `RMLUI_ASSET_ROOT=/path/to/scriptfiles/rmlui` pada proses server.
5. Restart server dan verifikasi UI lewat client.

---

## 10. Referensi Tambahan
- **RmlUi Official Docs**: https://github.com/mikke89/RmlUi/wiki
- **open.mp RPC Guide**: https://wiki.open.mp/RPC
- **JSON for Pawn**: https://github.com/Southclaws/pawn-JSON

---

*Dokumentasi ini disusun berdasarkan kode sumber proyek, contoh UI yang ada, serta praktik terbaik pengembangan UI berbasis RmlUi di sisi server open.mp.*
