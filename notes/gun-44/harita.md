# Gün 44 — Dersler 406–409: View'a liste vermek ve döngüyle basmak

Kaynak: Sadık Turan, ASP.NET Core kursu, dersler 406–409 (hepsi Ç)

**Kısa özet:** Hoca tek bir nesne yerine bir ürün listesini View'a verdi. View'da bu listeyi önce index'le, sonra `foreach` döngüsüyle bastı. Burada iki ayrı fikir var: **liste** birden fazla nesneyi bir arada tutan veri yapısı; **döngü** ise o listeyi eleman sayısını bilmeden gezen basma yöntemi. Capstone'da ikisi de zaten var, ama iki depoya bölünmüş durumda: listeyi API döndürüyor, döngüyü React'teki `.map()` yapıyor.

## 1. Akış

```
MVC:      Request → Controller ──liste──▶ View (@model List<Product>) → HTML → tarayıcı
                        ↕
                      Model (Product class'ı = verinin şekli)

Capstone: Request → Controller ──Ok(liste)──▶ serializer → JSON [...] → React .map() → HTML
```

View'ın rolünü capstone'da React üstleniyor. MVC'de HTML sunucuda üretiliyor, capstone'da tarayıcıda.

## 2. Listeyi kurmak

Hocanın kodunun şekli:

```csharp
List<Product> urunler = new List<Product>
{
    new Product { urunBaslik = "IPhone 15", urunFiyat = 80000, urunSatistami = true },
    new Product { urunBaslik = "IPhone 16", urunFiyat = 90000, urunSatistami = true },
};
return View(urunler);
```

- `new Product { urunBaslik = "..." }`: Önce boş bir nesne oluşturuluyor, alanlar hemen ardından atanıyor. Bu kısa yazıma **object initializer** deniyor. Hocanın Gün 43'te record değil class seçmesinin sebebi bu: positional record'da önce boş nesne oluşturulamaz.
- `new List<Product> { a, b }`: Listeyi oluşturup elemanları tek tek eklemenin kısa yazımı. Buna **collection initializer** deniyor.
- `[a, b]`: C# 12'nin JS dizi yazımına benzeyen biçimi; hedef tip dizi de olsa liste de olsa çalışıyor. Buna **collection expression** deniyor. Yeni kodda sektör bu yazıma yöneliyor, eski kodda `new List<T> { }` görürsün.
- Hocanın satırında tip iki kez yazılı. `var urunler = new List<Product> { ... }` ya da `List<Product> urunler = [ ... ];` aynı işi yapar.

## 3. View'ın ne beklediğini bildirmesi: `@model`

View dosyasının en üstündeki `@model` satırı, View'ın hangi tipte veri beklediğini söylüyor. Hoca üç seçenek gösterdi: `List<Product>`, `IEnumerable<Product>`, `Product[]`. Bunlar eşit seçenekler değil:

| Tip | Ne söylüyor | Ne zaman |
|---|---|---|
| `IEnumerable<T>` | "Baştan sona gezilebilir" | Sadece gezip basacaksan. Liste, dizi ve LINQ sonucunun üçünü de kabul eder |
| `IReadOnlyList<T>` | Gezilebilir + index + `Count`, değiştirilemez | Index ya da sayı lazımsa |
| `List<T>` / `T[]` | Somut tip, değiştirilebilir | Listeyi kuran ya da değiştiren kod için |

**Sektör kuralı:** Bir metot ya da View girdi olarak, işini görecek en genel tipi ister. Böylece onu çağıran tarafa "elindeki şeyi listeye çevir" diye bir yük bindirmez.

**LINQ bağlantısı (Gün 43):** LINQ bir liste değil, bir tarif döndürür; liste `ToList()` çağrılınca oluşur. Gün 48'deki bedeli şu: View'a ya da serializer'a `ToList()` yapılmamış bir veritabanı sorgusu verilirse, sorgu sayfa çizilirken çalışır. Bu durumda veritabanı hatası controller'da değil, HTML ya da JSON yazılırken patlar. Bu yüzden sonuç controller'dan dönmeden önce sabitlenir.

**Kablo:** JSON'da tek bir koleksiyon tipi var: dizi. `List`, `T[]` ve `IEnumerable` aynı `[...]`'e dönüşür. Tip seçimi kablo için değil, kodun içi için önemli.

## 4. Index'ten döngüye

```cshtml
@* index ile: View eleman sayısını önceden bildiğini varsayıyor *@
<h5>@Model[0].urunBaslik</h5>

@* döngü ile: sayı ne olursa olsun çalışıyor *@
@foreach (var urun in Model)
{
    <h5>@urun.urunBaslik</h5>
}
```

