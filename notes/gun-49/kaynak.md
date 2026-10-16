# Gün 49 — Kaynak: Service lifetimes

**Okunan:** Microsoft Learn — *Service lifetimes* (Transient, Scoped, Singleton) ve *.NET dependency injection* sayfasının *Scope validation* bölümü.

**Soru:** *Why must a service that receives `AppDbContext` through its constructor not be registered as a singleton, and in which environment does .NET catch this mistake for you?*

**Cevabım:** `AppDbContext` varsayılan olarak scoped kaydedilir ve thread-safe değildir. `AppDbContext` alan bir servis singleton olarak kaydedilirse `AppDbContext` de onun içinde hapsolur (captive dependency). Her HTTP isteğinin sonunda silinmek yerine uygulama boyunca yaşar. Bu, birden fazla isteğin aynı anda erişmesiyle çakışmalara, bellek sızıntısına ve bozuk veri takibine yol açar. .NET bu hatayı Development ortamında yakalar. ✅

## Mantık: defter benzetmesi

`AppDbContext` tek bir isteğin defteri gibi:
- O istekte veritabanından ne okuduğunu not eder (change tracker).
- Aynı anda iki kişinin yazmasına dayanmaz (thread-safe değil).
- Bu yüzden her istek kendi defterini alır ve istek bitince defter atılır.

DI bir nesneyi kurarken ne kadar yaşatacağını kayıt satırından bilir. Buna **lifetime** deniyor.

| Lifetime | Kaç nesne | Ne zaman atılır | Capstone'da örnek |
|---|---|---|---|
| Transient | Her istenişte yeni bir nesne | Onu isteyen scope bitince | Yok |
| Scoped | Her HTTP isteğine bir nesne | İstek bitince | `AppDbContext` (`AddDbContext`'in varsayılanı), `IItemService` |
| Singleton | Uygulama boyunca tek nesne | Uygulama kapanınca | Eski `SeedData` (Gün 49'da silindi), `ILogger<T>`, `IConfiguration` |

- Bir singleton aynı anda bütün isteklerin elindedir, bu yüzden **thread-safe** olmak zorundadır. `SeedData` bu yüzden singleton olmaya uygundu: `record` + `IReadOnlyList`, kimse değiştiremiyordu.
- Aynı istekte iki servis `AppDbContext` isterse **aynı** nesneyi alır. Gün 61'deki transaction buna dayanacak.

## Kural: bir servis bağımlılıklarından uzun yaşamaz

| Kim kimi kullanıyor | Sonuç |
|---|---|
| Scoped → Singleton | ✅ Kısa ömürlü olan, uzun ömürlüyü kullanabilir |
| Scoped → Scoped / Transient | ✅ |
| Singleton → Scoped | ❌ **Captive dependency** |
| Singleton → Transient | ⚠️ Transient de bir kez kurulur ve singleton'ın içinde yaşar. O transient `AppDbContext` alıyorsa sorun aynı |

**Captive dependency:** Uzun ömürlü bir nesnenin kısa ömürlü bağımlılığını içinde tutması ve böylece onun ömrünü de uzatması. Singleton bir `ItemService` ilk kurulduğunda bir `AppDbContext` alır ve uygulama kapanana kadar bırakmaz.

## Neler bozulur

1. **Aynı anda gelen iki istek:** EF bunu fark eder ve exception fırlatır: *"A second operation was started on this context instance before a previous operation completed."* Tek kullanıcıyla geliştirirken hiç görünmez; yük altında rastgele çıkar.
2. **Bayat veri:** Takip edilen (tracking) bir sorgu hafızada zaten olan bir kaydı döndürürse EF hafızadaki nesneyi verir ve değerlerini veritabanından gelenle ezmez. Başka biri kaydı değiştirse de eski hâli görünmeye devam eder.
3. **Büyüyen hafıza:** Takip edilen her nesne defterde kalır. Defter hiç atılmadığı için bellek sürekli büyür.

Capstone'daki okuma uçları DTO'ya projection yaptığı için EF hiçbir şeyi takip etmiyor. 2. ve 3. sorun yazma uçlarıyla (Gün 50) başlar. 1. sorun bugün de geçerli.

## Scope validation: .NET hatayı nerede yakalıyor

Bu kontrol varsayılan olarak **sadece Development'ta** açık. İki şeye bakıyor:
- Scoped bir servis, bir scope'un dışında, yani kök sağlayıcıdan isteniyor mu?
- Scoped bir servis bir singleton'a enjekte ediliyor mu?

**Ne zaman yakalıyor:** açılışta doğrudan yakalar, hangisi singleton açıldıysa onu işaret eder hata mesajında.

**Production'da bu kontrol yok.** Uygulama sorunsuz çalışıyor görünür ve bütün istekler aynı defteri paylaşır. Sorun ancak iki istek aynı anda geldiğinde, 1. maddedeki exception olarak ara sıra görünür. Yerelde hemen yakalanan hata, canlıda peşine düşülmesi zor bir hataya dönüşür.

## Singleton'ın veritabanına ihtiyacı varsa

Bazı şeyler doğası gereği singleton'dır. Örneğin arka planda zamanlanmış iş yapan servisler (**hosted service**; mesela her gece Seçilmişler'i hesaplayan bir iş) ve önbellekler. Bunların veritabanı ihtiyacı için iki standart yol var:

- **Kendi scope'unu açmak:** `IServiceScopeFactory` alınır. Her iş için `using var scope = _scopeFactory.CreateScope();` açılır ve `scope.ServiceProvider.GetRequiredService<AppDbContext>()` ile context istenir. Scope bitince defter atılır.
- **Context fabrikası:** `AddDbContextFactory<AppDbContext>()` ile kaydedilir, `IDbContextFactory<AppDbContext>` enjekte edilir. Her iş için `CreateDbContext()` kısa ömürlü bir defter üretir, `using` ile atılır.

Ortak fikir: Singleton defterin kendisini değil, **defter üretme yolunu** tutar.

## Capstone kararı

- `IItemService`, `ICategoryService` ve `ICommentService` `AddScoped` ile kaydediliyor. Hoca da `AddScoped<IProductService, ProductManager>()` kullanıyor; gerekçe aynı.
- `SeedData` singleton'ı silindi; seed dosyası sadece `database update` sırasında okunuyor.

## Senior notu

Lifetime hatası bir derleme hatası değil. Kod derlenir, tek istekle yapılan her deneme geçer. Development'taki doğrulama bu yüzden bir güvenlik ağı. Yeni bir servis kaydederken sorulacak iki soru:

1. Bu servis paylaşılan bir durum (state) tutuyor mu? Tutuyor ve singleton'sa, thread-safe olmalı.
2. Bağımlılıklarından en kısa yaşayanı hangisi? Servis ondan daha uzun yaşayamaz.
