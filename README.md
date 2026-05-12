a. Apa itu AMQP?

AMQP (Advanced Message Queuing Protocol) adalah protokol komunikasi standar yang digunakan untuk message broker. Protokol ini mendefinisikan bagaimana pesan dikirim, diterima, dan dikelola antar aplikasi melalui perantara yang disebut broker. AMQP menjamin pengiriman pesan yang andal (reliable) serta mendukung fitur seperti antrian (queue), routing, dan berbagai pola pengiriman pesan.

b. Apa arti `guest:guest@localhost:5672`?

Format tersebut adalah connection string untuk terhubung ke RabbitMQ. `guest` pertama merupakan username, sedangkan `guest` kedua adalah password yang digunakan untuk autentikasi. `localhost:5672` menunjukkan alamat host dan port tempat RabbitMQ berjalan. Port `5672` merupakan port default yang digunakan oleh protokol AMQP.

## Simulation Slow Subscriber

![Queue Menumpuk](screenshot_slow_queue.png)

Jumlah antrian menumpuk karena setiap kali publisher dijalankan, 5 pesan
langsung dikirim sekaligus. Subscriber yang dibuat lambat (delay 1 detik
per pesan) tidak mampu memproses pesan secepat publisher mengirimkannya.
Akibatnya pesan-pesan tersebut mengantri di RabbitMQ. Jika publisher
dijalankan 4 kali, ada 20 pesan dikirim, tapi subscriber hanya memproses
1 per detik.

Ini adalah keunggulan event-driven architecture: meski subscriber lambat,
sistem tidak crash. Pesan tersimpan aman di antrian dan akan diproses
satu per satu sesuai kapasitas subscriber.

## Reflection and Running at Least Three Subscribers

![Tiga Subscriber](screenshot_three_sub1.png)

![Tiga Subscriber](screenshot_three_sub2.png)

![Tiga Subscriber](screenshot_three_sub3.png)

![Queue Turun Cepat](screenshot_rabbitmq_three.png)

Dengan menjalankan 3 subscriber sekaligus, beban pemrosesan terbagi secara
otomatis. RabbitMQ mendistribusikan pesan secara round-robin ke subscriber
yang tersedia. Hasilnya, antrian berkurang jauh lebih cepat dibandingkan
hanya dengan 1 subscriber.

Ini adalah konsep horizontal scaling: saat satu subscriber tidak cukup,
tambah lebih banyak instance subscriber tanpa mengubah publisher atau broker
sama sekali. Sistem menjadi lebih responsif tanpa perubahan arsitektur besar.

Hal yang bisa diperbaiki dari kode saat ini:
1. URL koneksi AMQP masih hardcoded, sebaiknya pakai environment variable
   agar lebih aman dan mudah dikonfigurasi di environment berbeda.
2. Tidak ada error handling yang memadai, jika satu pesan gagal diproses,
   tidak ada retry mechanism yang jelas selain dead letter queue.
3. Data di publisher statis (nama hardcoded), bisa dibuat lebih dinamis
   dengan menerima input dari file atau command line argument.