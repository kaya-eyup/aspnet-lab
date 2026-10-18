# Gün 51 — Veriyi değiştirmek ve uygulamayı canlıya çıkarmak

**18 Ekim 2026 Pazar · Dersler 526–532 (Web API bölümünün sonu) + 511–519 (Hosting)**

## Günün konusu bir bakışta

Kurs bugün iki soruyu cevapladı:

1. **Bir API veriyi nasıl değiştirir?** Ekleme, güncelleme ve silme; dışarı giden verinin DTO ile şekillendirilmesi (526–531).
2. **API'yi başka uygulamalar nasıl kullanır, API internete nasıl çıkar?** HttpClient ve fetch ile çağırma, CORS; alan adı, barındırma, yayın çıktısı ve uzaktaki veritabanı (532, 511–519).

```
Kullanıcı ──▶ alan adı ──(DNS)──▶ sunucu ──▶ uygulama (publish çıktısı) ──▶ veritabanı
                                    ▲
           React / MVC / düz JS başka bir origin'den çağırır (CORS)
```

İkinci soru Gün 52–53'ün işi: capstone Salı günü canlıya çıkıyor.

---

## 1. Veriyi değiştirmek: ekleme, güncelleme, silme

### Hocanın kurduğu yapı

- Katmanlar: Controller → servis (`ProductManager`) → Unit of Work → repository → EF.
- Bütün işlemler async: `CreateAsync`, `UpdateAsync`, `DeleteAsync`. Değişiklikler `SaveAsync` ile yazılıyor.

| İşlem | Uç | Başarılı cevap | Diğer durumlar |
|---|---|---|---|
| Ekleme | `POST /api/products` | 201 + `Location: /api/Products/7` + kayıt | |
| Güncelleme | `PUT /api/products/{id}` | 204 `NoContent` | adresteki id ≠ gövdedeki id → 400 · kayıt yok → 404 |
| Silme | `DELETE /api/products/{id}` | 204 | kayıt yok → 404 |

- **201 ve Location:** `CreatedAtAction(nameof(GetProduct), new { id = … }, …)` yeni kaydın adresini `GetProduct` ucunun rotasından üretiyor. Bunun için tek kaydı gösteren bir uç gerekiyor.
- **204:** "İstediğin oldu, geri gönderecek bir şeyim yok." Güncelleme ve silmede yaygın.
- PUT ve DELETE kalıbı sektör standardı: önce kaydı bul, yoksa 404, varsa işle.

### Güncelleme change tracking'e dayanıyor

- `UpdateAsync`'te `Update()` çağrısı yok. `GetById`'ın getirdiği nesnenin alanları değiştiriliyor, ardından `SaveAsync` çağrılıyor.
- EF, Gün 50'de `Add`'de gördüğün defteri burada da tutuyor: Takip ettiği nesnede neyin değiştiğini fark ediyor ve sadece değişen sütunlar için `UPDATE` yazıyor.
- Şartı: Nesnenin takip edilerek getirilmiş olması. Projection yapan okuma uçları hiçbir şeyi takip etmez.
- Bu kalıp oku-değiştir-yaz. İki admin aynı ürünün formunu açıp sırayla kaydederse, ikincinin formundaki eski değerler birincinin değişikliğini ezer. Ürün düzenlemede bu çoğu zaman kabul edilir, sayaçta edilmez (Gün 60, kayıp oy deneyi).

### DTO iki yönlü bir kalkan

- Hocanın tanımı "ekrana gelmemesi gereken alanlar için" ve DTO'yu sadece çıkışta kullanıyor (`ProductToDTO`). Bu, DTO'nun yarısı.
- Öbür yarısı giriş yönü: istemcinin yazmaması gereken alanlar. Hocanın `CreateProduct(Product entity)` ucu girdiyi doğrudan entity'ye bağlıyor, yani toplu atama açık:
  - İstemci `"productId": 7` gönderirse EF bu değeri `INSERT`'e koyar. SQL Server'da sonuç 500; bazı veritabanlarında numarayı istemci seçmiş olur.
  - Entity'de onay gibi bir alan varsa istemci kendi kaydını onaylı yapabilir. Bu bir hata vermez; çalışan ama yanlış veri üretir.
- **Tutarsızlık:** `UpdateAsync` alanları tek tek elle kopyalıyor; güncellemede toplu atamaya karşı doğru yöntem bu. Ama `Url` ve `ImageUrl`'yi kopyalamıyor: PUT ile yeni url gönderen istemci 204 alır, url değişmez.

### Hocanın kodundaki diğer kusurlar

