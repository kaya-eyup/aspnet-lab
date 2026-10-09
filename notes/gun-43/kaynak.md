# Gün 43 — Kaynak: LINQ — aynı satır, iki farklı dünya

Kaynak: https://learn.microsoft.com/en-us/dotnet/csharp/linq/

**Soru:** Aynı `Where` satırı nasıl hem bugünkü bellekteki listede hem de Gün 48'deki SQL Server'da çalışabiliyor?

**Kısa cevap:** Kararı değişkenin tipi veriyor. Bellekteki listede derleyici lambda'yı çalıştırılabilir koda çevirir ve C# filtreyi kendisi yapar. Veritabanı tablosunda aynı lambda'yı kodun tarifine çevirir; EF Core bu tarifi okuyup SQL yazar ve filtreyi SQL Server yapar. Sorgu yazıldığı anda değil, sonucu istendiği anda çalıştığı için zincirin tamamı tek bir SQL'e dönüşebiliyor.

## 1. LINQ ne çözüyor

Eskiden veritabanı sorgusu C# kodunun içine düz metin olarak yazılırdı: `"SELECT * FROM Items WHERE CategoryId = ..."`. Metnin içindeki yazım hatasını derleyici göremez, hata ancak çalışma anında çıkar. Ayrıca her veri kaynağı (veritabanı, XML, web servisi) kendi dilini isterdi. LINQ'te sorgu C#'ın bir parçası: tipler derleme anında denetleniyor ve her kaynak için aynı yazım kullanılıyor.

| JS | LINQ | Not |
|---|---|---|
| `filter` | `Where` | |
| `find` | `FirstOrDefault` | Bulamazsa `undefined` değil, tipin varsayılanı: record için `null`, `int` için `0` (!) |
| `map` | `Select` | |
| `toSorted` | `OrderBy` / `OrderByDescending` | Yeni bir sıra üretir, kaynağa dokunmaz |
| `some` / `every` | `Any` / `All` | |
| `length` | `Count()` | Metot; listedeki `Count` özelliğiyle karıştırma |
| `slice` | `Skip` + `Take` | Gün 54'te sayfalama |

Yakın akrabalar: `First` bulamazsa hata fırlatır; `Single` / `SingleOrDefault` birden fazla eşleşme varsa hata fırlatır. "Bu id'yle kayıt var mı?" sorusu için sektördeki yaygın seçim `FirstOrDefault` + null kontrolü.

**Yazım:** Sayfadaki `from x in xs where … select x` biçimine query syntax, `xs.Where(x => …)` biçimine method syntax deniyor. Derleyici ilkini zaten ikincisine çeviriyor, yani anlam ya da performans farkı yok. Sektörde baskın yazım method syntax; biz de onu kullanıyoruz.

## 2. Sorgu bir tarif, sonuç değil

`var q = items.Where(i => i.CategoryId == "yemek");` satırı hiçbir şey filtrelemez. `q` bir tariftir: "istendiğinde şu kuralla süz". Tarif ancak biri sonucu gezdiğinde çalışır: `foreach`, `ToList()`, `Count()`, `First...` ya da cevabı JSON'a yazan serializer. Buna **deferred execution** (ertelenmiş çalıştırma) deniyor.

Bugün `Ok(_data.Items.Where(...))` yazdığında filtre controller'da değil, serializer cevabı yazarken çalışır. Sonuç aynı; sadece an farklı.

İki sonucu var:
- **İki kez gezilirse iki kez çalışır.** Bellekte bu iki döngü demek, veritabanında iki ayrı sorgu. Aynı sonucu birden çok kez kullanacaksan `ToList()` ile o anda çalıştırıp sabitle (materialize).
- **Tarif, çalıştığı andaki veriyi görür, yazıldığı andakini değil.** Bu, JS'teki closure'ın aynısı (Gün 5). Bizim verimiz değişmeyen bir singleton olduğu için bugün sorun değil.

## 3. Aynı satır, iki yol: tip karar veriyor

