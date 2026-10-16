# Gün 49 — Web API'ye giriş: web servisi, katmanlar ve `[ApiController]`

**16 Ekim 2026 · Dersler 436 + 520–525.** 436 MVC bölümünün son dersi. 520–525 Web API bölümünün başı; bu bölüm .NET Core 3.1 ile kaydedilmiş.

## Bu gün ne öğrendim

Kurs bugün MVC'den Web API'ye geçti. MVC'de sunucu HTML'i kendisi üretip tarayıcıya veriyordu. Web API'de sunucu sadece veriyi (JSON) veriyor, ekranı istemci çiziyor.

Hoca API'yi sıfırdan kurmadı. Mevcut e-ticaret projesinin **yanına** kurdu ve aynı iş katmanını (business) ve veri katmanını (data) hem MVC sitesine hem API'ye bağladı. Dersin asıl mesajı bu: **bir iş katmanı, çok istemci**.

Günün en önemli üç şeyi:

1. "Servis" kelimesinin üç ayrı anlamı var. Kurs, Microsoft dokümanı ve capstone bu kelimeyi farklı anlamlarda kullanıyor.
2. Katmanlı mimaride controller sadece HTTP konuşur. "Kayıt yok" bilgisini servis verir, 404 kararını controller verir. Hocanın kodu ve capstone bu kararda aynı.
3. Capstone'da gördüğüm otomatik 400 ve ProblemDetails'li 404 gibi davranışların sebebi `[ApiController]` attribute'u.

## 1. Web servisi neden var

Web sitesi, mobil uygulama, masaüstü uygulaması ve SPA aynı veriye ihtiyaç duyuyor. Her biri veritabanına doğrudan bağlansaydı iki sorun çıkardı:

- **Güvenlik:** Bağlantı bilgisi (sunucu adresi + parola) her uygulamanın içine gömülmek zorunda kalırdı. Mobil uygulamayı indiren herkes bu bilgiyi çıkarıp bütün veritabanına erişebilirdi. Araya bir HTTP servisi girince parola sunucuda kalır, istemci sadece izin verilen uçları görür.
- **Tekrar:** Her istemci iş kurallarını ayrı ayrı yazardı. Servisle kurallar tek yerde yazılır ve herkes aynı dili (JSON) konuşur.

