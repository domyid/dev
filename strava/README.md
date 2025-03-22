# Scraping Data menggunakan GoColly

Dokumentasi ini menjelaskan implementasi scraping data aktivitas Strava menggunakan GoColly dalam proyek ini. Tujuannya adalah untuk mengambil informasi aktivitas pengguna dari Strava dan menyimpannya ke database MongoDB.

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
