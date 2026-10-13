# Gün 46 · Kurs haritası (414–422)

**Bölüm 3'ün hedefi:** Bölüm 2'de HTML/CSS ile yapılan statik e-ticaret sayfası veritabanına bağlanıp dinamik hâle gelecek.

**Bölümün konuları:** veritabanı → SQL → ORM → EF Core → migrations → data seeding → sorgulama → listeleme, filtreleme → ViewComponent.

Hoca EF Core **9.x** kurdu. Capstone'da sen **10.x** kuruyorsun (paket kuralı).

## 1. Veriyi nerede tutarsın? (T)

| | Hafıza (RAM) | Dosya (txt, json, xml, xls) | Veritabanı servisi |
|---|---|---|---|
| Program kapanınca | gider | kalır | kalır |
| Tek bir kaydı bulmak | — | bütün dosya okunur | sorgu: `SELECT * FROM Urunler WHERE Id = 5` |
| İki kişi aynı anda yazarsa | — | biri diğerinin yazdığını ezer | veritabanı yönetir |
| Kurallar (tekil alan, tablolar arası bağ) | — | yok | var |

**Capstone'daki karşılığı:**
- Bugün ilk iki sütundasın: `db.json` (dosya) açılışta bir kez belleğe okunuyor (`SeedData`).
- Yorum yazma ucu bu hâlde yazılsaydı, eklenen yorum API yeniden başlayınca kaybolurdu.
- Hafta 2'nin bitiş ölçütü ("yorum, API yeniden başlatılınca da duruyor") tam olarak 1. ve 3. sütun arasındaki fark.

**"SQL" kelimesinin iki anlamı var:**
1. Verileri tablolarda ve tablolar arası bağlarla tutan veritabanı türü (relational): SQL Server (MsSQL), MySQL, PostgreSQL, SQLite.
2. Bu veritabanlarına soru sorma dili.

NoSQL türü tablo yerine örneğin JSON belgeleri tutar (MongoDB). Capstone relational, SQL Server.

## 2. Frontend neden veritabanına doğrudan bağlanmaz? (T)

- Veritabanına bağlanmak için adres, kullanıcı adı ve parola gerekir (slayttaki host address / username / password).
- Frontend kodu ziyaretçinin tarayıcısında çalışır; içindeki her şeyi herkes okuyabilir (DevTools → Sources).
- Parola frontend'de olsaydı herkes `DELETE FROM Urunler` çalıştırabilirdi.
- Backend'in işi bu yüzden iki tane: anahtarı saklamak ve kuralları uygulamak (kim neyi görebilir, ne yazabilir).

**Capstone'daki karşılığı:**
- Bağlantı adresi API'nin `appsettings.Development.json`'unda; web deposunda hiç yok.
- Vite'ta `VITE_` ile başlayan her değişken derlenen JS'e gömülür ve herkese açıktır.
- `VITE_API_URL` sorun değil, çünkü adres zaten açık. Bir sır ise asla `VITE_` değişkenine konmaz.

## 3. Tablolar arası bağ: hocanın üç tablosu (T)

Slayttaki örnek: Products, Categories ve iki sütunlu bir tablo (`Product_Id`, `Category_Id`).
- Bir telefon hem "Telefon" hem "Elektronik" kategorisinde olabiliyor. Yani bir ürünün çok kategorisi, bir kategorinin çok ürünü var.
- Bu bağ ne ürün satırına ne kategori satırına tek bir sütun olarak yazılabilir.
- Çözüm üçüncü bir tablo: her satırı "şu ürün şu kategoride" der. Bu ilişkiye **many-to-many**, aradaki tabloya **join table** deniyor.

