# Subscriber — Modul 9 Software Architectures

Repositori ini berisi kode **Subscriber** untuk Tutorial A Modul 9 — Event-Driven Architecture menggunakan Rust dan RabbitMQ sebagai message broker.

---

## a. Apa itu AMQP?

AMQP (Advanced Message Queuing Protocol) adalah **protokol komunikasi standar** yang dirancang khusus untuk message broker. Protokol ini mendefinisikan secara detail bagaimana pesan dikirim, diterima, di-route, dan dikelola antar aplikasi melalui perantara yang disebut **broker**. AMQP menjamin pengiriman pesan yang andal (*reliable delivery*) bahkan dalam kondisi jaringan yang tidak stabil, karena pesan disimpan di broker hingga berhasil diterima oleh konsumen.

Selain itu, AMQP mendukung berbagai fitur canggih seperti **antrian (queue)**, **routing berbasis exchange**, **acknowledgement**, dan **dead letter queue** untuk menangani pesan yang gagal diproses. Protokol ini bersifat *open standard*, artinya dapat diimplementasikan oleh berbagai message broker seperti RabbitMQ, Apache Qpid, dan lainnya, sehingga aplikasi yang menggunakan AMQP tidak terikat pada satu vendor tertentu.

---

## b. Apa Arti `guest:guest@localhost:5672`?

Format `guest:guest@localhost:5672` adalah bagian dari **connection string** AMQP yang digunakan untuk terhubung ke RabbitMQ. Berikut penjelasan tiap bagiannya:

- **`guest` (pertama)** — username untuk autentikasi ke RabbitMQ
- **`guest` (kedua)** — password yang bersesuaian dengan username tersebut
- **`localhost`** — alamat host tempat RabbitMQ berjalan; `localhost` berarti RabbitMQ ada di mesin yang sama dengan program yang berjalan
- **`5672`** — nomor port default yang digunakan oleh protokol AMQP untuk koneksi non-SSL

Dengan kata lain, baris `amqp://guest:guest@localhost:5672` berarti: *"Hubungkan ke RabbitMQ yang berjalan di mesin ini, pada port 5672, menggunakan kredensial username `guest` dan password `guest`."* Kredensial `guest/guest` adalah akun default bawaan RabbitMQ yang biasa digunakan untuk keperluan development dan eksperimen lokal.

---

## Simulation Slow Subscriber

![Queue Menumpuk](screenshot_slow_queue.png)

Pada eksperimen ini, baris `std::thread::sleep(_ten_millis)` di-uncomment agar subscriber memproses setiap pesan dengan jeda waktu **1 detik per pesan**. Hal ini mensimulasikan kondisi di mana subscriber bekerja lebih lambat dari kecepatan publisher mengirimkan pesan, situasi yang sangat umum terjadi di sistem nyata, misalnya saat server sedang tinggi beban atau melakukan proses yang memakan waktu seperti query database atau pemanggilan API eksternal.

Hasilnya terlihat jelas di grafik RabbitMQ: jumlah antrian (*queue depth*) terus **menumpuk** setiap kali publisher dijalankan. Jika publisher dijalankan 4 kali berturut-turut, total 20 pesan masuk ke antrian, namun subscriber hanya mampu memproses 1 pesan per detik, sehingga butuh waktu 20 detik untuk mengosongkan antrian.

Inilah salah satu **keunggulan utama event-driven architecture**: meskipun subscriber berjalan lambat, sistem **tidak crash** dan tidak ada pesan yang hilang. Semua pesan tersimpan aman di antrian RabbitMQ dan akan diproses satu per satu sesuai kapasitas subscriber. Publisher pun tidak perlu menunggu, ia tetap bisa mengirim pesan kapan saja tanpa terganggu oleh kecepatan subscriber.

---

## Reflection and Running at Least Three Subscribers

![Subscriber Terminal 1](screenshot_three_sub1.png)
![Subscriber Terminal 2](screenshot_three_sub2.png)
![Subscriber Terminal 3](screenshot_three_sub3.png)
![Queue Turun Lebih Cepat](screenshot_rabbitmq_three.png)

Untuk mengatasi masalah slow subscriber, solusi yang diterapkan adalah menjalankan **3 instance subscriber secara bersamaan**. Ketiga subscriber ini terhubung ke queue `user_created` yang sama di RabbitMQ, dan broker secara otomatis mendistribusikan pesan menggunakan mekanisme **round-robin**, pesan pertama ke subscriber 1, pesan kedua ke subscriber 2, pesan ketiga ke subscriber 3, dan seterusnya secara bergilir.

Hasilnya sangat signifikan: antrian yang sebelumnya menumpuk kini berkurang **tiga kali lebih cepat** dibandingkan hanya menggunakan satu subscriber. Dengan 3 subscriber yang masing-masing memproses 1 pesan per detik, kapasitas total pemrosesan menjadi 3 pesan per detik. Grafik di RabbitMQ dashboard menunjukkan penurunan antrian yang jauh lebih curam dibandingkan skenario single subscriber.