**Döngüye geçmenin asıl sebebi** "index uğraştırır" değil. View yazılırken listenin kaç elemanlı olacağı bilinemez:
- Liste 2 elemanlıysa `@Model[2]` exception fırlatır ve sayfa 500 döner.
- Liste 300 elemanlıysa 3'ü görünür, 297'si sessizce kaybolur.

**Döngü değişkeni (`urun`):**
- Tipi koleksiyondan gelir. `foreach (var urun in Model)` yazınca derleyici tipi `Product` olarak çıkarır. Sektörde `foreach`'te `var` yaygın. `Product` class'ı döngü için yazılmadı; veri o şekle sahip olduğu için var.
- Kopya değil, elemanın kendisi. Class bir reference type, yani `urun.urunBaslik = "x"` listedeki asıl nesneyi değiştirir.
- Değişkenin kendisi salt okunur. `urun = new Product();` derlenmez (CS1656).

**React bağlantısı:** Bu döngü `items.map(item => <Card />)`'un Razor hâli. Razor'da `key` yok, çünkü HTML bir kez üretilip gönderiliyor; karşılaştırılacak bir önceki çizim yok. React aynı sayfayı tekrar tekrar çizdiği için hangi kartın hangisi olduğunu `key` ile takip ediyor.

**`~/img/...`:** Baştaki `~` uygulamanın kök adresi demek. Razor bunu gerçek adrese çeviriyor. Capstone'da karşılığı yok.

## 5. Döngü kod sorununu çözer, veri hacmi sorununu çözmez

1000 ürünü döngüyle basmak üç satır kod. Ama sonuç 1000 kartlık tek bir HTML sayfası ya da 1000 elemanlı tek bir JSON cevabı olur. İkisi de yavaş iner ve kullanıcının çoğu ilk 20'den sonrasına hiç bakmaz. Çözüm, veriyi sunucuda sayfalara bölmek (Gün 54). "Üç soru"daki "veri bir milyon kayıt olursa" sorusu tam olarak bu.

## 6. Hocanın capstone'dan ayrıldığı yerler

**a) `.csproj`'da `<Nullable>disable</Nullable>`**
- Derleyici null'ı takip etmeyi bırakıyor ve CS8618 gibi uyarılar kayboluyor. Ama risk yerinde duruyor: null bir alana erişim çalışma anında yine `NullReferenceException` fırlatır, sadece önceden haber gelmez. Termometreyi kırmak ateşi düşürmez.
- **Gün 43 ölçümünü değiştiriyor:** Bu ayarla `string categoryId` zorunlu sayılmaz. `GET /items` 400 değil 200 döner, `categoryId` sessizce `null` gelir. Aynı kod iki projede farklı davranıyor; hocanın kodunu capstone'a taşırken buna dikkat.
- **Sektör:** Yeni projelerde açık (.NET 6'dan beri şablonların varsayılanı). Eski projelerde kapalı; dosya dosya açılarak taşınıyor. Kurs projesinde kapalı olması kabul edilebilir. Capstone'da açık kalıyor, `TreatWarningsAsErrors` da devam ediyor.

**b) Adlandırma: `urunBaslik`, `urunSatistami`**
- C#'ta property'ler PascalCase yazılır: `Title`.
- `urun` öneki zaten class'ın adında var: `Product.Title` yeterli.
- bool'lar soru gibi okunur: `IsOnSale`.
- JSON'daki camelCase'i (`"title"`) sen yazmıyorsun; ASP.NET Core'un serializer'ı varsayılan olarak çeviriyor. Capstone'da record'ların PascalCase, JSON'un camelCase olmasının sebebi bu.

**c) Fiyat:** Para için `int` ya da `double` değil, `decimal` kullanılır (Gün 43).

## 7. Sık karışanlar

| Yanlış model | Doğrusu |
|---|---|
| "Liste ile ekrana basmak" | Liste tutar, döngü basar; iki ayrı iş |
| "Döngü, index uğraştırdığı için" | View eleman sayısını bilemez: azsa 500, çoksa kayıp |
| "Döngü değişkeni için önce tip tanımı gerekir" | Değişkenin tipi koleksiyondan gelir; class verinin şekli için var |
| "Döngü değişkeni elemanı saklar" | Elemanın kendisine işaret eder; kopya değil |
| "LINQ sonucu bir listedir" | Bir tariftir; liste `ToList()` ile oluşur |
| "Capstone'da kullanacağız" | Zaten var: liste = API'deki `Ok(query.ToList())`, döngü = React'teki `.map()` |