**Capstone'daki karşılığı:**
- Her öğe tek bir kategoriye ait (zod'da `categoryId` tek değer).
- Bu yüzden bağ, öğe satırındaki tek bir sütun (`CategoryId`). Bu **one-to-many**; join table yok.
- Bir gün bir öğe iki kategoride görünecek olursa yapı join table'a döner. O değişiklik bir migration olur.

## 4. ORM (T)

Notundaki tanım doğru. Eksik kalanlar:

- **Ne yapmaz:** SQL'i ortadan kaldırmaz, sadece senin yerine yazar. Ürettiği SQL'i bilmezsen farkında olmadan yavaş sorgu yazarsın. Gün 49'daki N+1 deneyi tam olarak bu.
- **Yol 1: önce C# sınıfı, tablo ondan üretilir** (hocanın ve capstone'un yolu). Buna **Code First** deniyor.
- **Yol 2: var olan bir veritabanından C# sınıfı çıkarılır.** Buna **Database First** deniyor. Türkiye'de eski kurumsal projelerde çok çıkar.
- **Tablolar "otomatik" oluşmuyor:** Sen bir migration oluşturup komutla uygularsın (Gün 47).
- **Ekosistemler:** Node'da Prisma ve TypeORM, Java'da Hibernate, Python'da SQLAlchemy.
- **.NET'te ikinci yaygın araç Dapper:** SQL'i sen yazarsın, sonucu nesneye o çevirir. Sektörde sık görülen düzen şu: işlerin çoğu EF Core ile yapılır; çok sık çalışan ve yavaşlığı ölçülmüş birkaç sorgu elle SQL ile yazılır.

## 5. EF Core ve provider (K: 419)

- **Provider:** EF Core .NET'in ORM'i; ASP.NET'e özel değil, konsol ve masaüstü uygulamalarında da çalışır. Her veritabanıyla konuşmak için ayrı bir sürücü paketi kullanır; bu pakete **provider** deniyor.
  - Hoca: `Microsoft.EntityFrameworkCore.Sqlite`.
  - Capstone: `Microsoft.EntityFrameworkCore.SqlServer`.
- **`UseSqlite` / `UseSqlServer`:** Bu metotlar provider paketinden gelir. Derleyici bulamıyorsa ya paket ya da `using Microsoft.EntityFrameworkCore;` eksiktir (EF dokümanı).
- **SQLite:** Sunucusu olmayan, tek bir dosyadan ibaret veritabanı (`store.db`). Kurulumu yok; eğitim, demo ve mobil uygulamalar için çok uygun.
- **Capstone'da neden SQLite değil:** Canlıda Azure SQL (SQL Server motoru) kullanacağız. Geliştirmede de aynı motoru kullanmak "bende çalışıyordu" sürprizlerini önler: büyük/küçük harf karşılaştırması, tarih tipleri, aynı anda yazma davranışı motordan motora değişir.

## 6. Entity (Ç: 420)

- **Ne yaptı:** `Models/Urun.cs` → `Id`, `UrunAdi`, `Fiyat`, `Aktif`.
- **Neden:** Tablonun C# hâli. Her `Urun` nesnesi bir satır (slayttaki oklar).
- **Capstone'daki karşılığı:** `Entities/Category.cs`, `Item.cs`, `Comment.cs`.

**Hocanın kodunda iki noktayı örnek alma:**

**`public string UrunAdi { get; set; } = null!;`**
- Derleyiciyi susturur ama adı boş bir `Urun` oluşturmana izin verir. Hata ancak kaydederken, veritabanında çıkar.
- Capstone'da `required` kullanıyorsun: alanı unutursan derleme anında durur. EF dokümanı C# 11 ve sonrası için bunu öneriyor.
- `= null!` ise navigation'larda doğru seçim (yine doküman).

**`double Fiyat`**
- `double` ondalık sayıları ikilik sistemde yaklaşık tutar: C#'ta `0.1 + 0.2 == 0.3` sonucu `false`. Para hesabında kuruş farkları birikir.
- Sektörde para için `decimal` kullanılır.
- Capstone'da para yok, ama bu satırı kalıp olarak alma.

## 7. DbContext (Ç: 421)

- **Ne yaptı:**
  - `DataContext : DbContext`
  - `DbSet<Urun> Urunler`
  - constructor: `DataContext(DbContextOptions<DataContext> options) : base(options)`
- **Constructor neden var:** Ayarlar (hangi veritabanı, hangi adres) Program.cs'teki `AddDbContext`'ten servis kutusu (DI) üzerinden gelir ve `base(options)` ile EF'e verilir. Hocanın yorumundaki `new DataContext()` yazılmayacak: nesneyi DI üretir ve istek bitince atar (kaynak notu).
- **Düzeltme, DbSet şemayı tasarlamaz:**
  - Şemayı entity sınıfı, kurallar ve EF'in varsayılanları belirler.
  - DbSet o tabloya açılan kapıdır: `_context.Urunler.Where(…)` ile okursun, `.Add(…)` ile eklersin.
- **Connection ve Command:** DbContext bağlantıyı açıp kapatır, LINQ'i SQL'e çevirip veritabanına komut olarak gönderir.
- **Slaytta olmayan görev:** Getirdiği ve eklediği nesneleri izler (tracking). `SaveChanges` çağrılınca farkları tek seferde yazar (kaynak notundaki defter).
- **Capstone'daki karşılığı:** `Data/AppDbContext.cs`, `sealed`, üç DbSet; kurallar `OnModelCreating`'de.

## 8. Connection string ve ortam (K: 422)

Tanımın ("adres kartı ve giriş bileti") doğru.

**Her veritabanında farklı görünür:**

| Veritabanı | Connection string | Not |
|---|---|---|
| SQLite | `Data Source=store.db` | sadece dosya yolu, çünkü sunucu yok |
| LocalDB | `Server=(localdb)\MSSQLLocalDB;Database=…;Trusted_Connection=True` | Windows oturumunla girilir, parola yok |
| Azure SQL | sunucu + kullanıcı + parola | sır; depoya girmez (Gün 52) |

**Düzeltme: fark "development ile runtime" değil, ortam (environment).** İkisi de çalışma anı.
- Uygulama hangi ortamda çalıştığını `ASPNETCORE_ENVIRONMENT` değişkeninden öğrenir. Senin `launchSettings.json`'unda bu değer `Development`.
- `appsettings.json` her ortamda okunur. `appsettings.Development.json` sadece Development'ta okunur ve aynı ayarların üstüne yazar.
- Ortam belirtilmezse varsayılan Production'dır. Azure'da da durum bu: Development dosyası okunmaz.
- Bu yüzden canlıda bağlantı adresi yoksa capstone açılışta durur (`?? throw`). Adres Gün 52'de Azure'un ortam değişkenine konacak.

**Hocanın Program.cs'i adresi `AddDbContext`'in lambda'sının içinde okuyor:**
- O lambda açılışta değil, her istekte context oluşturulurken çalışır.
- Adres eksikse hata ilk veritabanı işinde çıkar.
- Capstone'da adres lambda'nın dışında, açılışta bir kez okunur. Yoksa uygulama hiç açılmaz (fail fast).

**Adlandırma:** Hoca `DefaultConnection` diyor (VS şablonlarının adı), capstone `Default`. İkisi de olur; önemli olan JSON'daki adla koddaki adın aynı olması.

## 9. `mvc-capstone-esleme.md`'ye eklenecek satırlar

| Hocanın projesi | Capstone | Neden farklı |
|---|---|---|
| `Models/Urun.cs` | `Entities/Category.cs`, `Item.cs`, `Comment.cs` | entity ayrı klasörde; `Models/` dışarı giden şekiller için kalıyor |
| `Models/DataContext.cs` | `Data/AppDbContext.cs` (`sealed`) | veriyle ilgili her şey `Data/`'da |
| `UseSqlite`, `Data Source=store.db` | `UseSqlServer`, LocalDB | canlıyla aynı motor |
| EF Core 9.x | EF Core 10.x | paket kuralı |
| `= null!` (metin alanı) | `required` | eksik alanı derleyici yakalar |
| `double Fiyat` | — (para yok) | para `decimal` olur |
| adres lambda içinde okunuyor | lambda dışında, `?? throw` | fail fast |
| many-to-many (ürün ↔ kategori) | one-to-many (öğe → kategori) | öğenin tek kategorisi var |