Ini adalah implementasi nyata dari konsep **horizontal scaling**: ketika satu instance tidak cukup menangani beban, kita cukup menambah lebih banyak instance tanpa perlu mengubah kode publisher, kode subscriber, maupun konfigurasi broker. Arsitektur tetap bersih dan tidak ada komponen yang perlu dimodifikasi.

### Hal yang Bisa Diperbaiki dari Kode Saat Ini

1. **URL koneksi AMQP masih hardcoded** : nilai `amqp://guest:guest@localhost:5672` tertanam langsung di kode. Sebaiknya dipindahkan ke **environment variable** agar lebih aman, mudah dikonfigurasi di berbagai environment (development, staging, production), dan tidak ada kredensial sensitif yang ikut ter-commit ke repository.

2. **Error handling yang belum memadai** : saat ini jika terjadi kegagalan saat memproses pesan, tidak ada mekanisme **retry** yang eksplisit selain dead letter queue. Idealnya ada logika retry dengan exponential backoff agar pesan yang gagal diproses dapat dicoba ulang secara otomatis sebelum masuk ke dead letter queue.

3. **Data publisher masih statis** :  nama-nama pengguna (`Amir`, `Budi`, dll.) saat ini di-hardcode langsung di `main.rs`. Untuk penggunaan yang lebih realistis, data ini sebaiknya dibaca dari file eksternal, database, atau diterima sebagai input dari command line argument, sehingga publisher bisa lebih fleksibel dan mudah diuji dengan berbagai skenario data.

---

# Bonus: Running on Cloud (CloudAMQP)

Sebagai bonus, seluruh eksperimen pada tutorial ini dijalankan ulang menggunakan **CloudAMQP** sebagai message broker berbasis cloud, menggantikan RabbitMQ yang sebelumnya dijalankan secara lokal via Docker.

---

### Mengapa Cloud?

Salah satu kekuatan terbesar event-driven architecture adalah kemampuannya untuk berjalan secara **terdistribusi**. Dengan memindahkan broker ke cloud menggunakan CloudAMQP, kita membuktikan bahwa subscriber tidak perlu berada di mesin yang sama dengan publisher. Selama keduanya terhubung ke URL CloudAMQP yang sama, komunikasi event-driven tetap berjalan dengan sempurna melalui internet.

Hal ini juga membuktikan bahwa arsitektur ini benar-benar **loosely coupled** dan **location-independent** : komponen-komponen sistem bisa berada di lokasi geografis yang berbeda, berjalan di mesin yang berbeda, bahkan dalam timezone yang berbeda, namun tetap dapat berkomunikasi melalui broker di cloud.

---

### Perubahan yang Dilakukan

URL koneksi AMQP pada subscriber diubah dari `localhost` ke URL yang disediakan oleh CloudAMQP:

```
// Sebelum (lokal):
amqp://guest:guest@localhost:5672

// Sesudah (cloud):
amqp://username:password@broker.cloudamqp.com/username
```

Hanya satu baris yang perlu diubah di `src/main.rs`, dan tidak ada perubahan logika apapun yang diperlukan. Ini menunjukkan betapa fleksibel dan portabelnya kode yang menggunakan protokol AMQP standar.

---

### Hasil Simulasi Slow Subscriber di Cloud

Simulasi slow subscriber berhasil dijalankan sepenuhnya di cloud. Antrian pesan tetap **menumpuk di CloudAMQP** ketika subscriber berjalan lambat, persis seperti perilaku yang terlihat saat menggunakan broker lokal. Kemudian saat 3 subscriber dijalankan sekaligus dan semuanya terhubung ke CloudAMQP, antrian berkurang jauh lebih cepat.

Hal ini membuktikan bahwa konsep **horizontal scaling** dan mekanisme **antrian event-driven** bekerja dengan cara yang identik baik di lingkungan lokal maupun di cloud. Perpindahan dari broker lokal ke broker cloud tidak mempengaruhi perilaku sistem sama sekali.

![CloudAMQP Subscriber](screenshot_cloudamqp_subscriber.png)

---

### Running at Least Three Subscribers (Cloud)

![Subscriber 1 Cloud](image-4.png)
![Subscriber 2 Cloud](image-6.png)
![Subscriber 3 Cloud](image-5.png)
![Queue CloudAMQP](image-7.png)

Dengan tiga subscriber yang terhubung ke CloudAMQP secara bersamaan, RabbitMQ di cloud mendistribusikan pesan secara round-robin ke ketiga subscriber tersebut. Grafik di dashboard CloudAMQP menunjukkan penurunan antrian yang signifikan, membuktikan bahwa horizontal scaling bekerja sama efektifnya di cloud seperti di lingkungan lokal.