- **Async sürüm doğrulamayı kaybetmiş.** `Create` önce `Validation(entity)` çağırıyor, `CreateAsync` çağırmıyor. API async sürümü kullandığı için API'den gelen hiçbir ürün doğrulanmıyor. Ders: Aynı işin iki kopyası tutulmaz; biri güncellenir, öbürü unutulur.
- **Dönüşüm bellekte.** `GetById` bütün sütunları çekiyor, `ProductToDTO` bellekte çeviriyor.
- **`double? Price`.** `double` ondalık sayıyı yaklaşık tutar (`0.1 + 0.2` sorunu), kuruş hataları birikir. C#'ta para için `decimal`.
- **Mapper.** Hoca eşlemeyi ileride bir kütüphaneyle, muhtemelen AutoMapper ile otomatikleştirecek. AutoMapper 2025'ten beri 15 ve sonraki sürümlerde ticari lisanslı; ücretsiz kalan son sürümdeki bir güvenlik açığı düzeltilmedi. Türkiye'deki mevcut projelerde çok yaygın, tanıman yeterli.
- **Repository + Unit of Work.** Hoca EF'in üstüne bir katman daha koyuyor: tablo başına veri işlerini toplayan bir sınıf ve hepsini tek seferde kaydeden ortak bir `Save`. Türkiye'deki kurumsal projelerde yaygın, stajda göreceksin. Tartışmalı bir konu, çünkü `DbContext` zaten aynı işi yapıyor.

### Capstone'daki karşılığı

| Konu | Hoca | Capstone | Gerekçe |
|---|---|---|---|
| Giriş | `Product entity` | `CreateCommentRequest` (3 yuva) | Toplu atama yapısal olarak imkânsız |
| Çıkış | `ProductToDTO`, bellekte | `CommentDto`; okumada `Select` projection, SQL'de | Sadece gereken sütunlar gelir |
| Kayıt ekleme | `AddAsync` (repository içinde) | `Add` | `AddAsync` sadece özel numara üreticileri için |
| 201 | `CreatedAtAction` + Location | `StatusCode(201, …)`, Location yok | Tek yorumu gösteren uç yok |
| Katmanlar | Controller → Manager → UoW → Repository | Controller → Service → `AppDbContext` | `DbContext` zaten UoW, `DbSet` zaten repository |
| Eşleme | İleride mapper | Elle | DTO alanları `required`; unutulan ya da adı değişen alan derleme hatası verir |
| Güncelleme / silme | Var | Gün 61–62 (oy değiştirme, anket oyu); silme yok | |

---

## 2. API'yi başka uygulamalardan çağırmak

### MVC'den: HttpClient

- Hoca MVC tarafında `HttpClient` ile `GET /api/products` attı, gelen JSON'u `List<Product>`'a çevirip Razor sayfasına verdi.
- Bu kodda üç sorun var:
  - **`using (var httpClient = new HttpClient())`:** Her istekte bağlantı açıp kapatmak, yük altında işletim sisteminin bağlantı kaynaklarını tüketir. Kapanan bağlantılar bir süre bekleme durumunda kalır ve bir noktadan sonra yenisi açılamaz. Buna **socket exhaustion** deniyor. Sektörün yolu `IHttpClientFactory`: `builder.Services.AddHttpClient()` ile kaydedilir, DI'dan alınır.
  - **Durum kodu kontrol edilmiyor:** API 500 dönerse hata gövdesi ürün listesi diye okunmaya çalışılır.
  - **`JsonConvert`:** Newtonsoft.Json, 3.1 döneminin kütüphanesi. Bugün `System.Text.Json` kullanılıyor.
- Capstone'da karşılığı yok (frontend React), ama klasik bir mülakat sorusu.

### Düz JS'ten: fetch

- Hoca düz JS'te `fetch` ile ürünleri çekip `innerHTML` ile ekrana bastı.
- **XSS:** API'den gelen metin `innerHTML`'e yazılırsa, ürün adındaki `<img src=x onerror="…">` metin olarak değil kod olarak çalışır. Başkasının tarayıcısında senin sitenin adına kod çalıştırmaya **XSS** deniyor. Doğrusu `textContent`. React `{x}`'i kaçışlıyor; `dangerouslySetInnerHTML` bu korumayı kaldırır.
- Ekran görüntüsündeki arrow fonksiyonda `async` yok; `await` bu hâliyle SyntaxError verir.

### CORS ve origin

