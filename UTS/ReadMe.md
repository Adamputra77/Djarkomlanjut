# Laporan Analisis UTS — Detail (Markdown)

**Nama:** Adam Putra Pratama
**NIM:** 20230801402
**Mata Kuliah:** Jaringan Komputer Lanjut
**Dosen:** Jefry Sunupurwa Asri, S.Kom., M.Kom.
**File simulasi:** `UTS FINAL CISCO`
**Alat simulasi:** Cisco Packet Tracer

---

## Ringkasan singkat

Simulasi merupakan jaringan LAN sederhana: beberapa PC, sebuah server, sebuah switch, router (mengarah ke cloud) dan sebuah access point/laptop. Pengujian utama dilakukan dengan `ping` dari PC0 ke Server0 (IP server: `192.168.20.10`). Hasil ping menunjukkan 4 paket terkirim, 3 diterima, 1 hilang → **25% packet loss** (pada paket pertama muncul `Request timed out`, sisanya reply <1ms). Analisis berikut menjelaskan struktur, konfigurasi, interpretasi hasil, penyebab kemungkinan, serta rekomendasi perbaikan dan pengujian lanjutan.

---

## 1. Topologi dan Komponen

* **Router:** Cisco 1941 — menghubungkan jaringan lokal ke cloud (gateway).
* **Switch:** Cisco 2960 — pusat koneksi semua perangkat wired.
* **PC:** PC0, PC1, PC2 — end-host untuk pengujian.
* **Server:** Server0 — host tujuan ping.
* **Laptop / Access Point:** perangkat nirkabel (simulasi).
* **Cloud:** simulasi koneksi eksternal.
  Semua perangkat wired terhubung ke switch; switch trunk/port ke router sebagai uplink menuju cloud/router.

---

## 2. Alamat IP & Subnet (yang dipakai di simulasi)

Tabel IP (versi yang digunakan pada laporan PDF):

| Perangkat | IP Address    | Subnet Mask   | Gateway      |
| --------- | ------------- | ------------- | ------------ |
| PC0       | 192.168.10.10 | 255.255.255.0 | 192.168.10.1 |
| PC1       | 192.168.20.10 | 255.255.255.0 | 192.168.20.1 |
| PC2       | 192.168.30.10 | 255.255.255.0 | 192.168.30.1 |
| Server0   | 192.168.50.10 | 255.255.255.0 | 192.168.50.1 |
| Laptop1   | 192.168.50.8  | 255.255.255.0 | 192.168.50.1 |

> Gateway (`192.168.20.1`) diasumsikan di router atau subinterface router pada skenario single-subnet (tidak memakai VLAN terpisah pada laporan ini).

---

## 3. Perintah/konfigurasi penting (contoh)

Berikut contoh perintah yang biasa dipakai di Cisco IOS untuk konfigurasi dasar pada skenario single-subnet:

**Switch (port akses):**

```
Switch(config)# interface fa0/2
Switch(config-if)# switchport mode access
Switch(config-if)# switchport access vlan 20
```

**Port trunk ke router (jika menggunakan VLAN/trunk):**

```
Switch(config)# interface fa0/1
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk allowed vlan 20
```

**Router (gig0/0 sebagai interface ke switch; contoh tanpa subinterface):**

```
Router(config)# interface gig0/0
Router(config-if)# ip address 192.168.20.1 255.255.255.0
Router(config-if)# no shutdown
```

> Catatan: Jika topologi awal menggunakan satu subnet saja, tidak perlu konfigurasi subinterface. Jika inter-VLAN diperlukan, gunakan Router-on-a-Stick (subinterface + encaps dot1Q).

---

## 4. Hasil pengujian `ping` (detail)

Output yang didapat (direkam di screenshot):

```
Pinging 192.168.20.10 with 32 bytes of data:
Request timed out.
Reply from 192.168.20.10: bytes=32 time<1ms TTL=127
Reply from 192.168.20.10: bytes=32 time<1ms TTL=127
Reply from 192.168.20.10: bytes=32 time<1ms TTL=127

Ping statistics for 192.168.20.10:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
    Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
```

**Interpretasi langkah demi langkah:**

1. Paket 1: `Request timed out.` → tidak mendapat balasan (hilang atau tidak sempat dibalas).
2. Paket 2–4: mendapat balasan dengan latency sangat kecil (<1ms) → jalur fisik/logikal berfungsi baik.
3. Statistik: 1 dari 4 paket hilang → ada gangguan sesaat.

---

## 5. Kemungkinan penyebab packet loss (25% pada pengujian ini)

Urutan penyebab potensial, dari yang paling umum sampai yang kurang mungkin pada simulasi Packet Tracer:

1. **Inisialisasi koneksi/perangkat baru:** perangkat (server atau interface) baru aktif dan sedang menyelesaikan ARP/initialization sehingga paket pertama tidak direspon.
2. **ARP lookup:** saat PC mengirim ping pertama, PC harus melakukan ARP untuk menemukan MAC server/gateway — paket ICMP pertama terkirim sebelum ARP selesai → hilang.
3. **Port flapping atau interface down saat paket pertama:** interface pada switch/router mungkin belum aktif (cek status `show interface`).
4. **Queue/processing delay pada switch/router:** antrian paket singkat bisa menyebabkan drop paket awal. Di simulasi ini kecil kemungkinannya tapi masih mungkin.
5. **ACL atau firewall rules (di server atau router):** kebijakan akses mungkin menolak paket awal. Jika ada ACL yang ketat, cek konfig.
6. **Kesalahan konfigurasi IP (duplicate IP) atau konflik ARP:** jika ada perangkat lain memakai IP sama, bisa menyebabkan paket hilang atau balasan tak konsisten.
7. **Masalah kabel logis di simulasi:** kabel yang terhubung bukan jenis yang semestinya (mis. wrong cable type) — di Packet Tracer biasanya otomatis jadi benar, tapi perlu dicek.

---

## 6. Langkah pemeriksaan & troubleshooting (urutan tindakan yang disarankan)

Lakukan langkah-langkah ini di Packet Tracer untuk menemukan akar masalah:

1. **Cek status interface:**

   * Router: `show ip interface brief` / `show interface gig0/0`
   * Switch: `show interface status`
     Pastikan semua interface `up/up` atau `up/connected`.

2. **Cek ARP table pada PC dan router/server:**

   * Di PC (Packet Tracer GUI) lihat ARP cache.
   * Router: `show ip arp`
     Jika ARP entry belum ada sebelum ping, ARP akan dibuat saat pengujian; itu menjelaskan paket pertama hilang.

3. **Uji ping berkala (extended):**

   ```
   ping 192.168.20.10 repeat 10
   ```

   Amati apakah packet loss tetap terjadi hanya pada paket pertama atau acak.

4. **Cek tabel routing (di router):**

   * `show ip route` — pastikan ada rute untuk 192.168.20.0/24.

5. **Cek konfigurasi IP pada server dan PC:**

   * Pastikan subnet mask, gateway, dan IP benar (tidak duplikat).

6. **Periksa firewall/ACL di server (jika ada simulasi service firewall):**

   * Nonaktifkan sementara firewall untuk pengujian.

7. **Observasi log (jika tersedia):**

   * Router/switch logging, event pada server.

8. **Uji koneksi layer-2:**

   * `show mac address-table` di switch — pastikan MAC server dan PC tercatat pada port yang benar.

9. **Ganti kabel / port di simulasi:**

   * Coba pindahkan server ke port switch lain untuk mengeliminasi masalah port.

10. **Uji dari perangkat lain:**

    * Lakukan ping dari PC1/PC2 ke Server0 untuk melihat pola apakah hanya PC0 yang mengalami loss.

---

## 7. Rekomendasi perbaikan & pencegahan

* **Jika penyebab ARP/initialization:** beri jeda sejenak setelah menyalakan perangkat, atau jalankan ARP resolve terlebih dahulu (`arp -a` / ping sekali lalu ulangi test).
* **Jika ada konflik IP:** perbaiki IP duplikat segera.
* **Jika ACL/firewall:** tambahkan rule yang mengizinkan ICMP antara host yang relevan atau matikan sementara saat testing.
* **Monitoring berkala:** jalankan ping berulang dan monitoring `show` pada perangkat untuk mendeteksi intermittent issue.
* **Dokumentasi konfigurasi:** simpan konfigurasi router/switch sehingga konfigurasi yang benar dapat dipulihkan jika terjadi modifikasi yang tidak diinginkan.
* **Gunakan SNMP/log untuk jaringan nyata:** di jaringan produksi gunakan monitoring untuk mendeteksi packet loss dan latency. (Di Packet Tracer cukup observasi manual.)

---

## 8. Rekomendasi pengujian lanjutan

* Lakukan `ping` dengan jumlah paket lebih banyak (mis. 50 atau 100) untuk statistik yang lebih memadai.
* Gunakan `traceroute` untuk melihat jalur paket jika jaringan lebih kompleks.
* Tes transfer file atau aplikasi (mis. HTTP) untuk memastikan packet loss tidak mengganggu layanan.
* Jika menggunakan VLAN di topologi akhir: verifikasi trunk, subinterface router, dan tag VLAN (dot1Q). Pastikan `switchport trunk allowed` mencakup VLAN yang dipakai.

---

## 9. Kesimpulan (per hal)

* Konektivitas dasar antara PC0 dan Server0 **berfungsi** (bukti: 3/4 paket mendapat reply <1ms).
* Terdapat **packet loss 25%** pada pengujian singkat; kemungkinan besar bersifat **sementara** (mis. ARP/inisialisasi) pada simulasi.
* Dengan langkah troubleshooting sederhana (cek ARP, interface status, routing, dan firewall), masalah ini dapat diidentifikasi dan diatasi.
* Setelah perbaikan/verifikasi, ulangi pengujian untuk memastikan packet loss tidak lagi terjadi.

---

## 10. Lampiran — Perintah cepat untuk troubleshooting

* `show ip interface brief`
* `show interfaces`
* `show ip arp`
* `show mac address-table` (switch)
* `show running-config` (untuk verifikasi konfigurasi)
* `ping 192.168.20.10 repeat 20` (uji 20 packet)
* `traceroute 192.168.20.10`

---