```
_data.Items.Where(i => i.CategoryId == id)   // IEnumerable<Item>: bellekteki liste
db.Items.Where(i => i.CategoryId == id)      // IQueryable<Item>:  EF Core'un tablosu
```

- **`IEnumerable<T>`:** Derleyici lambda'yı çalıştırılabilir bir fonksiyona çevirir; buna **delegate** deniyor (JS'te bir değişkende tutulan fonksiyon gibi). `Where` elemanları tek tek dolaşıp bu fonksiyonu çağırır. Bellekteki listelere uygulanan bu yönteme LINQ to Objects deniyor.
- **`IQueryable<T>`:** Derleyici aynı lambda'yı çalıştırmaz. "Sol taraf `CategoryId` özelliği, işlem eşitlik, sağ taraf `id` değişkeni" diye kodun yapısını anlatan bir veri ağacı üretir; buna **expression tree** deniyor. Bu ağacı okuyup hedef dile çeviren araca **provider** deniyor. EF Core bir provider ve ağaçtan `WHERE [i].[CategoryId] = @p0` üretiyor.

"C# akıllıca karar verir" diye bir şey yok: Hangi `Where`'in çağrılacağını derleme anında değişkenin tipi belirliyor.

**Çevirinin bedeli:** Provider sadece SQL karşılığı olan şeyleri çevirebilir. Lambda'nın içinde kendi yazdığın bir C# metodunu çağırırsan (ör. frontend'deki `normalizeForSearch`'ün C# hâli), derleme geçer ama EF Core çalışma anında "could not be translated" hatası verir. Gün 53'te aranabilir metnin ayrı bir sütunda tutulmasının sebebi bu.

## 4. Sektör boyutu: en pahalı hata, sessizce belleğe düşmek

Zincirin ortasında `IQueryable` bir `IEnumerable`'a dönerse sonraki her adım artık SQL olmaz:

```
db.Items.ToList().Where(i => i.CategoryId == id)       // tablonun TAMAMI belleğe, süzme C#'ta
db.Items.AsEnumerable().Where(i => i.CategoryId == id) // aynısı
```

Bu dönüşüm parametre tipi `IEnumerable<Item>` olan bir metoda `db.Items` geçmekle de olur. Derleme geçer, sonuç doğru çıkar, 24 kayıtlık test verisiyle hızlı çalışır. Bir milyon satırda her istek bütün tabloyu çeker; "üç soru"daki "bir milyon kayıt" sorusunun en sık görülen cevabı bu. Dikkat: EF Core'un "could not be translated" hata mesajı çözüm olarak tam da `AsEnumerable` / `ToList` öneriyor. Mesajın önerdiği kolay yol, bu tuzağın ta kendisi.

Sektör kuralları:
- Süzme, sıralama ve sayfalama veritabanında yapılır; `ToListAsync()` zincirin en sonunda gelir.
- Sonuç controller'dan dönmeden önce sabitlenir. Serializer'ın veritabanı sorgusunu kendisi çalıştırmasına bırakılmaz.
- Şüphe varsa EF Core'un ürettiği SQL log'da açılıp okunur (Gün 49'daki N+1 deneyinde yapacağın).

## 5. Capstone karşılığı

- **Bugün:** `_data.Items` bir `IReadOnlyList<Item>`, yani LINQ to Objects. `GET /items/{id}` için `FirstOrDefault`, `GET /items?categoryId=` için `Where`.
- **Gün 48:** Aynı satırlar `DbSet` üzerinde çalışacak ve SQL'e dönecek. Değişen şeyler: async sürümler (`FirstOrDefaultAsync`, `ToListAsync`) ve sonucun controller'da sabitlenmesi.
- **Gün 49:** N+1 deneyi. Sorgu sayısı, ertelenmiş çalıştırma ile ilişkili verinin ne zaman yüklendiğinden doğuyor.
- **Gün 53–54:** Arama, normalize edilmiş ayrı bir sütunda ve sunucuda sayfalama ile; bir C# metodu SQL'e çevrilemediği için.