- **Origin:** şema + host + port. `https://example.com` ile `https://example.com:9000` farklı origin'ler; sadece port farkı bile yeter.
- **Same-origin policy:** Tarayıcının varsayılan kuralı. Bir sayfa, başka bir origin'den gelen cevabı okuyamaz.
- **CORS:** Sunucunun "şu origin'lere izin veriyorum" demesinin yolu. Yani aynı origin şartı değil, farklı origin'de çalışmanın çaresi. Notumdaki "aynı originde çalışmalı" cümlesi eksikti.
- **CORS sunucuyu korumaz.** Kuralı uygulayan tarayıcı. Postman, curl, `.http` dosyası ya da bir bot CORS'a hiç bakmaz. Korunan şey kullanıcının tarayıcısı: Kötü bir site, kullanıcının tarayıcısında açıkken senin API'nin cevabını okuyamaz.
- Hoca `AllowAnyOrigin()` kullandı. Sektörde izinli origin'ler tek tek yazılır (`WithOrigins`).
- **Preflight:** Farklı bir origin'e JSON gövdeli bir POST gitmeden önce tarayıcı sunucuya "bu origin'den bu isteği kabul ediyor musun?" diye sorar. Bu ön soru bir `OPTIONS` isteği ve adı **preflight**. Chrome Network'te ayrı bir satır olarak, sadece **All** filtresinde görünüyor; Fetch/XHR filtresinde görünmüyor.

### Capstone'daki karşılığı

- React (5173) ile API (7129) farklı origin'ler. Gün 45'ten beri `WithOrigins` + `Cors:AllowedOrigins` ayarı var.
- Gün 53'te listeye Vercel adresi eklenecek.
- Bugün React formu `POST /comments`'e yazdı (log: `{ input }` olayı).

---

## 3. Uygulamayı canlıya çıkarmak (hosting)

### Zincir

```
sadikturan.com ──(DNS)──▶ sunucunun IP adresi ──▶ sunucu, istekteki ada bakıp siteyi seçer ──▶ publish klasörü
```

- **Alan adı (domain):** Sitenin adı; bir firmadan yıllık kiralanıyor.
- **DNS:** Adı sunucunun adresine (IP) çeviren rehber. "Yönlendirme", alan adı firmasının panelinde bu rehbere kayıt girmek demek. Alan adı ve hosting aynı firmadaysa bu kaydı firma girer, farklı firmalardaysa sen girersin.
- Hocanın slaytında ok doğrudan klasöre gidiyor (`c:\sadikturan\index.html`). Kesin hâli şöyle: DNS isteği sunucuya kadar götürür; hangi klasörün açılacağına, gelen istekteki alan adına bakarak sunucu karar verir.

### Barındırma türleri (slayttaki kutular)

| Tür | Ne demek | Kim yönetiyor |
|---|---|---|
| Hosting (paylaşımlı) | Tek sunucuda yüzlerce site, panelden yönetilir | Firma; sen sadece dosyalarını koyarsın. Ucuz ve sınırlı |
| VDS / VPS | Sana ayrılmış sanal bir sunucu | İşletim sistemi dahil sen |
| Server | Bütün fiziksel makine senin | Sen |
| Azure / AWS | Bulut: kaynak kullandıkça ödenir | Seçtiğin hizmete göre. App Service'te sen kodu verirsin, sunucuyu Microsoft yönetir |

- Hocanın yolu: paylaşımlı hosting + **Plesk**, yani hosting firmasının verdiği web yönetim paneli (diğer yaygın panel cPanel).
- Hosting'in "ASP.NET sürümleriyle uyumlu" olması şu yüzden önemli: Varsayılan publish çıktısı .NET'i içinde taşımaz, sunucuda doğru .NET sürümünün kurulu olmasını bekler.

### Yayın çıktısı: `dotnet publish`

