# Gün 43 — Harita 1: Dersler 402–405

**Günün çizgisi:** Hoca, veriyi controller'dan View'a taşımanın üç basamağını gösterdi: tek tek değişkenler → diziler → class. Her basamak bir öncekinin sorununu çözüyor. Capstone'da View yok; aynı veri `Ok(...)` ile JSON'a çevrilip React'e gidiyor.

## 1. Veri tipleri

**Ne yaptı:** Bir telefon satış kartının alanlarını tipleriyle tanımladı: başlık, açıklama, resim → `string`; fiyat → `int`; stokta var mı → `bool`.

| Veri | C# tipi | Not |
|---|---|---|
| Metin (`"Samsung S25"`) | `string` | Çift tırnak. Karakterlerden oluşan metin |
| Tek karakter (`'A'`) | `char` | Tek tırnak. "karakter → string" yanlış eşleme |
| Tam sayı | `int` | ±2,1 milyar; daha büyüğü için `long` |
| Ondalıklı sayı | `double` | Ölçüm, oran, ortalama |
| Para | `decimal` | `59999.90m` (aşağıda) |
| Doğru/yanlış | `bool` | `true` / `false` |

**Neden önemli:**
- `double` ondalık sayıları ikilik tabanda yaklaşık tutar: `0.1 + 0.2 == 0.3` → `false`. JS'teki `number` da double olduğu için aynısı orada da olur. Para için bu kabul edilemez, kuruşlar zamanla kayar. Sektör standardı para için `decimal`, onluk tabanda tam tutar: `0.1m + 0.2m == 0.3m` → `true`. Veritabanındaki karşılığı genelde `decimal(18,2)`.
- Hocanın `int fiyat`'ı kuruşu tutamaz (59.999,90 TL yazılamaz). Ders için yeterli, gerçek e-ticarette değil.
- JS'te tek bir `number` tipi var; C#'ta tipi sen seçersin ve seçim davranışı değiştirir: `7 / 2` → `3` (int / int), `7.0 / 2` → `3.5`.

**Capstone:** `order` ve `distribution` → `int`, `createdAt` → `DateTime`. Para yok. Ortalama puan hesaplanırsa `double` yeterli; o bir ölçüm, para değil.

## 2. ViewData ile View'a veri taşıma

**Ne yaptı:** Kartın alanlarını controller'da `ViewData["Baslik"] = "...";` diye tek tek koydu. `.cshtml`'de `@ViewData["Baslik"]` ile alıp ekrana bastı.

**Mantık:** ViewData, controller ile View arasında duran bir kutu. İçine "anahtar → değer" çiftleri atıyorsun (JS'te `kutu["baslik"] = ...` gibi). Kutu her şeyi kabul eder ama karşılığında hiçbir şeyi garanti etmez:
- Anahtarı yanlış yazarsan (`"Basilk"`) derleme geçer ve ekranda sessizce boşluk çıkar.
- Değer `object` olarak durur. Sayıyla işlem yapmak için tipine geri çevirmen (cast) gerekir.

Tipi derleme anında denetlenmeyen bu tür taşıyıcılara **weakly typed** deniyor. Hocanın ardından Model'e geçmesinin sebebi bu. `ViewBag` aynı kutunun `ViewBag.Baslik` biçiminde yazılan hâli; zayıflığı da aynı.

**Capstone:** Karşılığı yok. API'de View yok; veri controller'dan `Ok(...)` ile çıkıp JSON'a çevriliyor.

## 3. Diziler

**Ne yaptı:** Her kartı ayrı değişkenlerle tanımlamak yerine değerleri dizilere koyup index ile aldı.

**Tuzak:** Her özellik için ayrı bir dizi tutulursa (başlıklar dizisi, fiyatlar dizisi), bir kartın verisi birkaç diziye dağılır ve onları bir arada tutan tek şey aynı index olur. Bir diziye fazladan bir eleman girerse bütün kartlar sessizce kayar; React'teki `key={index}` kaymasının akrabası. Bu yapıya **parallel arrays** deniyor. Class'ın çözdüğü sorun tam olarak bu: bir kartın bütün alanları tek bir nesnede durur.

**Ek:** C# dizisi (`T[]`) sabit boyutlu, büyüyebilen liste `List<T>`. Aradaki fark park.md'deki [gun42] maddesinde.

**Capstone:** Bütün listeler `IReadOnlyList<T>`.

## 4. Model ve class

**Ne yaptı:** `Models` klasöründe `Course` class'ını (`Title`, `Image`) yazdı. `CourseController`'da `new Course()` ile bir nesne oluşturdu, alanlarını tek tek atadı ve View'a verdi; View ekrana bastı. Slayttaki zincir: Request → Controller (veriyi db'den Course listesi olarak alır) → Model → View → HTML.

