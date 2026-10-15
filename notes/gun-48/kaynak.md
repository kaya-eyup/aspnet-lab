# Gün 48 — Kaynak: async sorgular ve projection

**Okunanlar:**
- EF Core: *Asynchronous Programming*
- *Efficient Querying*, sadece "Project only properties you need" bölümü

## Soru ve cevabım

**Soru:** *Why does the page recommend async database calls in a web application, if an async query does not finish any faster than a sync one?*

**Cevabım:** Async'in amacı tek bir isteği hızlandırmak değil, sunucunun aynı anda hizmet verebileceği toplam kullanıcı sayısını, yani iş hacmini artırmak.

**Değerlendirme:**
- ✅ Sonuç doğru.
- Eksik halka: async'in bekleme sırasında neyi serbest bıraktığı. Cevabı **thread**. Aşağıda.

## 1. Async neyi kurtarıyor?

**Lokanta örneği:** Garson siparişi mutfağa verir.
- Senkron garson yemek pişene kadar mutfak kapısında dikilir.
- Async garson o sırada başka masalara bakar, yemek çıkınca geri gelir.
- Yemek iki durumda da aynı sürede pişer. Değişen, aynı sayıda garsonla kaç masaya bakılabildiği.

**ASP.NET Core'da:**
- Her istek, sınırlı bir işçi havuzundan alınan bir thread üzerinde çalışır. Bu havuza **thread pool** deniyor.
- `ToList()` (senkron): SQL Server cevap verene kadar thread hiçbir şey yapmadan bekler.
- `await ToListAsync()`: Bekleme süresince thread havuza geri döner. Cevap gelince boştaki herhangi bir thread kaldığı yerden devam eder.
- Tek istek aynı sürede biter (hatta çok az daha yavaş). Kazanç, aynı thread sayısıyla daha çok eşzamanlı isteğe bakabilmek. Buna **scalability** deniyor.

**Yük altında senkron kodun ne yaptığı:**
- Bütün thread'ler veritabanını bekler.
- Yeni istekler kuyruğa girer.
- İşlemci boşta dururken site yavaşlar.

Buna **thread pool starvation** deniyor. Sayfanın "senkron ve async kodu karıştırma" uyarısı bundan. En kötü örnek, async bir metodun sonucunu `.Result` ya da `.Wait()` ile beklemek: hem async'in kazancı kaybolur hem thread kilitlenir.

**JS bağlantısı (Gün 13):**
- JS'te tek thread var. Event loop beklemeyi zaten bloklamadan yapıyor.
- .NET'te çok thread var ama sayıları sınırlı. Async aynı fikrin .NET hâli.
- `await` iki dilde de aynı anlama geliyor: "sonuç gelene kadar bu satırda bekle, ama işçiyi serbest bırak."

## 2. Sayfanın uyarıları

- **Sadece sorguyu çalıştıran operatörlerin async hâli var:** `ToListAsync`, `FirstOrDefaultAsync`, `AnyAsync`, `CountAsync`.
  - `Where`, `OrderBy` ve `Select` sorguyu sadece kurar, hiçbir şey çalıştırmaz. Bu yüzden async hâlleri yok.
  - Bunlar `Microsoft.EntityFrameworkCore` namespace'inde. `using` eksikse VS "`ToListAsync` bulunamadı" der; Ctrl+. ekler.
- **Aynı context üzerinde aynı anda iki işlem yapılamaz.**
  - `Task.WhenAll(_db.Items.ToListAsync(), _db.Comments.ToListAsync())` hata verir: *"A second operation was started on this context instance…"*
  - Context aynı anda tek iş yapar. Her isteğe ayrı context verilmesinin (scoped) bir sebebi de bu.
- **SqlClient'ın async okumasında bilinen bir yavaşlık var.**
  - Çok büyük metin veya ikili sütunlarda görülüyor (`nvarchar(max)`, `varbinary(max)`).
  - Capstone'da böyle sütun yok, en uzunu `Body` (500).
  - Sektörde varsayılan yine async; senkrona ancak ölçüm bir sorun gösterirse dönülür.
- **İptal sinyali:** EF iptal sinyalini veritabanı sağlayıcısına iletir. Bizdeki zincir şöyle:
  1. React'te `AbortController` isteği iptal eder.
  2. Tarayıcı bağlantıyı keser.
  3. ASP.NET bunu `CancellationToken` parametresine bildirir.
  4. `ToListAsync(cancellationToken)` çağrısı üzerinden SqlClient sorguyu yarıda keser.

## 3. Projection: sadece gerekeni iste

- Entity'yi bütün hâlinde yüklemek, tablonun bütün sütunlarını getirir.
- `Select` ile istenen sütunlar seçilirse SQL'de sadece onlar gelir:
  `Select(c => new CategoryDto(c.Slug, …))` → `SELECT [c].[Slug], [c].[Name], …`
- `i.Category.Slug` yazmak bir JOIN üretir, ama kategori satırının tamamını değil sadece slug'ı getirir.
- EF, projection sonucu olan DTO'ları değişiklik için izlemez, yani `AsNoTracking()` gereksiz. Entity'nin kendisini döndürseydik, salt okunan sorgularda `AsNoTracking()` sektör standardı olurdu.
- Sayfanın uyarısı: Projection okumak içindir. Güncelleme için entity gerekir, çünkü değişikliği izleyen odur. Gün 50'deki `POST /comments`'te entity oluşturacağız.

## 4. Sektör boyutu

- **Yeni .NET kodunda varsayılan async'tir, "async all the way":** Controller'dan veritabanına kadar zincirin her halkası async olur. Tek bir senkron halka bütün zincirin kazancını götürür.
- **Mülakat klasiği:** "Async bir isteği hızlandırır mı?" Cevap: hayır, ölçeklenebilirliği artırır.
- **Türkiye'deki eski projelerde senkron EF ve `ToList()` çok yaygın.** Hoca da 524. derse kadar senkron yazıyor. Gördüğünde bu yanlış değil, eski bir alışkanlık; yeni kodda async yazılır.
- **Async, I/O beklemesi içindir:** veritabanı, ağ, dosya. Hesaplama gibi işlemci işini `Task.Run` ile sarmanın web'de bir kazancı yok.
- **`DbSeeder` senkron ve bu doğru.** Web uygulamasının istek yolunda değil, `dotnet ef` aracında tek seferlik çalışıyor (Gün 47 kararı). "Karıştırma" uyarısı aynı uygulamanın istek yolu için geçerli.

## 5. Bugün capstone'da karşılığı

- [ ] Her action `async Task<ActionResult<…>>` ve `CancellationToken cancellationToken` alıyor.
- [ ] Sorgu `ToListAsync(cancellationToken)` / `FirstOrDefaultAsync(cancellationToken)` ile bitiyor.
- [ ] Sonuç `Select` ile doğrudan DTO'ya alınıyor; entity dışarı çıkmıyor.
- [ ] Konsoldaki SQL'de sadece DTO'nun istediği sütunlar var.
