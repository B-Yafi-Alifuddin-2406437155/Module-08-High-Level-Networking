# Module-08-High-Level-Networking


### 1. What are the key differences between unary, server streaming, and bi-directional streaming RPC (Remote Procedure Call) methods, and in what scenarios would each be most suitable?
*   **Unary RPC:** Klien mengirimkan satu *request* tunggal ke server dan server membalas dengan satu *response* tunggal (mirip seperti pemanggilan fungsi biasa).
    *   *Skenario yang cocok:* Mengambil data profil pengguna, memproses pembayaran sederhana, atau melakukan autentikasi login.
*   **Server Streaming RPC:** Klien mengirimkan satu *request*, namun server membalas dengan aliran (*stream*) data yang terdiri dari banyak *response* secara berurutan.
    *   *Skenario yang cocok:* Mengunduh file berukuran besar, mengambil data riwayat transaksi yang panjang (seperti contoh di tutorial), atau menerima pembaruan harga saham secara *real-time*.
*   **Bi-directional Streaming RPC:** Klien dan server saling mengirimkan aliran (*stream*) pesan secara independen dalam satu koneksi yang sama. Keduanya dapat membaca dan menulis kapan saja.
    *   *Skenario yang cocok:* Aplikasi *chatting real-time*, sistem *multiplayer game* untuk sinkronisasi posisi pemain, atau panggilan video/audio.

### 2. What are the potential security considerations involved in implementing a gRPC service in Rust, particularly regarding authentication, authorization, and data encryption?
*   **Authentication (Autentikasi):** Memastikan identitas klien yang mengakses layanan. Di Rust (menggunakan `tonic`), kita bisa mengimplementasikan *Interceptors* untuk memeriksa token JWT atau *API keys* yang disisipkan klien pada *metadata* (header) setiap *request*.
*   **Authorization (Otorisasi):** Setelah klien dikenali, sistem harus memeriksa apakah klien tersebut memiliki izin (seperti *role admin* atau *user*) untuk mengeksekusi metode RPC tertentu.
*   **Data Encryption (Enkripsi Data):** Menjaga agar data tidak bisa dibaca oleh pihak ketiga saat transmisi. Ini wajib dilakukan dengan mengaktifkan TLS/mTLS (Transport Layer Security) yang juga didukung secara bawaan oleh `tonic` untuk mengenkripsi jalur komunikasi gRPC.

### 3. What are the potential challenges or issues that may arise when handling bidirectional streaming in Rust gRPC, especially in scenarios like chat applications?
*   **Manajemen Status dan Konkurensi:** Melacak koneksi mana yang masih aktif dan memastikan pesan dikirim ke pengguna yang tepat (*routing* pesan) bisa menjadi rumit, terutama saat jumlah klien membesar.
*   **Penanganan Error & Putus Koneksi:** Klien bisa terputus secara tiba-tiba karena masalah jaringan. Server harus dirancang agar tidak *crash* atau *panic* saat hal ini terjadi (menggunakan `unwrap_or_else` atau penanganan hasil `Result` yang tepat).
*   **Kebocoran Memori (Resource Leaks):** Jika *channel* pengiriman pesan (`mpsc`) atau *background task* (`tokio::spawn`) tidak ditutup/dibersihkan saat klien *disconnect*, memori server akan terus membengkak (*memory leak*).

### 4. What are the advantages and disadvantages of using the tokio_stream::wrappers::ReceiverStream for streaming responses in Rust gRPC services?
*   **Keuntungan:** Sangat mudah diintegrasikan dengan arsitektur *channel* `mpsc` milik Tokio. Ini memungkinkan kita untuk melempar pekerjaan berat atau proses iterasi ke *background task* (`tokio::spawn`), sementara *channel receiver* diubah dengan mulus menjadi *stream* yang siap dikembalikan oleh *handler* gRPC `tonic`.
*   **Kerugian:** Ada sedikit *overhead* memori karena data harus melewati *buffer channel* terlebih dahulu sebelum menjadi *stream*. Jika ukuran *buffer* tidak diatur dengan tepat (misal terlalu kecil dan produsen terlalu cepat), bisa terjadi *bottleneck* atau tertahannya eksekusi program.

### 5. In what ways could the Rust gRPC code be structured to facilitate code reuse and modularity, promoting maintainability and extensibility over time?
*   **Pemisahan Modul (Separation of Concerns):** Pisahkan kode definisi proto, implementasi *server*, implementasi *client*, dan logika bisnis ke dalam *file* atau modul terpisah (misalnya `payment.rs`, `chat.rs`, `transaction.rs`).
*   **Decoupling Logika Bisnis:** Jangan menaruh logika bisnis yang kompleks (seperti kalkulasi pajak atau validasi *database*) langsung di dalam fungsi gRPC. Buatlah *layer* layanan tersendiri (*Dependency Injection*), lalu panggil *layer* tersebut dari *handler* gRPC.
*   **Manajemen Error Sentral:** Buat tipe *error* kustom dan gunakan utilitas pemetaan agar *error* internal sistem bisa diterjemahkan menjadi kode status gRPC (seperti `Status::not_found`) secara seragam.

