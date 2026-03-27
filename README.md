# Blockchain Technology - Simulasi Blockchain Multi-Node

> Simulasi blockchain multi-node berbasis Python dan Flask yang mengimplementasikan wallet, transaksi bertanda tangan digital, Proof of Work, miner reward, dan sinkronisasi chain antar-node melalui HTTP.

| Nama | NRP |
|------|-----|
| Danar Bagus Rasendriya | 5027231055 |
| Diandra N. A. | 5027231004 |
| Chalvin Reza F. | 5025221054 |
| Tio Axellino I. | 5027231065 |


![Python](https://img.shields.io/badge/Python-3.x-3776AB?logo=python&logoColor=white)
![Flask](https://img.shields.io/badge/Flask-3.x-000000?logo=flask&logoColor=white)
![Cryptography](https://img.shields.io/badge/Cryptography-ECDSA%20secp256k1-2E7D32)
![Consensus](https://img.shields.io/badge/Consensus-Longest%20Valid%20Chain-1565C0)
![Storage](https://img.shields.io/badge/Storage-In%20Memory-6D4C41)
![Status](https://img.shields.io/badge/Status-Academic%20Project-blue)

## Daftar Isi / Table of Contents

- [Daftar Isi / Table of Contents](#daftar-isi--table-of-contents)
- [Deskripsi Proyek / Project Overview](#deskripsi-proyek--project-overview)
- [Fitur Utama / Key Features](#fitur-utama--key-features)
- [Arsitektur Sistem / System Architecture](#arsitektur-sistem--system-architecture)
- [Teknologi yang Digunakan / Tech Stack](#teknologi-yang-digunakan--tech-stack)
- [Instalasi & Konfigurasi / Installation & Setup](#instalasi--konfigurasi--installation--setup)
- [Menjalankan Aplikasi / Running the Application](#menjalankan-aplikasi--running-the-application)
- [Tampilan Web UI / Web UI Overview](#tampilan-web-ui--web-ui-overview)
- [Dokumentasi Pengujian / Testing Documentation](#dokumentasi-pengujian--testing-documentation)
- [10.1 Penambahan Transaksi / Adding Transactions](#101-penambahan-transaksi--adding-transactions)
- [10.2 Proses Mining / Mining Process](#102-proses-mining--mining-process)
- [10.3 Reward Miner / Miner Reward](#103-reward-miner--miner-reward)
- [10.4 Validasi Digital Signature / Digital Signature Validation](#104-validasi-digital-signature--digital-signature-validation)
- [10.5 Sinkronisasi Antar-Node / Node Synchronization](#105-sinkronisasi-antar-node--node-synchronization)
- [10.6 Broadcast Transaksi ke Peer / Broadcasting Transactions](#106-broadcast-transaksi-ke-peer--broadcasting-transactions)
- [10.7 Daftar Peer Node / Registered Peer Nodes](#107-daftar-peer-node--registered-peer-nodes)
- [Implementasi Kriptografi / Cryptography Implementation](#implementasi-kriptografi--cryptography-implementation)
- [Endpoint API / API Reference](#endpoint-api--api-reference)
- [Arsitektur P2P / P2P Network Architecture](#arsitektur-p2p--p2p-network-architecture)
- [Penafian / Disclaimer](#penafian--disclaimer)

## Deskripsi Proyek / Project Overview

Proyek ini membangun sebuah node blockchain yang berjalan sebagai aplikasi web Flask. Setiap node memiliki wallet sendiri, blockchain lokal sendiri, kumpulan `pending_transactions`, dan daftar peer yang dapat dipakai untuk sinkronisasi chain. Tujuan utamanya adalah memperlihatkan alur inti blockchain secara konkret: pembuatan transaksi, digital signature, validasi transaksi, mining block baru, pemberian reward kepada miner, dan consensus berbasis `longest valid chain`.

Konsep blockchain yang diimplementasikan meliputi:

- `Proof of Work` berbasis pencarian `nonce` sampai hash block memiliki prefiks nol sejumlah `difficulty`.
- `Digital signature` menggunakan ECDSA di kurva `secp256k1`.
- `SHA-256` untuk hashing payload transaksi, hashing block, dan pembentukan address dari public key.
- `Pending transaction pool` untuk menyimpan transaksi sebelum ditambang.
- `Mining reward` sebesar `25.0` coin melalui transaksi khusus dari `SYSTEM`.
- `Chain validation` yang memeriksa integritas hash, relasi `previous_hash`, aturan reward, validitas signature, dan overspending.
- `Peer-to-peer synchronization` berbasis HTTP dengan aturan penggantian ke chain valid yang lebih panjang.

Seluruh state disimpan di memori proses, sehingga saat node di-restart, wallet, chain, dan saldo akan dibentuk ulang dari awal. Tidak ada broadcast transaksi atau block secara otomatis; sinkronisasi antar-node dilakukan secara eksplisit melalui endpoint `GET /nodes/resolve`.

## Fitur Utama / Key Features

- Wallet otomatis per node dengan pasangan private key dan public key ECDSA `secp256k1`.
- Address wallet diturunkan dari hash SHA-256 public key PEM, lalu dipotong menjadi 40 karakter heksadesimal.
- Pembuatan transaksi lokal dengan auto-signing dari wallet node pengirim.
- Validasi transaksi mencakup amount positif, recipient wajib ada, signature valid, kecocokan address dengan public key, larangan self-transfer, dan pengecekan `spendable_balance`.
- Penyimpanan transaksi pending sebelum dimasukkan ke block baru.
- Deteksi duplikasi transaksi di mempool maupun chain.
- Mining block dengan difficulty default `3` dan transaksi reward `25.0` coin.
- Validasi chain yang mewajibkan tepat satu reward transaction per block, nilainya benar, dan posisinya berada di akhir daftar transaksi.
- Registrasi peer melalui argumen CLI `--peers` maupun endpoint `POST /nodes/register`.
- Sinkronisasi chain antar-node melalui `GET /nodes/resolve` dengan aturan `longest valid chain`.
- Broadcast transaksi dari pending pool ke semua peer terdaftar melalui `POST /transactions/broadcast`, memungkinkan propagasi transaksi secara eksplisit tanpa menunggu proses `resolve`.
- Dashboard web untuk melihat wallet, peer, pending transactions, blockchain, dan respons API terakhir.

## Arsitektur Sistem / System Architecture

Secara struktural, satu proses Flask merepresentasikan satu node blockchain. Node tersebut dibungkus oleh `NodeState` di `app.py`, yang menggabungkan wallet, blockchain, dan daftar peer.

```text
Browser / Postman
        |
        v
+------------------------------+
| Flask App (app.py)           |
| Routes HTTP                  |
| /chain /wallet /mine /...    |
+--------------+---------------+
               |
               v
+------------------------------+
| NodeState                    |
| - node_url                   |
| - wallet: Wallet             |
| - blockchain: Blockchain     |
| - peer_nodes: set[str]       |
+---------+--------------------+
          |                         +----------------------+
          |                         | Peer Nodes           |
          | resolve_chain() ------> | GET /chain via HTTP  |
          |                         +----------------------+
          |
          +--> Wallet
          |    - private/public key
          |    - address
          |    - sign_transaction()
          |
          +--> Blockchain
               - chain
               - pending_transactions
               - validate_transaction()
               - mine_pending_transactions()
               - is_valid()
               - replace_chain()
```

Alur kerja sistem:

1. Saat node dijalankan, `NodeState` membuat wallet baru dan blockchain baru dengan genesis block deterministik.
2. Transaksi lokal dibuat oleh wallet node pengirim dan otomatis ditandatangani.
3. Blockchain memvalidasi transaksi lalu menyimpannya ke `pending_transactions`.
4. Endpoint `POST /mine` membuat reward transaction dari `SYSTEM`, menambang block baru, lalu menambahkan block tersebut ke chain.
5. Endpoint `GET /nodes/resolve` menghubungi peer, mengambil data dari `GET /chain`, lalu mengganti chain lokal jika peer memiliki chain yang lebih panjang dan valid.

Poin penting arsitektur ini adalah bahwa setiap node menyimpan data secara independen. Jika Node A mining transaksi ke dalam chain-nya, Node B tidak langsung mengetahui perubahan itu sampai Node B memanggil `GET /nodes/resolve`.

## Teknologi yang Digunakan / Tech Stack

| Komponen | Teknologi | Peran dalam Proyek |
|----------|-----------|--------------------|
| Bahasa utama | Python | Implementasi blockchain, wallet, consensus, dan HTTP server |
| Web framework | Flask (`>=3.0,<4.0`) | Menyediakan endpoint API dan dashboard web |
| Library kriptografi | `cryptography` (`>=42.0,<46.0`) | ECDSA `secp256k1`, serialisasi key PEM, dan verifikasi signature |
| Hashing | `hashlib` (standard library) | SHA-256 untuk hash block, transaction ID, dan address |
| HTTP client antar-node | `urllib.request` (standard library) | Mengambil chain peer saat proses resolve |
| Serialisasi data | `json` (standard library) | Canonical JSON untuk hashing dan signing, serta payload API |
| Antarmuka web | HTML5, CSS3, Vanilla JavaScript | Dashboard untuk inspeksi node dan trigger API |
| Konfigurasi CLI | `argparse` (standard library) | Mengatur `--host`, `--port`, dan `--peers` saat startup |

## Instalasi & Konfigurasi / Installation & Setup

### Prasyarat / Prerequisites

- Python 3 dan `pip`
- Terminal atau command prompt
- Virtual environment

### Membuat Virtual Environment / Create a Virtual Environment

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Pada Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
```

### Instalasi Dependency / Install Dependencies

```bash
pip install -r requirements.txt
```

### Konfigurasi Environment / Environment Configuration

Proyek ini tidak menyediakan `.env` maupun `.env.example`, karena seluruh konfigurasi runtime dilakukan melalui argumen command line:

- `--host`: host Flask, default `127.0.0.1`
- `--port`: port Flask, default `5001`
- `--peers`: daftar peer node dalam format URL

Contoh konfigurasi startup:

```bash
python app.py --host 127.0.0.1 --port 5001 --peers http://127.0.0.1:5002 http://127.0.0.1:5003
```

## Menjalankan Aplikasi / Running the Application

### Menjalankan Satu Node / Run a Single Node

```bash
python app.py --host 127.0.0.1 --port 5001
```

Node akan tersedia di:

- `http://127.0.0.1:5001/` untuk dashboard HTML
- `http://127.0.0.1:5001/chain` untuk melihat blockchain dalam format JSON

### Menjalankan Beberapa Node / Run Multiple Nodes

Jalankan tiap node pada terminal berbeda agar dapat menguji sinkronisasi antar-node.

Terminal 1:

```bash
python app.py --port 5001 --peers http://127.0.0.1:5002 http://127.0.0.1:5003
```

Terminal 2:

```bash
python app.py --port 5002 --peers http://127.0.0.1:5001 http://127.0.0.1:5003
```

Terminal 3:

```bash
python app.py --port 5003 --peers http://127.0.0.1:5001 http://127.0.0.1:5002
```

### Catatan Operasional / Operational Notes

- Setiap start ulang node akan menghasilkan wallet baru, sehingga address akan berubah.
- Saldo awal semua wallet adalah `0.0` karena genesis block tidak berisi transaksi.
- Sebelum node dapat mengirim transaksi, node tersebut harus memperoleh coin terlebih dahulu, misalnya dengan mining.
- Sinkronisasi antar-node tidak berjalan otomatis; node yang tertinggal harus memanggil `GET /nodes/resolve`.

## Web UI

**Screenshot:**
<img width="1228" height="1570" alt="screenshot-web-ui" src="https://github.com/user-attachments/assets/a2a40597-5b09-442f-86d4-f247b230d841" />

## Dokumentasi Pengujian / Testing Documentation

### 10.0 Proses Mining Awal

Agar dapat melakukan transaksi, sebuah node (pada konteks ini node 5001) harus memiliki balance terlebih dahulu. Maka dilakukan proses mining pertama kali.

**Screenshot:**
<img width="2083" height="1737" alt="00-proses-mining-awal" src="https://github.com/user-attachments/assets/f18fd3f4-ef41-4f33-ba03-1f3ed882c766" />

### 10.1 Penambahan Transaksi / Adding Transactions

Endpoint utama untuk menambahkan transaksi adalah `POST /transactions/new`. Endpoint ini mendukung dua mode:

- Mode sederhana: cukup kirim `recipient_address` dan `amount`, lalu node akan membuat dan menandatangani transaksi menggunakan wallet lokalnya.
- Mode penuh: kirim seluruh payload transaksi yang sudah ditandatangani (`sender_address`, `recipient_address`, `amount`, `timestamp`, `signature`, `public_key`).

Pastikan node pengirim sudah memiliki saldo yang cukup dari hasil mining.

Request:

```bash
curl -X POST http://127.0.0.1:5001/transactions/new \
  -H "Content-Type: application/json" \
  -d '{
    "recipient_address": "WALLET ADDRESS PENERIMA",
    "amount": 10
  }'
```

Body minimal yang diharapkan:

```json
{
  "recipient_address": "WALLET ADDRESS PENERIMA",
  "amount": 10
}
```

Hasil sukses akan mengembalikan field seperti `message`, `transaction`, `signature_valid`, dan `pending_count`. Transaksi yang valid akan menghasilkan `signature_valid: true` dan masuk ke daftar `pending_transactions`.

**Screenshot:**
<img width="2094" height="1744" alt="01-penambahan-transaksi" src="https://github.com/user-attachments/assets/f3f37797-be43-4007-b062-49fa6e0a323d" />

### 10.2 Proses Mining / Mining Process

Mining dilakukan melalui endpoint `POST /mine`. Implementasinya bekerja sebagai berikut:

1. Seluruh transaksi di `pending_transactions` divalidasi ulang.
2. Sistem membuat reward transaction dari `SYSTEM` ke address miner.
3. Block baru dibentuk menggunakan `previous_hash` dari block terakhir.
4. Fungsi `mine_block()` menaikkan `nonce` sampai hash block diawali `difficulty` buah angka nol.

Pada konfigurasi default proyek ini, `difficulty = 3`, sehingga hash block yang valid harus diawali pola `000`.

Request:

```bash
curl -X POST http://127.0.0.1:5001/mine \
  -H "Content-Type: application/json" \
  -d '{}'
```

Endpoint ini mengembalikan block baru, wallet miner setelah mining, dan panjang chain terbaru. Saat diuji, mining sukses menghasilkan block dengan hash berprefiks `000`, `chain_length` bertambah, dan daftar pending transaction menjadi kosong karena transaksi sudah masuk ke block.

**Screenshot:**
<img width="2094" height="1744" alt="02-proses-mining" src="https://github.com/user-attachments/assets/7f5c9def-1596-4e03-9830-b06477a042ff" />

### 10.3 Reward Miner / Miner Reward

Mekanisme reward diimplementasikan melalui transaksi khusus yang dibuat oleh fungsi `create_reward_transaction()`. Nilai reward default di kode adalah `25.0`, dengan format:

- `sender_address = "SYSTEM"`
- `recipient_address` berisi address wallet miner
- `amount = 25.0`
- `signature = ""`
- `public_key = ""`

Transaksi reward selalu ditempatkan sebagai transaksi terakhir dalam block. Aturan ini juga divalidasi kembali saat chain diperiksa melalui `is_valid()`.

Untuk melihat reward miner, lakukan mining lalu cek wallet miner atau lihat block terbaru di chain:

```bash
curl http://127.0.0.1:5001/wallet
curl http://127.0.0.1:5001/chain
```

Dalam pengujian runtime, setelah mining kedua pada node pengirim, block berisi dua transaksi: transaksi user yang dipendingkan sebelumnya dan reward transaction dari `SYSTEM` sebesar `25.0` di posisi terakhir.

**Screenshot:**
<img width="2094" height="1744" alt="03-reward-miner" src="https://github.com/user-attachments/assets/55d7409c-977d-4d22-aa68-2bb50c6be880" />

**Screenshot:**
Validasi Block
<img width="2094" height="1744" alt="03-reward-miner-block" src="https://github.com/user-attachments/assets/0b88d729-5152-422b-8af8-d90a770c0ee1" />

### 10.4 Validasi Digital Signature / Digital Signature Validation

Validasi signature dilakukan melalui endpoint `POST /transactions/validate`. Endpoint ini membutuhkan payload transaksi lengkap:

- `sender_address`
- `recipient_address`
- `amount`
- `timestamp`
- `signature`
- `public_key`

Cara paling akurat untuk menguji endpoint ini adalah mengambil objek `transaction` dari response `POST /transactions/new`, lalu mengirimkannya kembali tanpa perubahan.

Request:

```bash
curl -X POST http://127.0.0.1:5001/transactions/validate \
  -H "Content-Type: application/json" \
  -d '{
    "sender_address": "66f65e06dbcc351bd1c65630a08d44cf1d083420",
    "recipient_address": "cb3996d886eb2a89348da1ceb49c2e488078f1f9",
    "amount": 10.0,
    "timestamp": "2026-03-26T19:11:45Z",
    "signature": "3045022038dfc282e300f97d641ba58644f6948f6fa24029deded702aae8d0043bf55d33022100ec1c2044274a9b1dc3f03173d2d8a26ed7844de778d4b25a3f8e416f7b29ebe6",
    "public_key": "-----BEGIN PUBLIC KEY-----\nMFYwEAYHKoZIzj0CAQYFK4EEAAoDQgAEiTe+7l/htCCTQPyR4WyvNxb0/g7McPuL\nK1mUUmbQNkvQ3UFTvgutrtsliDN1I+d5/Y55Deki7669LYah1lCzOQ==\n-----END PUBLIC KEY-----"
  }'
```

Hasil valid akan mengembalikan `valid: true` dan pesan `Digital signature valid.`. Saat payload yang sama dimodifikasi, misalnya `amount` diubah tanpa mengubah `signature`, endpoint akan menolak dengan `valid: false` dan pesan `Digital signature verification failed.`.

**Screenshot:**
<img width="2094" height="1744" alt="04-validasi-digital-signature" src="https://github.com/user-attachments/assets/3d74daa8-3418-4c97-8e71-0d18cc95df9d" />

**Screenshot:**
Validasi jika amount diubah, sehingga tidak valid
<img width="2094" height="1744" alt="04-validasi-digital-signature-invalid" src="https://github.com/user-attachments/assets/e76fdc30-320a-4fe8-8d42-691a7d46e004" />

### 10.5 Sinkronisasi Antar-Node / Node Synchronization

Node tidak menemukan peer secara otomatis. Peer harus didaftarkan dengan salah satu cara berikut:

- Dikirim saat startup melalui argumen `--peers`
- Dikirim setelah startup melalui `POST /nodes/register`

Sinkronisasi chain dilakukan melalui `GET /nodes/resolve`. Saat endpoint ini dipanggil, node lokal akan:

1. Mengakses endpoint `GET /chain` milik setiap peer.
2. Membaca `length` dan isi `chain` dari peer.
3. Melakukan deserialisasi chain kandidat menjadi objek `Block`.
4. Memvalidasi chain kandidat.
5. Mengganti chain lokal jika kandidat lebih panjang dan valid.

Contoh request sinkronisasi:

```bash
curl http://127.0.0.1:5002/nodes/resolve
```

Hasil verifikasi runtime menunjukkan bahwa node penerima transaksi tetap memiliki balance `0.0` sebelum `resolve`, lalu berubah mengikuti chain baru setelah `resolve` berhasil. Pada pengujian, Node 2 mengganti chain lokalnya dengan chain dari Node 1 dan balance wallet penerima berubah menjadi `10.0`.

**Screenshot:**
Node 5002 SEBELUM Resolve (Sinkronisasi) - LIHAT BALANCE
<img width="2094" height="1744" alt="05-sinkronisasi-antar-node-before" src="https://github.com/user-attachments/assets/7d9ddc73-602a-42d5-a3d3-0bdb69679dcf" />

**Screenshot:**
Proses Resolve
<img width="2094" height="1744" alt="05-sinkronisasi-antar-node-resolve" src="https://github.com/user-attachments/assets/fd200aed-ef64-4388-b637-4a9bcdf20306" />

**Screenshot:**
Node 5002 SETELAH Resolve - Balance bertambah, status `replaced: true`, `source: http://127.0.0.1:5001`
<img width="2094" height="1744" alt="05-sinkronisasi-antar-node-after-resolve" src="https://github.com/user-attachments/assets/4357f7c7-ffd5-45ac-a57c-5332b5475521" />

### 10.6 Broadcast Transaksi ke Peer / Broadcasting Transactions

Endpoint `POST /transactions/broadcast` memungkinkan propagasi transaksi dari pending pool satu node ke semua peer yang terdaftar, tanpa harus menunggu proses `resolve`.

Cara penggunaan:
1. Buat transaksi baru via `POST /transactions/new` pada Node A.
2. Salin `transaction_id` dari response.
3. Kirim request berikut ke Node A:

```text
POST /transactions/broadcast
Content-Type: application/json

{
    "transaction_id": "<transaction_id_yang_disalin>"
}
```

Node A akan mengirimkan transaksi tersebut ke semua peer yang terdaftar.
Response akan menunjukkan daftar peer yang berhasil menerima (`success`) dan yang gagal (`failed`).

Catatan: Endpoint ini hanya menyebarkan transaksi ke peer, bukan block. Setiap node
tetap harus melakukan mining sendiri untuk memasukkan transaksi ke dalam chain-nya.

**Screenshot:**
<img width="2094" height="1744" alt="06-transaction-broadcast" src="https://github.com/user-attachments/assets/3f0f3a41-e47e-42c4-a50c-0ccb8658759c" />

### 10.7 Daftar Peer Node / Registered Peer Nodes

Endpoint `GET /nodes` digunakan untuk melihat daftar peer yang saat ini tersimpan di node lokal. Endpoint ini berguna untuk memverifikasi hasil `POST /nodes/register` atau memastikan parameter `--peers` saat startup sudah terbaca dengan benar oleh aplikasi.

Contoh request:

```bash
curl http://127.0.0.1:5001/nodes
```

Contoh response:

```json
{
    "node_url": "http://127.0.0.1:5001",
    "nodes": [
        "http://127.0.0.1:5002",
        "http://127.0.0.1:5003"
    ],
    "total": 2
}
```

Penjelasan field:

- `node_url` menunjukkan alamat node lokal yang sedang memberikan response.
- `nodes` berisi daftar peer yang sudah terdaftar pada node tersebut.
- `total` menunjukkan jumlah peer yang tersimpan di daftar lokal.

Catatan penting: Endpoint ini hanya menampilkan peer yang terdaftar secara lokal. `GET /nodes` tidak memeriksa apakah peer sedang aktif, tidak melakukan ping ke peer, dan tidak menjamin bahwa request broadcast atau resolve ke peer tersebut pasti berhasil.

**Screenshot:**
<img width="2094" height="1744" alt="07-get-peer-nodes" src="https://github.com/user-attachments/assets/cfe5497e-d0ff-4515-8eca-6558b9aaee10" />

## Implementasi Kriptografi / Cryptography Implementation

Implementasi kriptografi proyek ini berpusat pada tiga elemen: canonical JSON, hashing SHA-256, dan digital signature ECDSA `secp256k1`.

### Canonical JSON dan Hashing

Payload transaksi dan block diubah ke format JSON agar hasil hash dan signature selalu konsisten.

```python
def canonical_json(data):
    return json.dumps(data, sort_keys=True, separators=(",", ":"))


def sha256_hex(text):
    return hashlib.sha256(text.encode()).hexdigest()
```

Hash block dihitung dari gabungan `index`, `timestamp`, daftar transaksi, `nonce`, dan `previous_hash`. Transaction ID juga dibentuk dari hash SHA-256 atas payload transaksi lengkap.

### Pembentukan Address / Address Derivation

Address wallet bukan hasil skema address blockchain publik seperti Bitcoin atau Ethereum, melainkan diturunkan dari public key:

```python
def address_from_public_key(public_key_pem):
    return sha256_hex(public_key_pem.strip())[:40]
```

Pendekatan ini memudahkan pembuktian bahwa address pengirim benar-benar berhubungan dengan public key yang dipakai untuk verifikasi signature.

### Digital Signature ECDSA

Wallet menggunakan kurva `SECP256K1` untuk membuat key pair dan menandatangani payload transaksi:

```python
self.private_key = ec.generate_private_key(ec.SECP256K1())

def sign_payload(self, payload_dict):
    signature = self.private_key.sign(
        canonical_json(payload_dict).encode(),
        ec.ECDSA(hashes.SHA256()),
    )
    return signature.hex()
```

Verifikasi dilakukan dengan memuat public key PEM, lalu memeriksa signature terhadap canonical JSON dari payload transaksi:

```python
public_key = serialization.load_pem_public_key(self.public_key.encode())
public_key.verify(
    bytes.fromhex(self.signature),
    canonical_json(self.payload_dict()).encode(),
    ec.ECDSA(hashes.SHA256()),
)
```

Selain memverifikasi signature, kode juga memastikan `sender_address` identik dengan address yang dihitung dari public key. Ini mencegah pengirim mengklaim address yang tidak sesuai dengan key yang dia lampirkan.

## Endpoint API Documentation

Dokumentasi API dalam format Postman collection dapat diunduhnmelalui tautan berikut:

[Download Postman Collection](./Blockchain%20Technology.postman_collection.json)

| Method | Path | Deskripsi | Request Body | Response Utama |
|--------|------|-----------|--------------|----------------|
| `GET` | `/` | Menampilkan dashboard HTML node | Tidak ada | Halaman web dashboard |
| `GET` | `/dashboard-data` | Ringkasan node untuk dashboard | Tidak ada | Wallet, peers, chain, pending transactions, status validasi |
| `GET` | `/chain` | Mengambil blockchain lokal | Tidak ada | `node_url`, `length`, `difficulty`, `mining_reward`, `valid`, `chain` |
| `GET` | `/wallet` | Mengambil ringkasan wallet node | Tidak ada | `address`, `public_key`, `balance`, `spendable_balance`, `pending_transactions` |
| `GET` | `/transactions/pending` | Mengambil daftar transaksi pending | Tidak ada | `count`, `pending_transactions` |
| `POST` | `/transactions/validate` | Memvalidasi digital signature transaksi | Payload transaksi lengkap | `valid`, `message`, `transaction` |
| `POST` | `/transactions/new` | Membuat transaksi baru atau menerima transaksi yang sudah signed | Minimal `recipient_address` dan `amount`, atau payload signed lengkap | `message`, `transaction`, `signature_valid`, `pending_count` |
| `POST` | `/transactions/broadcast` | Menyebarkan transaksi dari pending pool lokal ke semua peer node yang terdaftar | `{"transaction_id": "<id>"}` | `message`, `transaction_id`, `success` (list peer berhasil), `failed` (list peer gagal) |
| `POST` | `/mine` | Menambang block dari pending transactions dan menambahkan reward miner | Umumnya `{}` | `message`, `block`, `miner_wallet`, `chain_length` |
| `POST` | `/nodes/register` | Menambahkan peer node ke daftar lokal | `{"nodes": ["http://127.0.0.1:5002", "..."]}` atau string dipisah koma | `registered_now`, `total_nodes` |
| `GET` | `/nodes` | Mengambil daftar peer node yang terdaftar di node lokal | Tidak ada | `node_url`, `total`, `nodes` |
| `GET` | `/nodes/resolve` | Membandingkan chain lokal dengan peer dan mengganti jika ada chain valid yang lebih panjang | Tidak ada | `replaced`, `message`, `source`, `chain_length`, `chain` |

Catatan tambahan:

- `POST /transactions/new` akan gagal jika sender tidak memiliki `spendable_balance` yang cukup.
- `POST /mine` akan gagal bila ada transaksi pending yang tidak valid.
- `GET /nodes/resolve` tidak akan mengubah chain bila chain lokal sudah merupakan chain valid terpanjang.

## Arsitektur P2P / P2P Network Architecture

Walaupun proyek ini menggunakan istilah P2P, implementasinya adalah jaringan node sederhana berbasis HTTP. Node tidak menggunakan socket P2P, tidak memiliki discovery protocol, dan tidak melakukan broadcast transaksi maupun block secara otomatis.

Skema koneksi multi-node:

```text
                 GET /chain saat resolve
+---------------------------+        +---------------------------+
| Node 5001                 |<------>| Node 5002                 |
| Wallet + Blockchain       |        | Wallet + Blockchain       |
| Peers: 5002, 5003         |        | Peers: 5001, 5003         |
+-------------+-------------+        +-------------+-------------+
              ^                                    ^
              |                                    |
              |                                    |
              v                                    v
        +---------------------------+
        | Node 5003                 |
        | Wallet + Blockchain       |
        | Peers: 5001, 5002         |
        +---------------------------+
```

Mekanisme komunikasi antar-node:

1. Peer didaftarkan secara manual melalui `--peers` atau `POST /nodes/register`.
2. Node yang tertinggal memanggil `GET /nodes/resolve`.
3. Fungsi `resolve_chain()` mengunjungi setiap peer dan mengambil data dari `GET /chain`.
4. Chain kandidat dideserialisasi dan divalidasi dengan aturan yang sama seperti chain lokal.
5. Jika ada chain yang lebih panjang dan valid, `replace_chain()` mengganti chain lokal dan menghapus transaksi pending yang sudah terkonfirmasi.

Implikasi arsitektur ini:

- Pending transaction tidak direplikasi otomatis ke node lain.
- Block baru tidak dikirim otomatis ke peer.
- Balance wallet antar-node bisa berbeda sementara waktu sampai proses `resolve` dilakukan.
- Consensus yang dipakai murni `longest valid chain`, tanpa voting dan tanpa difficulty adjustment dinamis.

## Penafian / Disclaimer

Proyek ini adalah tugas. Implementasi difokuskan untuk kebutuhan pembelajaran konsep dasar blockchain.
