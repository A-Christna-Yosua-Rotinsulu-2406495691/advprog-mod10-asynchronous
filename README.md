# ⚡ Modul 10: Asynchronous Programming di Rust
### Nama: Christna Yosua Rotinsulu
### NPM: 2406495691
### Kelas: Pemrograman Lanjut - Fasilkom UI

---

## 🧭 Eksperimen 1.2: Memahami Cara Kerja Eksekusi Asinkronus

Pada eksperimen ini, saya mengeksplorasi bagaimana alur eksekusi asinkronus (*asynchronous execution flow*) bekerja di Rust dengan menambahkan baris cetak (`hey hey`) di utas utama (*main thread*) tepat setelah tugas didelegasikan ke Spawner.

### 📸 Hasil Eksekusi Program di Terminal Saya

Berikut adalah cuplikan keluaran ketika saya menjalankan program hasil modifikasi ini menggunakan perintah `cargo run`:

```bash
Christna Yosua Rotinsulu's Komputer: hey hey
Christna Yosua Rotinsulu's Komputer: howdy!
Christna Yosua Rotinsulu's Komputer: done!
```

---

## 🧠 Analisis & Penjelasan Mengapa `"hey hey"` Muncul Pertama

Meskipun baris `println!("... hey hey")` ditulis setelah pemanggilan `spawner.spawn(async { ... })`, teks `"hey hey"` dicetak **paling pertama** di terminal saya. Berikut adalah penjelasan logis mengapa hal ini terjadi:

1. **Sifat `spawner.spawn` yang Bersifat Non-Blocking (Asinkronus)**:
   Saat saya memanggil `spawner.spawn(...)`, blok kode `async` di dalamnya tidak langsung dieksekusi secara instan. Fungsi `spawn` hanya membungkus masa depan (*Future*) tersebut ke dalam bentuk `Task`, lalu mengirimkannya ke dalam antrean tugas (`ready_queue`) melalui pengirim channel asinkronus (`task_sender`). Proses pengiriman tugas ke antrean ini berlangsung sangat cepat dan tidak memblokir utas utama.

2. **Eksekusi Sinkronus pada Utas Utama**:
   Setelah tugas berhasil dimasukkan ke dalam antrean, utas utama (*main thread*) segera melanjutkan eksekusi ke baris berikutnya secara sinkronus. Baris berikutnya adalah `println!("Christna Yosua Rotinsulu's Komputer: hey hey");`. Karena utas utama saat itu bebas dari hambatan, kalimat `"hey hey"` dicetak pertama kali.

3. **Executor Belum Berjalan**:
   Blok `async` yang berisi `"howdy!"` dan `"done!"` belum dapat dieksekusi karena mesin penggeraknya, yaitu `executor.run()`, baru dipanggil di bagian paling akhir fungsi `main`. Sebelum `executor.run()` dieksekusi, tidak ada komponen yang memproses atau melakukan polling terhadap antrean tugas tersebut.

4. **Alur Eksekusi di dalam Executor**:
   Setelah `"hey hey"` dicetak dan `drop(spawner)` dipanggil, program akhirnya memanggil `executor.run()`. Di sinilah tugas di dalam antrean mulai dieksekusi:
   - **Tugas Dipolling (Poll Pertama)**: Executor mengambil tugas dari antrean dan memanggil metode `poll()`. Ini memicu pencetakan `"howdy!"`.
   - **Menunggu Timer**: Saat menemui `TimerFuture::new(...).await`, executor mendeteksi bahwa timer asinkronus belum selesai (`Poll::Pending`). Executor secara cerdas melepas tugas ini dan menunggu waker di utas terpisah untuk membangunkannya kembali setelah 2 detik berlalu.
   - **Tugas Dipolling Kembali (Poll Kedua)**: Setelah 2 detik, utas timer memanggil `waker.wake()`, menaruh kembali tugas ke antrean. Executor mendapati statusnya telah selesai (`Poll::Ready`) lalu melanjutkan eksekusi hingga mencetak kata `"done!"`.

---