### 6. In the MyPaymentService implementation, what additional steps might be necessary to handle more complex payment processing logic?
*   **Integrasi Database:** Menyimpan status transaksi (pending, success, failed) ke dalam basis data (seperti PostgreSQL) menggunakan ORM seperti Diesel atau SeaORM.
*   **Komunikasi dengan Payment Gateway:** Melakukan HTTP *request* ke layanan pihak ketiga (seperti Stripe atau Midtrans) untuk memproses uang secara riil.
*   **Idempotensi:** Menerapkan mekanisme idempotensi menggunakan *unique request ID* untuk mencegah terjadinya tagihan ganda (*double charge*) jika terjadi *retry* akibat masalah jaringan.
*   **Logging dan Metrik:** Menambahkan *tracing* atau *logging* mendetail untuk keperluan audit jika ada pembayaran yang bermasalah.

### 7. What impact does the adoption of gRPC as a communication protocol have on the overall architecture and design of distributed systems, particularly in terms of interoperability with other technologies and platforms?
*   **Arsitektur:** Mendorong penggunaan arsitektur berbasis *Microservices*. gRPC memaksa adanya kontrak API yang ketat via Protobuf, sehingga mengubah cara tim merancang API menjadi lebih disiplin dibandingkan REST konvensional.
*   **Interoperabilitas:** Sangat tinggi antar layanan *backend*. Protobuf memungkinkan kita membuat *client* dan *server* di berbagai bahasa pemrograman (Rust, Go, Python, Java) dan mereka dapat berkomunikasi tanpa memikirkan cara *parsing* data. Namun, gRPC kurang *interoperable* dengan *web browser* standar, sehingga seringkali membutuhkan proksi (seperti Envoy gRPC-Web) agar bisa dipanggil oleh aplikasi *frontend* (React/Vue).

### 8. What are the advantages and disadvantages of using HTTP/2, the underlying protocol for gRPC, compared to HTTP/1.1 or HTTP/1.1 with WebSocket for REST APIs?
*   **Kelebihan HTTP/2:** 
    *   **Multiplexing:** Banyak *request* dan *response* dapat dikirim secara bersamaan melalui satu koneksi TCP tunggal tanpa *blocking*.
    *   **Header Compression (HPACK):** Mengompresi ukuran *header* secara drastis sehingga menghemat *bandwidth*.
    *   **Format Biner:** Membaca data biner jauh lebih cepat bagi mesin daripada membaca teks (HTTP/1.1).
*   **Kekurangan HTTP/2:**
    *   **Sulit di-debug:** Karena formatnya biner, kita tidak bisa lagi menggunakan *tools* sederhana seperti `curl` atau sekadar membaca teks jaringan via *browser devtools* dengan mudah.
    *   **Kompatibilitas:** Masih ada beberapa infrastruktur lama (*load balancer* atau *proxy* lawas) yang tidak mendukung penuh HTTP/2.

### 9. How does the request-response model of REST APIs contrast with the bidirectional streaming capabilities of gRPC in terms of real-time communication and responsiveness?
*   **REST (Request-Response):** Arsitekturnya bersifat *pull-based* (klien harus meminta). Untuk membuat aplikasi *real-time*, klien harus melakukan *Long Polling* atau bertanya terus-menerus (*polling*) ke server apakah ada pesan baru. Ini sangat lambat, membuang banyak koneksi jaringan, dan boros CPU.
*   **gRPC (Bidirectional Streaming):** Memungkinkan komunikasi dua arah yang bersifat *push-based* pada satu koneksi presisten. Begitu ada pesan baru di server, server bisa langsung "mendorong" (push) pesan tersebut ke klien tanpa klien harus bertanya. Hasilnya adalah latensi yang sangat rendah dan sangat responsif.

### 10. What are the implications of the schema-based approach of gRPC, using Protocol Buffers, compared to the more flexible, schema-less nature of JSON in REST API payloads?
*   **Protobuf (Schema-based):** 
    *   Mewajibkan definisi struktur data (*contract*) secara jelas di awal (`.proto`).
    *   Ukuran *payload* sangat kecil dan proses serialisasi/deserialisasi sangat cepat.
    *   Menjamin tipe data aman (*type safety*), sehingga klien tidak akan menerima tipe data yang salah.
    *   *Trade-off:* Kurang fleksibel. Jika ada perubahan struktur, file `.proto` harus diubah dan klien/server harus di-*compile* ulang.
*   **JSON (Schema-less):**
    *   Sangat fleksibel, struktur bebas diubah kapan saja tanpa perlu kompilasi ulang kontrak.
    *   Mudah dibaca oleh manusia (*human-readable*).
    *   *Trade-off:* Ukuran *payload* lebih besar, proses *parsing* lebih berat untuk CPU, dan rawan terjadi *error* pada *runtime* (misalnya klien mengharapkan angka tapi menerima string dari server).