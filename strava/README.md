# Scraping Data menggunakan GoColly

Dokumentasi ini menjelaskan implementasi scraping data aktivitas Strava menggunakan GoColly dalam proyek ini. Tujuannya adalah untuk mengambil informasi aktivitas pengguna dari Strava dengan cara user mengirimkan link aktivitas strava ke bot domyikado dan menyimpannya ke mongodb.

## Install Dependensi

```bash
go get github.com/gocolly/colly/v2
```

## Fungsi Code

```go
c := colly.NewCollector(
    colly.AllowedDomains(domApp, domWeb),
)

```

Code diatas digunakan untuk membuat objek collector dan menambahkan collector options salah satunya `colly.AllowedDomains` untuk membatasi akses domain hanya ke domain yang di izinkan.

```go
c.OnHTML("a", func(e *colly.HTMLElement) {
	link := e.Attr("href")
    ...
})
```

Code diatas setiap kali melakukan scraping dan menemukan tag `<a>` akan mengekstrak `href` dan menyimpan dalam variable link.

```go
c.OnHTML("main", func(e *colly.HTMLElement) {
	stravaActivity.Name = e.ChildText("h3[class^='styles_name__']")
	stravaActivity.Title = e.ChildText("h1[class^='styles_name__']")
	stravaActivity.TypeSport = e.ChildText("span[class^='styles_typeText__']")

	e.ForEach("time[class^='styles_date__']", func(_ int, timeEl *colly.HTMLElement) {
		dt := timeEl.Attr("datetime")
		if dt != "" {
			stravaActivity.DateTime = formatDateTimeToIndo(dt)
		} else {
			stravaActivity.DateTime = dt
		}
	})
})
```

Code diatas setiap kali menemukan tag `<main>` dan didalamnya memiliki tag, contoh `h3[class^='styles_name__']`, maka akan mengekstrak teks yang ada pada tag tersebut dengan menggunakan `e.ChildText()`. `e.ForEach("time[class^='styles_date__']")` akan menelusuri semua tag `<time>` dengan class yang di tentukan yang dimana tag tersebut berada didalam `<main>`, kemudian mengambil attribute `datetime` yang terkandung didalam tag `<time>`.

```go
c.OnScraped(func(r *colly.Response) {
    ...
})
```

Event handler yang akan di panggil ketika proses scraping telah selesai di lakukan yaitu ketika `c.OnHTML()` sudah dipanggil semuanya. Didalam event handler ini bisa diisi dengan banyak hal, seperti validasi data, penyimpanan data ke database, logging.

```go
err := c.Visit(link)
if err != nil {
    fmt.Println("Error saat mengunjungi halaman:", err)
}
```

Digunakan untuk memulai proses scraping, event ini yang memicu berjalannya event `c.OnHTML`, `c.OnScraped`, dan lainnya. event `c.Visit()` biasanya selalu berada di paling bawah code.

## Struktur Fungsi

1. ### `StravaActivityHandler`, `StravaActivityUpdateIfEmptyDataHandler`, `StravaIdentityHandler`, `StravaIdentityUpdateHandler`

    - Fungsi utama yang menangani input dari pengguna.
    - Mengekstrak link Strava dari pesan pengguna.
    - Cek link menggunakan domain web atau app.
    - memanggil fungsi scraping data.
    - menampilkan response ke bot.

2. ### `scrapeStravaActivity`, `scrapeStravaActivityUpdate`, `scrapeStravaIdentity`

    - Mengambil data dari link strava yang di kirim ke bot dengan scraping data menggunakan library GoColly.
    - Memeriksa data apakah sudah ada di database mongodb.
    - Melakukan validasi untuk data yang akan disimpan ke database.
    - Menandai data sebagai Valid, Invalid, atau Fraudulent ketika menyimpan data ke database.
    - Ketika data yang disimpan Valid, maka akan melakukan POST data ke database web domyid untuk menyimpan data pengguna ke collection user dan poinstrava.

3. ### Helper Functions

    - `extractStravaLink` → Mengekstrak URL dari pesan pengguna.
    - `extractContains` → Mengecek link url strava dari `extractStravaLink` apakah menggunakan domain web atau app. Kemudian mengambil full url strava dan id activity/athelete dari url tersebut.
    - `formatDateTimeToIndo` → Mengonversi format waktu Strava (UTC) ke format Indonesia (WIB).
    - `parseDistance` → Mengonversi jarak dari string ke float.