**Başkasının servisini kullanmak (TMDB, kur API'leri vb.):**
- Bu durumda biz istemciyiz. Bu servisler belli bir ölçeğe kadar ücretsiz, sonrası ücretli.
- Kimin istek attığını bilmek için bir **API anahtarı** istiyorlar. TMDB dokümanındaki 401 sekmesi, anahtarsız gönderilen isteğin cevabı.
- Anahtar bir paroladır. React koduna gömülürse tarayıcıda herkes görür. Bu yüzden ücretli ya da kotalı bir servis kendi backend'imiz üzerinden çağrılır ve anahtar sunucuda kalır. Bu, "veritabanı parolası istemcide olmaz" kuralının aynısı.

**Bir API'nin dokümanı:** TMDB'nin sayfası her uç için olası durum kodlarını (200/401/404) ve örnek cevabı gösteriyor.
- Capstone'daki karşılığı README'deki "Uçlar" tablosu.
- Aynı bilginin makinenin okuyabildiği hâli de var: **OpenAPI** dokümanı. `Program.cs`'teki `AddOpenApi` + `MapOpenApi` satırları bunu Development'ta `/openapi/v1.json` adresinde üretiyor. Bu tür doküman sayfaları genellikle böyle bir dosyadan üretilir.

**Capstone'un yeri:** `turkiye-simulasyonu-api` diyagramdaki "Service" kutusu. Şimdilik tek istemcisi var: React, yani SPA kutusu.

## 2. "Servis" kelimesinin üç anlamı

| Anlam | Ne demek | Hocada | Capstone'da |
|---|---|---|---|
| **Web servisi** | HTTP üzerinden başka programlara veri sunan uygulamanın tamamı | `shopapp.webapi`; diyagramdaki "Service" kutusu | `turkiye-simulasyonu-api` deposunun tamamı |
| **Servis sınıfı** | Controller'ın arkasında iş kurallarını ve veri işini yapan sınıf | `IProductService` → `ProductManager`; diyagramdaki "Business" kutusu | `IItemService` → `ItemService` |
| **DI servisi** | DI'ın kurabildiği, kayıtlı her tip | `services.Add…` ile kaydedilen her şey | `builder.Services.Add…` satırları: `AppDbContext`, controller'lar, CORS, OpenAPI |

Hangisi kastediliyor: Cümlede istemci ya da HTTP geçiyorsa 1. anlam. "Controller'ın arkası" geçiyorsa 2. anlam. `builder.Services` ya da lifetime geçiyorsa 3. anlam (kaynak notu: `kaynak.md`).

## 3. Katmanlı mimari: istek nereden geçiyor

```
Hoca: 5 proje (entity · data · business · webapi · webui)
ProductsController ─▶ IProductService → ProductManager ─▶ IUnitOfWork → EfCore repository'ler ─▶ ShopContext ─▶ SQL Server
 webapi ("Service")     business ("Business")             data ("Data")

Capstone: tek proje, klasörler
ItemsController ─▶ IItemService → ItemService ─▶ AppDbContext ─▶ SQL Server
 Controllers/        Services/                    (DbSet = repository, SaveChanges = unit of work)
```

**Hocada neden 5 proje var:** MVC sitesi (`webui`) ve API (`webapi`) iki ayrı uygulama ve ikisi de aynı iş kurallarını kullanıyor. Paylaşılan kodun ikisinin de referans verebileceği ayrı bir projede durması gerekiyor. Yan kazancı da var: bağımlılığın yönünü derleyici denetliyor. Business projesi webapi'yi göremez, yani iş kuralları HTTP'ye bulaşamaz.

**Capstone neden tek proje:** Aynı iş kurallarını paylaşan ikinci bir uygulama yok. Klasörler yeterli. İkinci bir uygulama gelirse (admin paneli, gece çalışan bir arka plan işi) katmanlar ayrı projelere taşınır.

**Repository ve Unit of Work tartışması:**
- **Repository:** Veritabanını saklayan, sadece "getir, ekle, sil" yapan sınıf.
- **Unit of Work:** Birden çok değişikliği tek seferde kaydeden birim.
- EF Core'da `DbSet` zaten repository'nin, `DbContext.SaveChanges` zaten unit of work'ün işini yapıyor. Bu yüzden capstone'da bu katman yok.
- Üstüne bir kat daha eklenince iki yoldan biri seçilmek zorunda:
  - `IQueryable` gizlenir: dün gördüğüm projection gücü kaybolur, her sorgu şekli için yeni bir metot gerekir.
  - `IQueryable` dışarı açılır: soyutlamanın anlamı kalmaz.
- Savunanların gerekçesi: EF'i ileride başka bir araçla değiştirebilmek ve testte sahte bir repository takabilmek.
- Senior seviyesi, iki tarafı da gerekçesiyle savunabilmek.

**Türkiye'de yaygın kalıp:** `Abstract`/`Concrete` klasörleri, `Manager` eki, `IUnitOfWork` + generic repository (`IRepository<T>`). Stajda büyük ihtimalle bu iskeletle karşılaşacağım. Microsoft'un kendi örneklerinde bu adlar yok; bu yerel bir gelenek, yanlış değil. İş yerinde ekibin kuralına uyulur. Capstone'da `Service` eki kullanılıyor.

## 4. Controller'ın işi: sadece HTTP

Katmanlı yapıda controller üç iş yapar:
1. Adresi ve parametreleri alır.
2. Servisi çağırır.
3. Sonucu HTTP'nin diline çevirir: 200, 404, 400.

Servis HTTP bilmez; sadece veri ya da `null` döndürür. Bu sayede aynı servisi ileride bir arka plan işi de çağırabilir, testler de HTTP olmadan çağırabilir.

**Hocanın API controller'ı ile capstone aynı kararı veriyor:** Servis `null` dönerse controller `NotFound()`, değilse `Ok(...)` döner.

**MVC'de aynı durum (436):** Olmayan ürüne `RedirectToAction("List")` dönüyor. Kullanıcı liste sayfasına gidiyor ama durum kodu hiç 404 olmuyor (302 → 200). Arama motoru da istemci de ürünün olmadığını öğrenemiyor. Buna **soft 404** deniyor. MVC'de de doğrusu `NotFound()`.

**API'de bu karar ikiye bölünüyor:**
- API "yok" der: `ItemsController.GetById` → 404.
- Kullanıcıya ne gösterileceğine React karar verir: `endpoints.ts`'teki `fetchItem` 404'ü `null`'a çevirir, `ItemPage.tsx` `<NotFoundPage />` gösterir.

MVC'de bu iki karar aynı action'ın içinde.

**Hocanın controller'ı ile capstone'unki arasındaki farklar:**

| | Hoca | Capstone | Neden |
|---|---|---|---|
| Dönüş tipi | `IActionResult` | `ActionResult<ItemDto>` | Tip imzada yazılı olduğu için derleyici ve OpenAPI dönüş şeklini görür |
| Servis alanı | `private IProductService _productService;` | `private readonly …` | Constructor dışında yeniden atanamaz |
| `CancellationToken` | Yok | Her action'da var, servise ve EF'e kadar iniyor | Kullanıcı sayfadan ayrılınca sorgu da kesilir |
| Dışarı giden | Entity (hoca ileride düzeltecek) | DTO | Sözleşme veritabanının şekline bağlı kalmaz |
| Lifetime | `AddScoped` | `AddScoped` | İkisinde aynı: `DbContext` scoped, servis ondan uzun yaşayamaz |

## 5. `[ApiController]`: capstone'daki davranışların sebebi

`ControllerBase`, `Controller`'ın view desteği olmayan hâli. API, HTML değil JSON döndürdüğü için `ControllerBase` kullanır. Üstüne konan `[ApiController]` dört davranışı açar:

| Davranış | Capstone'daki kanıtı |
|---|---|
| **Attribute routing zorunlu:** `[Route]` olmadan action'a ulaşılamaz | `[Route("items")]` |
| **Parametrenin nereden geleceğini tahmin eder (binding source inference):** Parametrenin adı route'ta geçiyorsa route'tan, geçmiyorsa query'den okur. Parametre bir sınıf tipindeyse gövdeden okur | `GetById(string id)` route'tan · `GetAll(string? categoryId)` query'den · Gün 50'de `POST /comments` gövdeden |
| **Otomatik 400:** Parametre bağlanamazsa ya da doğrulanamazsa action hiç çalışmaz, `errors` alanlı bir 400 döner | `.http`: `itemId` yok ya da boş → 400, `errors.itemId`. Nullable açık olduğu için `?` işareti olmayan `string` zorunlu sayılıyor |
| **400 ve üstü hatalara standart bir gövde (ProblemDetails) koyar** | README: olmayan id → 404 (ProblemDetails). 405'i ise controller değil routing üretiyor, bu yüzden gövdesi boş (Gün 57) |

**`[Route("api/[controller]")]` ve düz adres:**
- `[controller]` URL'yi sınıfın adından üretiyor: `ProductsController` → `products`. Sınıfın adı değişirse URL de sessizce değişir ve istemciler kırılır.
- Capstone'da adres sözleşmenin parçası (`endpoints.ts`), bu yüzden düz yazılıyor.
- `api/` öneki, API ile MVC sayfaları aynı alan adında çalıştığında ikisini ayırmak için kullanılıyor (`shopapp.com/api/products` ↔ `shopapp.com/products`). Capstone'un API'si ayrı bir adreste çalışacak.

**Hocanın ilk demo controller'ı:** `return Products[id];`
- `GET /api/products/10` → `IndexOutOfRangeException` → 500 döner, 404 değil. Sınırı kontrol etmek kodun işi.
- `GET /api/products/abc` → `"abc"` sayıya çevrilemediği için otomatik 400 döner.

## 6. Sorgu tuzakları (436)

Ders: Ürün detayında aynı kategoriden, aktif, ürünün kendisi hariç (`i.Id != id`) 4 benzer ürün (`Take(4)`).

- **Sırasız `Take(4)`:** `OrderBy` yok, yani hangi 4 ürünün geleceği garanti değil. SQL'de `ORDER BY` olmadan sıra tanımsız. EF bu durum için bir uyarı tanımlıyor (`RowLimitingOperationWithoutOrderByWarning`). Capstone'da `Take(4)` Gün 56'da, Haftanın Seçilmişleri'nde `OrderByDescending`'den sonra geliyor.
- **`Find(id)`:** Sadece birincil anahtarla arar. Önce EF'in hafızasına (change tracker) bakar ve tam entity döndürür. Capstone'da arama slug'la yapılıyor ve sonuç DTO'ya projection'la alınıyor, bu yüzden `Where` + `Select` + `FirstOrDefaultAsync` kullanılıyor.
- **`ViewData["BenzerUrunler"]`:** Anahtarı metin olan bir sözlük. Anahtar yanlış yazılırsa hatayı derleyici değil çalışma anı yakalar (değer `null` gelir), Razor'da da tip dönüşümü gerekir. Sektörde bunun yerine sayfanın ihtiyaç duyduğu her şeyi taşıyan tipli bir sınıf kullanılır; buna **ViewModel** deniyor. API'deki karşılığı DTO.

## 7. Hocanın iş arayüzü = capstone'un önümüzdeki iki haftası

`IProductService : IValidator<Product>` metotlarının capstone'daki karşılıkları:

| Hoca | Capstone |
|---|---|
| `GetById` | `ItemService.GetByIdAsync` (Gün 49) |
| `GetProductsByCategory(name, page, pageSize)` + `GetCountByCategory` | Gün 54 sayfalama: liste + toplam = iki ayrı sorgu |
| `GetSearchResult(searchString)` | Gün 54 `?q=` |
| `GetHomePageProducts()` (`Anasayfa` bayrağı) | Gün 56 Seçilmişler: bayrakta kararı insan verir, hesapta veri |
| `IValidator<Product>` + `bool Create` | Gün 55 doğrulama. `bool` işlemin başarısız olduğunu söyler ama nedenini söylemez. API'de neden, 400 cevabının `errors` alanında söylenir |

Hoca arayüzde senkron imzalar ve entity kullanıyor. 524'te bir kısmını async'e çevirdi, `CancellationToken` kullanmadı.

## 8. Eski kayıtları okumak: .NET Core 3.1 → .NET 10

| Hoca (3.1) | Capstone (.NET 10) |
|---|---|
| `Startup.ConfigureServices(IServiceCollection services)` | `Program.cs`: `builder.Build()`'den önceki `builder.Services.…` satırları |
| `Startup.Configure(IApplicationBuilder app, …)` | `Program.cs`: `var app = builder.Build();`'den sonraki `app.Use…` / `app.Map…` satırları |
| `Configuration.GetConnectionString(…)` | `builder.Configuration.GetConnectionString("Default") ?? throw …` (değer eksikse uygulama açılışta durur) |
| `dotnet new` ile şablon listesi | `dotnet new list` |
| `dotnet new webapi` → controller'lı proje | .NET 8'den beri bu komut **minimal API** üretir |

- **Minimal API:** Uçlar sınıf ve metot yerine `Program.cs`'te doğrudan `app.MapGet("/items", …)` ile yazılıyor. Controller'lı proje için `dotnet new webapi --use-controllers` gerekiyor; VS'deki karşılığı "Use controllers" kutusu.
- Türkiye'de eski kurumsal projelerin çoğunda `Startup.cs` var. Stajda görürsem bu tabloyla çeviririm.

## 9. Araçlar: Postman ve `.http`

- Postman koleksiyonu Postman hesabında yaşar; istenirse dışa aktarılabilir. Türkiye'de ekiplerde çok yaygın.
- `.http` dosyası depoda durur ve kodla birlikte commit'lenir; Visual Studio'da yerleşik destekleniyor.
- Capstone'daki karşılığı `TurkiyeSimulasyonu.Api.http`.

## Bu gün düzelttiğim yanlışlar

- **"API'nin döndürdüğü JSON = db.json"**
  - Doğrusu: `db.json` sadece seed girdisi ve yalnızca `database update` sırasında okunuyor. İstemciye giden JSON'u DTO'lardan JSON yazıcısı üretiyor.
  - Karışıklığın kaynağı json-server dönemi; o zaman db.json hem depoydu hem cevap.
- **"520'deki servis = `ItemService`"** (plandaki eşleme)
  - Doğrusu: 520 web servisini anlatıyor. `ItemService`'in hocadaki karşılığı Business kutusu.
- **Kapalı kitap özetinde "her şey karşılıyor"**
  - Ders: Karşılık satırında bir dosya adı ya da gün numarası yoksa karşılık yazılmamış demektir.
