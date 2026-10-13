# Gün 46 · Kaynak — DbContext'in ömrü

- **Doküman:** [DbContext Lifetime, Configuration, and Initialization](https://learn.microsoft.com/en-us/ef/core/dbcontext-configuration/)
- **Oku:** The DbContext lifetime · DbContext in dependency injection for ASP.NET Core · Avoiding DbContext threading issues. Kalan bölümlere (`new` ile oluşturma, factory, ayar tabloları) sadece göz gezdir.
- **Soru:** Why does `AddDbContext` give each HTTP request its own DbContext instance, and what goes wrong when two operations use the same instance at the same time?

## 1. Mantık: bir defter, bir iş

Garson masaya gelir, siparişi deftere yazar; "şunu çıkar, bunu ekle" değişikliklerini de aynı sayfada düzeltir. Masa bitince sayfayı mutfağa bir kerede verir ve kapatır. Sonraki masaya temiz sayfa açar.

DbContext bu defter:
- Sorguyla getirdiği ya da senin eklediğin her nesnenin "geldiği hâl / şimdiki hâl" kaydını tutar. Buna **tracking** deniyor.
- `SaveChangesAsync` çağrılınca farkları bulur ve tek seferde veritabanına yazar.
- İş bitince atılır, tuttuğu kaynakları bırakır. Buna **dispose** deniyor.

Bu "oluştur → izle → kaydet → at" döngüsünün adı **unit of work**. Doküman bunu açıkça söylüyor: bir DbContext tek bir iş için tasarlanmış, ömrü bu yüzden çok kısa.

## 2. Neden her isteğe ayrı bir tane?

Web uygulamasında çoğu zaman bir istek = bir iş: "yorumu kaydet" gelir, iş yapılır, cevap döner. Defterin ömrünü isteğin ömrüne bağlamak bu yüzden doğal varsayılan. `AddDbContext` tam bunu yapar.

ASP.NET Core, bir servis için "kaç tane, ne kadar süre?" sorusuna üç cevap veriyor:

| Kaç tane, ne kadar? | Adı | Senin kodunda |
|---|---|---|
| Uygulama açık kaldıkça tek bir tane | **Singleton** | `SeedData`: db.json açılışta bir kez okunuyor |
| Her istek için bir tane, istek bitince atılır | **Scoped** | `AppDbContext` (bugün) |
| Her istendiğinde yenisi | **Transient** | henüz yok |

Ayarlar DbContext'e constructor'dan gelir. `AddDbContext`'e verdiğin `UseSqlServer(...)` bilgisi `DbContextOptions<AppDbContext>` olarak paketlenir, senin sınıfının constructor'ına, oradan `base(options)` ile EF'e teslim edilir.

## 3. Neden paylaşılmaz?

İki garson aynı anda aynı deftere yazarsa satırlar birbirine girer. DbContext de aynı anda birden çok işin kullanmasına göre tasarlanmamış; yani **thread-safe** değil. EF bunu fark ederse şu hatayı fırlatır:

> A second operation started on this context before a previous operation completed.

Fark edemezse sonuç daha kötü: tanımsız davranış, çökme, bozuk veri. İki yaygın yol:
- **`await`'i unutmak:** async bir sorgu bitmeden aynı context'le ikinci işi başlatmak. Kural: EF'in async metotları hemen `await` edilir.
- **Context'i paylaşmak:** birden çok iş parçacığının aynı nesneye erişmesi. Scoped kayıt her isteğe ayrı nesne verdiği için bunu çoğu durumda kendiliğinden önler.

## 4. Sektörde karşına çıkacak durumlar

**a. Kısa ömürlüyü uzun ömürlünün içine koymak (yarın karşına çıkacak).** Singleton bir servis constructor'ından DbContext isterse, o tek DbContext uygulama boyunca yaşar ve bütün isteklere paylaşılır. Bölüm 3'teki her şey olur, üstüne izlenen nesneler bellekte birikir. Buna **captive dependency** deniyor. ASP.NET Core, Development ortamında bunu açılışta yakalar ve uygulamayı başlatmaz: `Cannot consume scoped service '…' from singleton '…'.` `SeedData` singleton; seed'i yazarken DbContext'i ona constructor'dan vermek bu hatayı üretir. Doğru yolu Gün 47'de seçeceğiz.

**b. İstekten doğmayan kod.** "Her gece eski kayıtları temizle" gibi arka planda çalışan bir işe kendiliğinden bir istek kapsamı açılmaz. Ya elle kapsam açılır (`IServiceScopeFactory`), ya da `AddDbContextFactory` ile ihtiyaç anında yeni context üretilir. Factory'den alınan context'i `using` ile atmak senin sorumluluğun.

**c. Paralel sorgu cazibesi.** "İki sorguyu `Task.WhenAll` ile aynı anda atayım, hızlansın" aynı context'le yapılırsa bölüm 3'teki hata çıkar. Ya sırayla `await` edilir ya da her sorguya ayrı context verilir.

**d. Hatadan sonra devam etmek.** EF'in fırlattığı `InvalidOperationException` context'i kurtarılamaz bir duruma sokabilir. Yakalayıp aynı context'le işe devam edilmez; bu hata bir programlama hatasının işaretidir.

**e. Yüksek trafik.** Saniyede binlerce istek alan servislerde her istekte context kurmanın maliyeti hissedilebilir. `AddDbContextPool` context nesnelerini temizleyip yeniden kullanır. Capstone'da gerek yok; ölçmeden açılmaz.

## 5. Bugünkü koddaki karşılığı

```csharp
var connectionString = builder.Configuration.GetConnectionString("Default")
    ?? throw new InvalidOperationException("Connection string 'Default' not found.");
builder.Services.AddDbContext<AppDbContext>(options => options.UseSqlServer(connectionString));
```

- **Dün `?? []`, bugün `?? throw`. Neden?** CORS listesi eksikse güvenli bir varsayılan vardı: kimseye izin yok. Bağlantı adresinin güvenli varsayılanı yok; eksikse uygulama hiçbir veri işini yapamaz. O zaman ilk istekte, kullanıcının önünde değil, açılışta düşsün. Buna **fail fast** deniyor. Ayrıca `TreatWarningsAsErrors` açıkken null olabilecek bir değeri `UseSqlServer`'a vermek derlemeyi zaten durdurur.
- **`sealed`:** Doküman, miras alınmak üzere tasarlanmamış DbContext'i `sealed` yapmayı iyi uygulama olarak gösteriyor.
- **Bugün context'i hiçbir controller istemiyor.** Kayıt yapıldı; ilk kullanan Gün 47'deki seed, ilk uç Gün 48.

## 6. park.md bağlantısı

[gun42] ömürler maddesinin DbContext kısmını bu not kapatıyor. Genel üçlü bölüm 2'deki tabloda; maddeyi Gün 51'in park bloğunda tamamen kapatırsın.