4. ### Service Functions
    - `getConfigByPhone` → Mengambil data API yang tersimpan di database.
    - `postToDomyikado` → Mengirimkan data ke API yang diambil dari `getConfigByPhone`.

## Alur Scraping

1. User mengirim pesan berisi link aktivitas Strava.
2. Sistem mengecek validitas link dengan fungsi extractStravaLink.
3. Jika link valid:
    - Sistem mengambil ID aktivitas Strava.
    - Data aktivitas (nama, jenis olahraga, jarak, waktu tempuh, dll.) di-scrape menggunakan Colly.
    - Data diperiksa di database MongoDB.
4. Jika data belum ada, data akan disimpan dengan status Valid/Invalid/Fraudulent.
5. Jika data valid, informasi dikirim kembali ke user.
6. Jika ternyata user mengirimkan lagi link yang sama, informasi data yang sudah ada akan dikirimkan lagi ke user tanpa menyimpan data.

## Contoh Response

1. ### Jika Aktivitas Valid

    ```
    Haiiii kak, *John Doe*! Berikut Progres Aktivitas kamu hari ini yaaa!! 😀

    - Activity ID: 123456789
    - Name: John Doe
    - Title: Morning Run
    - Date Time: 24 Maret 2025
    - Type Sport: Run
    - Distance: 5.2 KM
    - Moving Time: 30 Menit
    - Elevation: 10 Meter
    - Status: Valid

    Semangat terus, jangan lupa jaga kesehatan dan tetap semangat!! 💪🏻💪🏻💪🏻
    ```

2. ### Jika Aktivitas Invalid

    ```
    Maaf kak, sistem hanya dapat mengambil data aktivitas jalan dan lari. Silakan share link aktivitas jalan dan lari Strava kamu.
    ```

    dan/atau

    ```
    Wahhh, kamu malas sekali ya, jangan malas lari terus dong kak! 😏
    Satu hari minimal 3 km, masa kamu cuma 2.5 km aja 😂
    xixixixiixi

    Jangan lupa jaga kesehatan dan tetap semangat!! 💪🏻💪🏻💪🏻
    ```

3. ### Jika Data Terindikasi Curang

    ```
    Jangan Curang donggg! Silahkan share record aktivitas yang benar dari Strava ya kak, bukan dibikin manual kaya gitu.
    Yang semangat dong... yang semangat dong...
    ```

4. ### Jika Data Terdeteksi Sudah Ada

    - #### Jika Status Valid

        ```
        Maaf kak, *John Doe*! Kamu sudah pernah share aktivitas ini sebelumnya pada tanggal 18 Mar 2025 09:56 WIB! Berikut data aktivitas kamu yang sudah tersimpan.

        - Activity ID: 123456789
        - Name: John Doe
        - Title: Morning Run
        - Date Time: 24 Maret 2025
        - Type Sport: Run
        - Distance: 5.2 KM
        - Moving Time: 30 Menit
        - Elevation: 10 Meter
        - Status: Valid

        Jangan Curang yaaaa!
        Sana Lari lagi jangan malas!
        Semangat terus, jangan lupa jaga kesehatan dan tetap semangat!! 💪🏻💪🏻💪🏻
        ```

    - #### Jika Status Invalid

        ```
        Maaf kak, *John Doe*! Aktivitas yang kamu kirimkan tidak valid. Berikut datanya.

        - Type Sport: Run
        - Distance: 2.2 KM
        - Moving Time: 30 Menit
        - Elevation: 10 Meter
        - Status: Invalid

        Sana Lari lagi jangan malas!
        Semangat terus, jangan lupa jaga kesehatan dan tetap semangat!! 💪🏻💪🏻💪🏻
        ```

    - #### Jika Status Fraudulent

        ```
        Jangan Curang donggg! Silahkan share record aktivitas yang benar dari Strava ya kak, bukan dibikin manual kaya gitu.
        Yang semangat dong... yang semangat dong...
        ```

## Kesimpulan

Dokumentasi ini memberikan gambaran bagaimana sistem scraping aktivitas Strava bekerja, mulai dari cara mengolah input, proses scraping, hingga validasi data sebelum disimpan ke database. Pastikan untuk menguji setiap skenario agar sistem dapat bekerja secara optimal.
