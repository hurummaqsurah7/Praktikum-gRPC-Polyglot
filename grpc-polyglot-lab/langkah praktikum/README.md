# gRPC Polyglot

Praktikum komunikasi inter-service dengan gRPC. Satu server gRPC ditulis dengan **Go**, lalu diakses oleh dua client yang ditulis dengan **Python** dan **Node.js**. Ketiganya memakai satu berkas kontrak `user.proto` (pendekatan *Schema-First Development*).

- Mata kuliah: Paradigma Sistem untu IT
- Politeknik Negeri Lhokseumawe, Program Studi Teknologi Rekayasa Komputer Jaringan
- Nama / NIM / Kelas: Hurum Maqsurah / 2024903430046 / TRKJ 3C
- Tanggal praktikum: 15 Oktober 2026

## Tujuan

1. Memahami arsitektur *Schema-First Development* menggunakan Protocol Buffers.
2. Menginstalasi komponen compiler gRPC (`protoc`).
3. Membuat gRPC Server menggunakan Go.
4. Membuat gRPC Client menggunakan Python dan Node.js untuk berkomunikasi dengan server Go yang sama.

## Struktur Project

```
grpc-polyglot-lab/
├── .gitignore
├── README.md
├── user.proto
├── go-server/
│   ├── go.mod
│   ├── go.sum
│   ├── main.go
│   └── pb/
│       ├── user.pb.go
│       └── user_grpc.pb.go
├── python-client/
│   ├── client.py
│   ├── user_pb2.py
│   └── user_pb2_grpc.py
└── node-client/
    ├── client.js
    ├── package.json
    └── package-lock.json
```

## Prasyarat

Pastikan perangkat lunak berikut sudah terpasang, lalu cek dengan perintah verifikasinya:

| Perangkat lunak | Perintah verifikasi |
|---|---|
| Git | `git --version` |
| Python 3.10+ | `python --version` |
| Node.js dan npm | `node -v` dan `npm -v` |
| Go | `go version` |
| Protocol Buffers Compiler | `protoc --version` |

## Langkah-langkah Praktikum

### 1. Setup folder dan Git

```bash
mkdir grpc-polyglot-lab
cd grpc-polyglot-lab
git init
echo "node_modules/" >> .gitignore
echo "__pycache__/" >> .gitignore
echo ".venv/" >> .gitignore
echo "*.exe" >> .gitignore
git add .gitignore
git commit -m "Initial commit: Add gitignore rules"
```

### 2. Menulis kontrak API (`user.proto`)

Buat `user.proto` di folder utama. Berkas ini menjadi *Single Source of Truth* untuk ketiga komponen.

```proto
syntax = "proto3";

package user;

option go_package = "./pb";

message UserRequest {
  int32 user_id = 1;
}

message UserResponse {
  string name = 1;
  string email = 2;
}

service UserService {
  rpc GetUserProfile (UserRequest) returns (UserResponse);
}
```

### 3. gRPC Server (Go)

```bash
mkdir go-server
cd go-server
go mod init go-server
go get google.golang.org/grpc
go get google.golang.org/protobuf
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
protoc --proto_path=../ --go_out=. --go-grpc_out=. ../user.proto
```

Perintah `protoc` menghasilkan `pb/user.pb.go` dan `pb/user_grpc.pb.go`. Setelah itu buat `main.go`: server mendengarkan port `:50051`, mencatat log setiap request, dan mengembalikan data "Budi Santoso" untuk `user_id` bernilai 1.

### 4. gRPC Client (Python)

```bash
cd ../
mkdir python-client
cd python-client
pip install grpcio grpcio-tools
python -m grpc_tools.protoc --proto_path=../ --python_out=. --grpc_python_out=. ../user.proto
```

Buat `client.py` yang membuka kanal ke `localhost:50051`, memanggil `GetUserProfile` dengan `user_id=1`, lalu mencetak hasilnya.

### 5. gRPC Client (Node.js)

```bash
cd ../
mkdir node-client
cd node-client
npm init -y
npm install @grpc/grpc-js @grpc/proto-loader
```

Buat `client.js` yang membaca `../user.proto` saat program berjalan dengan `protoLoader.loadSync`, lalu memanggil `GetUserProfile`.

### 6. Pengujian

Buka tiga terminal:

```bash
# Terminal 1 (Server Go)
cd go-server
go run main.go

# Terminal 2 (Client Python)
cd python-client
python client.py

# Terminal 3 (Client Node.js)
cd node-client
node client.js
```

Hasil yang diharapkan:

- Terminal server mencetak `gRPC Server (Go) berjalan di port :50051...` dan satu baris log `Menerima request untuk User ID: 1` untuk setiap client yang dijalankan.
- Kedua client mencetak `Respon dari Server -> Nama: Budi Santoso, Email: budi@universitas.ac.id`.

### 7. Tugas evaluasi: menambah field `phone`

1. Tambahkan field baru pada `UserResponse` di `user.proto`:

   ```proto
   message UserResponse {
     string name = 1;
     string email = 2;
     string phone = 3;
   }
   ```

2. Generate ulang kode Go dan Python dengan perintah `protoc` yang sama seperti pada langkah 3 (dari folder `go-server`) dan langkah 4 (dari folder `python-client`). Node.js tidak perlu generate ulang.
3. Pada `main.go`, isi `Phone: "081234567890"` untuk User ID 1.
4. Pada `client.py` dan `client.js`, tambahkan `Phone: {response.phone}` pada baris cetak.
5. Jalankan ulang server dan kedua client. Keluaran client kini memuat `Phone: 081234567890`.

## Analisis Pendekatan Kompilasi

Pada pendekatan **static code generation** (Go dan Python), skema `.proto` diproses oleh `protoc` pada tahap pengembangan, sebelum program dijalankan. `protoc` dengan plugin `protoc-gen-go` dan `protoc-gen-go-grpc` menghasilkan `user.pb.go` dan `user_grpc.pb.go`, sedangkan `python -m grpc_tools.protoc` menghasilkan `user_pb2.py` dan `user_pb2_grpc.py`. Berkas-berkas ini menjadi bagian dari kode proyek dan diimpor oleh program. Akibatnya, setiap kali `user.proto` berubah, misalnya saat field `phone` ditambahkan, `protoc` harus dijalankan ulang agar kode hasil generate ikut diperbarui.

Pada pendekatan **dynamic runtime generation** (Node.js), tidak ada berkas kode hasil generate. `client.js` membaca `user.proto` langsung ketika program dijalankan melalui `@grpc/proto-loader`, lalu `@grpc/grpc-js` membentuk definisi layanan di memori. Karena skema dibaca saat program berjalan, perubahan pada `user.proto` tidak memerlukan perintah generate. Yang perlu diubah hanya kode yang memakai field baru.

Kedua pendekatan memberikan hasil yang sama pada praktikum ini: client menerima data yang sama dari server Go. Perbandingan kinerja tidak diuji.

## Push ke GitHub

```bash
cd ..
git add .
git commit -m "Docs: Add README analisis static vs dynamic"
git remote add origin https://github.com/hurummaqsurah7/Praktikum-gRPC-Polyglot.git
git push -u origin main
```

Jika repositori sudah terhubung (`git remote -v` menampilkan `origin`), lewati `git remote add` dan cukup jalankan `git push`.