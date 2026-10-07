# Gün 41 · Kaynak — `record`

**Kaynak:** [Roadmap for JavaScript and TypeScript developers learning C#](https://learn.microsoft.com/en-us/dotnet/csharp/tour-of-csharp/tips-for-javascript-developers) → "Classes" örneği → [Records](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records)

**Soru:** `record` nedir, `class`'tan farkı ne, hangi durumda tercih edilir?

**Anahtar terimler:** *value equality* · *immutable*

---

## Cevabım (düzeltilmiş)

`record`, `class` gibi bir referans tipidir. Farkı eşitliği nasıl kontrol ettiğindedir: iki nesneyi bellek adreslerine göre değil, içerdikleri verilere göre karşılaştırır (*value equality*). Kısa (positional) yazımla tanımlanınca oluşturulduktan sonra değiştirilemez (*immutable*). Bu iki özellik yüzünden, davranış barındırmayan ve sadece okunup taşınan veri şekilleri (DTO) için standart seçimdir. Bu akşam `Category` tipini bu yüzden `record` olarak yazacağız.

---

## Düzeltmeler ve eklemeler

### 🔧 Immutability `record` kelimesinden değil, kısa yazımdan geliyor

```csharp
public record Point(int X, int Y);   // kısa (positional) yazım → X ve Y sonradan değiştirilemez

public record Point2                  // record ama özellikler set'li yazılmış → değiştirilebilir
{
    public int X { get; set; }
    public int Y { get; set; }
}
```

Sayfadaki örnek kısa yazımla olduğu için "record = değişmez" gibi görünüyor. Kural şu: değişmezliği kısa yazım sağlıyor. `record` kelimesinin kendisinin garanti ettiği şey value equality.

### ⚠️ Değişmezlik sığ: içteki dizi yine değişebilir

```csharp
public record Poll(string Question, int[] Votes);

var a = new Poll("Çay mı kahve mi?", new[] { 3, 5 });
var b = new Poll("Çay mı kahve mi?", new[] { 3, 5 });

Console.WriteLine(a == b);   // False: diziler içeriğe göre değil, adrese göre karşılaştırılır
a.Votes[0] = 99;             // derlenir: Votes özelliği değişmez ama dizinin içi değişir
```

Bu iki tuzak, içinde dizi ya da liste taşıyan her record'da karşına çıkar. Value equality ve immutability sadece "bir seviye" derindir; içteki dizinin kendi kuralları geçerlidir.

### ➕ Değiştirmek istersen: `with`

```csharp
var p1 = new Point(1, 2);
var p2 = p1 with { Y = 5 };   // p1 aynen kalır; Y'si farklı yeni bir kopya oluşur
```

Bu React'teki `{ ...state, y: 5 }` alışkanlığının C# karşılığı.

### ➕ Okunur `ToString`

`Console.WriteLine(p1)` çıktısı `Point { X = 1, Y = 2 }` olur. Breakpoint'te ya da log'da nesnenin içeriğini hemen görürsün. Normal bir `class` ise sadece tip adını yazar.

### ➕ `record struct`

C# 10'dan beri `record struct` da var; bu bir değer tipi. Sadece `record` yazınca `record class` anlaşılır, yani referans tipi. Bu akşam kullanacağımız hâli bu.

### ➕ Record her yerde kullanılmaz: veritabanı sınıfları (Gün 46)

Gün 46'da veritabanı tablolarına karşılık gelen sınıflar (entity) yazılacak. Bunlar `class` olur, `record` olmaz. Sebebi: EF Core her nesneyi kimliğiyle (bellekteki hangi nesne olduğuyla) takip eder. Value equality bu takibi karıştırır, çünkü iki ayrı satırı "aynı" sayabilir. Microsoft dokümanı da record'ların EF Core entity'si olarak uygun olmadığını açıkça yazıyor.

Kısa kural:
- **Dışarı giden veri şekli (DTO)** → `record`
- **Veritabanı satırı (entity)** → `class`