**Class:** Bir şablon. `Urun` class'ı `urunAdi` ve `fiyat` alanlarını tanımlar; `urun1` ve `urun2` o şablondan üretilmiş nesnelerdir (instance). C#'ta class bir reference type'tır: Nesne heap'te durur, değişken nesnenin adresini tutar. `var b = a;` yeni bir nesne üretmez, aynı nesneye ikinci bir ok çizer; JS'teki nesnelerle aynı. Karşıtı value type: `int`, `double`, `bool`, `DateTime` (ve her `struct`) atamada kopyalanır. `string` bir reference type'tır ama değiştirilemez ve `==` içeriği karşılaştırır.

**Model:** MVC'deki "M" bir dosya türü değil, bir rol: uygulamanın taşıdığı veri. Kodda sıradan C# class'larıdır. `Models` klasörü bir kural değil, proje şablonunun koyduğu bir gelenek. "Model" kelimesi ileride ayrışacak birkaç şekil için kullanılıyor:
- Veritabanı tablosuna karşılık gelen şekil → entity (Gün 46)
- Dışarıya gönderilen JSON'un şekli → DTO (Gün 48)
- İstekten gelen veriyi karşılayan şekil → girdi modeli (Gün 50)
- Bir View için hazırlanan şekil → view model (MVC)

Doğrulama kuralları modelde attribute olarak durabilir (Gün 55). İş mantığı ise sektörde modelde değil, servislerde durur (Gün 49).

**Hocanın kodundaki iki uyarı:** Sekmede `Course.cs 2` yazıyor ve `Title`/`Image` altında dalgalı çizgiler var. `string Title` "bu asla null olmaz" diyor, ama `new Course()` ilk anda `Title`'ı null bırakıyor; derleyici bunu uyarıyor (CS8618). Capstone'da uyarılar hata sayıldığı için bu kod orada derlenmezdi. Çözümler:
- `public required string Title { get; set; }`: Nesneyi oluşturan kişi alanı doldurmak zorunda kalır (C# 11). DTO'larda yaygın.
- `= string.Empty;`: Varsayılan değer verir. Eski projelerde ve Türkiye'deki birçok projede görülür, ama boş başlığı geçerli bir değer gibi gösterip hatayı gizleyebilir.
- `string? Title`: Null gerçekten izinliyse.
- Positional record (`record Course(string Title, string Image)`): Değerler nesne oluşturulurken verilir. Capstone'da dün yaptığın bu.

Atamaları tek tek yazmak yerine sektörde **object initializer** kullanılır: `var kurs1 = new Course { Title = "Django Kursu", Image = "1.jpg" };`. `required` ile birlikte kullanılırsa unutulan bir alan derleme hatası verir.

**Capstone karşılığı ve karar:**
- Seed verisi (`Category`, `Item`, `Comment`, `SeedData`) record olarak kalıyor. Bu veri sadece okunup taşınıyor; içerik eşitliği ve değişmezlik tam istediğimiz şey.
- Gün 46'daki entity'ler class olacak. EF Core bir kaydı nesne kimliğiyle ("aynı nesne mi?") takip ediyor, record'un içerik eşitliği bununla çakışıyor. Microsoft'un record dokümanı da record'u EF Core entity'leri için uygun görmüyor.
- Gün 48'deki DTO'lar yine record olacak.
- Hocanın `return View(kurs1)` dediği yerde capstone `return Ok(...)` diyor. View'ın HTML üretme işini React yapıyor.