- Sunucuya kaynak kod değil, **derlenmiş çıktı** gider.
- `dotnet publish` (.NET 8'den beri varsayılan olarak Release yapılandırmasıyla) `bin/Release/net10.0/publish/` klasörünü üretir. İçinde projenin `.dll`'i, bağımlılıklar, `appsettings*.json`, `wwwroot` ve IIS'e uygulamayı nasıl başlatacağını söyleyen `web.config` var.
- Hoca klasörü zip'leyip panelden yükledi; bu elle yayın. Bir de **FTP** anlatıldı: dosyaları bilgisayarından sunucuya kopyalamanın eski, standart yolu. Düz FTP parolayı şifresiz gönderir; güvenli hâlleri SFTP ve FTPS.
- **Capstone:** Aynı adımları (derle → publish → sunucuya gönder) GitHub Actions her push'ta otomatik yapacak (Gün 52). Vercel'de frontend için yaşadığının aynısı.

### SSL

- `https`'teki s. Tarayıcı ile sunucu arasındaki trafiği şifreliyor; sertifika "bu sunucu gerçekten o alan adına ait" diyor.
- Zorunluluğu yasal değil, pratik: Tarayıcılar `http` siteleri "Güvenli değil" diye işaretliyor.
- **Capstone için kritik kural:** `https` bir sayfa `http` bir API'yi çağıramaz, tarayıcı isteği engeller. Buna **mixed content** deniyor. Vercel sayfası https olduğu için API da https olmak zorunda. Azure'un ücretsiz alt alan adı (`….azurewebsites.net`) sertifikasıyla birlikte geliyor; alan adı da sertifika da satın almıyoruz.

### Uzaktaki veritabanı

- Hoca Plesk'te `storeDb` adında bir SQL Server veritabanı ve bir veritabanı kullanıcısı oluşturdu, `appsettings.json`'daki bağlantı dizesini güncelledi. Ardından **SSMS** (SQL Server'a bağlanıp tablolara bakmaya ve sorgu çalıştırmaya yarayan Microsoft masaüstü aracı) üzerinden yapıyı oraya taşıdı; bu adımlar notlara girmedi.
- **Bağlantı dizesinin farkı:** Yerelde `Trusted_Connection=True` ile Windows kimliğinle bağlanıyorsun. Uzaktaki sunucuya kullanıcı adı ve parolayla bağlanılıyor: `User Id=…;Password=…`. O parola bir **sır**.
- **Hocanın yaptığı ve bizim yapmayacağımız:** Parolayı `appsettings.json`'a yazmak. O dosya depoya da girer, publish çıktısına da; bu derste videoya bile girdi.
- **Sızan bir sır silinerek kurtarılamaz,** çünkü git geçmişi onu saklar. Yapılacak tek şey parolayı değiştirmek; buna **rotate** deniyor.
- **Capstone:** Sırlar geliştirmede bilgisayarın gizli deposunda, canlıda App Service'in ayarlarında tutulacak (Gün 52'nin ilk konusu). LocalDB dizesinde parola olmadığı için o bir sır değil; asıl sır Azure SQL'in dizesi.

### Uygulama canlıda olduğunu nereden biliyor?

- `ASPNETCORE_ENVIRONMENT` ortam değişkeninden. Değer verilmemişse varsayılan **Production**.
- Production'da `appsettings.Development.json` okunmaz.
- **Capstone'a etkisi:** `ConnectionStrings:Default` ve `Cors:AllowedOrigins` ayarlarımız `appsettings.Development.json`'da. Azure'da ikisi de boş gelecek:
  - Bağlantı dizesi yoksa `Program.cs`'deki fail fast satırı uygulamayı açılışta durdurur. Bu bir hata değil, tasarım: Yanlış ayarla yarım çalışmak yerine hiç açılmıyor. Çözüm, bağlantı dizesini App Service ayarına yazmak.
  - CORS listesi boş olduğu için hiçbir origin'e izin verilmez. Vercel adresi Gün 53'te eklenecek.

### Hosting'in dört sorusu: hoca ve capstone

| Soru | Hoca | Capstone (Gün 52) |
|---|---|---|
| Sunucuya ne gidiyor? | `dotnet publish` çıktısı, zip + panel | Aynı çıktı, GitHub Actions ile otomatik |
| Bağlantı bilgisi nerede? | `appsettings.json`, parolasıyla | Geliştirmede bilgisayarın gizli deposu, canlıda App Service ayarı |
| Tablo yapısı uzaktaki veritabanına nasıl gidiyor? | SSMS üzerinden elle | Kendi bilgisayarından, canlı bağlantıyla `dotnet ef database update`; otomatikleştirmesi Blok H |
| Uygulama canlıda olduğunu nereden biliyor? | Notlara girmedi | `ASPNETCORE_ENVIRONMENT`; yoksa Production |

Ders listesinde e-posta hesabı da vardı; capstone'da karşılığı yok.

---

## Bu günden akılda kalacak beş şey

1. DTO iki yönlü: Çıkışta alan sızdırmaz, girişte alan kabul etmez.
2. Aynı işin iki kopyası (sync/async) zamanla ayrışır.
3. CORS bir izin mekanizması, güvenlik duvarı değil.
4. Sunucuya derlenmiş çıktı gider; sırlar o çıktıya girmez.
5. Canlıda ortam varsayılan olarak Production'dır; Development ayarları orada yoktur.
