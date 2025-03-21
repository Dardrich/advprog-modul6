## Commit 1 Reflection Notes

What is inside the `handle_connection` method?

```rust
fn handle_connection(mut stream: TcpStream) {
    let buf_reader = BufReader::new(&mut stream);
    let http_request: Vec<_> = buf_reader
        .lines()
        .map(|result| result.unwrap())
        .take_while(|line| !line.is_empty())
        .collect();

    println!("Request: {:#?}", http_request);
}
```

1) Membaca data dari koneksi TCP
Method menggunakan `BufReader` untuk membaca data dari `TcpStream` secara efisien secara baris perbaris
2) Mengumpulkan header HTTP
Method membaca setiap baris request HTTP hingga menemukan baris kosong (sebagai penanda akhir dari header). Baris-baris ini disimpan dalam sebuah vektor (`Vec<String>`)
3) Mencetak request ke console
Method mencetak seluruh header HTTP yang telah dikumpulkan ke console dalam format yang mudah dibaca (`{:#?}` untuk pretty-print)

## Commit 2 Reflection Notes
![Commit 2 screen capture](capture-2-mod6.jpg)

What is inside the new `handle_connection` method?
1) Membaca File HTML
Kode sekarang membaca isi file `hello.html` menggunakan `fs::read_to_string("hello.html").unwrap()`. File ini akan dikirim sebagai bagian dari respons HTTP.
2) Membuat Respons HTTP
Kode membuat respons HTTP lengkap dengan status line (HTTP/1.1 200 OK), header Content-Length, dan konten dari file `hello.html`. Format respons sesuai dengan protokol HTTP.
3) Mengirim Respons ke Client
Kode mengirim respons yang telah dibuat ke client menggunakan `stream.write_all(response.as_bytes()).unwrap()`. Ini mengirimkan data melalui koneksi TCP yang sama.

## Commit 3 Reflection Notes
Intinya refactoring yang ada memisahkan logika pemrograman menjadi dua hal:
- `handle_connection` untuk menangani koneksi TCP dan membaca request
- `generate_resp` untuk membuat respons HTTP berdasarkan request

Hal ini dilakukan untuk mencapai modularitas (karena logika respons dipisah, kode yang ada lebih mudah dikembangkan dan diuji), fleksibilitas (kini, server mampu menangani berbagai request, misalnya `200 OK` untuk `/` dan `400 NOT FOUND` untuk route lain), dan maintainability (karena method sekarang lebih kecil dan focused, perbaikan dan penambahan fitur mudah untuk dilakukan)

![Commit 3.1 screen capture](capture-3-mod6-1.jpg)
![Commit 3 screen capture](capture-3-mod6.jpg)

## Commit 4 Reflection Notes
```rust
let (status_line, filename) = match &request_line[..] { 
    "GET / HTTP/1.1" => ("HTTP/1.1 200 OK", "hello.html"), 
    "GET /sleep HTTP/1.1" => { 
        thread::sleep(Duration::from_secs(10)); 
        ("HTTP/1.1 200 OK", "hello.html") 
    } 
    _ => ("HTTP/1.1 404 NOT FOUND", "404.html"), 
}; 
```
Intinya kode ini memiliki tujuan untuk menambahkan durasi sebanyak 10 detik bagi program yang ada untuk berjalan kembali. Program ini bersifat single-threaded dan blocking. Jadi, selama waktu 10 detik tersebut, thread utama tertahan dan tidak bisa merespons request lain, bahkan jika request tersebut sederhana. Hal ini membuat semua request harus menunggu giliran untuk diproses.

## Commit 5 Reflection Notes

Masalah sebelumnya yang dihadapi adalah program yang di-desain secara single-threaded yang menyebabkan program dipaksa berjalan sekuensial dan saling tunggu-menunggu. Pada modifikasi kali ini, telah digunakan ThreadPool dan Worker. ThreadPool membuat sejumlah thread (di kasus ini, empat) yang siap menangani request, lalu ThreadPool mengirim request ke worker yang tersedia melalui channel. Worker akan menjalankan tugas secara bersamaan tanpa membuat thread baru setiap kali ada request.
Mekanisme kerja:
1) ThreadPool dibuat:
Saat server dimulai, ThreadPool dibuat dengan sejumlah worker (misalnya, 4 worker). Setiap worker siap menerima tugas
2) Request masuk:
Ketika client mengirim request, server menerima koneksi (`TcpStream`). ThreadPool mengirim tugas (`handle_connection`) ke channel.
3) Worker menangani request:
Salah satu worker menerima tugas dari channel. Worker menjalankan `handle_connection`, yang membaca request, memprosesnya, dan mengirim respons.
4) Concurrent handling:
Jika ada banyak request, ThreadPool akan mendistribusikan tugas ke worker yang tersedia. Jika semua worker sibuk, tugas akan antri di channel sampai ada worker yang selesai.