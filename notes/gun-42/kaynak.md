# Gün 42 · Kaynak — JSON okurken null ve eksik alan

> **Kaynak:** Microsoft Learn
> - [Respect nullable annotations](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/nullable-annotations)
> - [Required properties → Non-optional constructor parameters](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/required-properties#non-optional-constructor-parameters)
>
> **Yol:** "Roadmap for JS/TS developers learning C#" → "Nullable and non-nullable types" maddesi → System.Text.Json tarafı.

## Soru

> *When System.Text.Json deserializes into a record with a non-nullable `string` property, what happens if the JSON has that property as `null`, and what happens if the property is missing entirely? Which setting covers each case?*

## Kısa cevap

- **Alan `null` gelirse** (`{"Name": null}`): Varsayılan ayarlarla sessizce `null` atanır. `RespectNullableAnnotations` açıksa hata fırlatılır.
- **Alan hiç yoksa** (`{}`): Varsayılan ayarlarla sessizce `null` atanır. `RespectNullableAnnotations` açık olsa bile hata çıkmaz. Bu durumu `RespectRequiredConstructorParameters` yakalar.
- **İki ayar da varsayılan olarak kapalı.** İkisi de .NET 9'da isteğe bağlı olarak geldi; doküman yeni projelerde açılmalarını öneriyor.

---

## Bağlam: Neden böyle bir ayara ihtiyaç var?

C#'ta tipler çalışma anında da var; yanlış bir dönüşüm sessizce geçmez, hata fırlatır. Ama **null güvenliği bunun istisnası.** `string` ile `string?` arasındaki fark, TS'teki gibi sadece derleyicinin tuttuğu bir not; çalışma anında kimse kontrol etmiyor.

JSON ise çalışma anında, dışarıdan (dosyadan, HTTP isteğinden) gelir. Derleyici dosyanın içini göremez. `string Name` yazdığın bir alana JSON'dan null gelirse, derleyici "burası asla null olmaz" diye güvenmeye devam eder; hata ancak o alan kullanıldığı yerde (`Name.Length`) `NullReferenceException` olarak patlar. Bu, TS'te `as Item` yazmanın C#'taki karşılığı: Etiket doğru, içerik kontrol edilmemiş.

Frontend'de bu boşluğu zod kapatıyordu. Backend'de JSON okuyucusunun iki ayarı kapatacak.

## İki ayrı soru: "Olmak zorunda mı?" ve "Null olabilir mi?"

C#, bir alan hakkında iki ayrı soru sorar:

1. **Alan JSON'da olmak zorunda mı?** Buna **required** deniyor.
2. **Alanın değeri null olabilir mi?** Buna **nullable** deniyor.

Bu iki soru birbirinden bağımsız. TS'te de aynı ayrım vardı:

| TS | Anlamı |
|---|---|
| `name?: string` | Alan hiç olmayabilir (`undefined`) |
| `name: string \| null` | Alan olmak zorunda, değeri null olabilir |

Fark şu: JS'te "alan yok" (`undefined`) ile "alan var ama boş" (`null`) iki ayrı değer. C#'ta `undefined` yok; çalışma anında ikisi de tek bir `null`'a iner. Bu yüzden iki durumu ayırmak için iki ayrı ayar gerekiyor.

## Ayar 1: `RespectNullableAnnotations`

**Ne yapar:** JSON'da açıkça `null` yazan bir değeri, null olmaması gereken (`?` almamış) bir alana atamayı reddeder.

```csharp
using System.Text.Json;

// Kısa yazımlı (positional) record: Name aynı zamanda constructor parametresi
record Person(string Name);

var options = new JsonSerializerOptions
{
    RespectNullableAnnotations = true
};

JsonSerializer.Deserialize<Person>("""{"Name":null}""", options); // JsonException: Name null olamaz
JsonSerializer.Deserialize<Person>("""{}""", options);            // Hata YOK, Name = null
```

**Yakalamadığı şey:** Eksik alan. `{}` geldiğinde alan "verilmemiş" sayılır ve hata çıkmaz.

## Ayar 2: `RespectRequiredConstructorParameters`

**Ne yapar:** Varsayılan değeri olmayan constructor parametrelerini zorunlu sayar. JSON'da karşılığı yoksa hata fırlatır. Kısa yazımlı record'larda alanlar zaten constructor parametresi olduğu için bizim record'larda doğrudan işe yarıyor.

```csharp
// Age'in varsayılan değeri var → opsiyonel. Name'in yok → zorunlu.
record Person(string Name, int? Age = null);

var options = new JsonSerializerOptions
{
    RespectRequiredConstructorParameters = true
};

JsonSerializer.Deserialize<Person>("""{"Age":42}""", options);     // JsonException: Name eksik
JsonSerializer.Deserialize<Person>("""{"Name":"Ayşe"}""", options); // Geçer, Age = null
```

**.NET 9'dan önce:** Bütün constructor parametreleri opsiyonel sayılıyordu. `{}` okununca `Name = null`, `Age = 0` oluyordu; hiçbir hata çıkmadan.

## İkisi birlikte

```csharp
var options = new JsonSerializerOptions
{
    RespectNullableAnnotations = true,
    RespectRequiredConstructorParameters = true
};
```

`record Person(string Name)` için:

| JSON'da | Sonuç |
|---|---|
| `"Name": "Ayşe"` | ✅ Geçer |
| `"Name": null` | ❌ `RespectNullableAnnotations` yakalar |
| Alan hiç yok | ❌ `RespectRequiredConstructorParameters` yakalar |

## Nasıl açılır

**1. Okuma yapılan yerde, options nesnesiyle** (yukarıdaki örnekler gibi). Sadece o okuma için geçerli; kural kodda, riskin olduğu yerde görünür.

**2. Proje genelinde, `.csproj` ile.** Uygulamadaki bütün JSON okumaları için varsayılanı değiştirir:

```xml
<ItemGroup>
  <RuntimeHostConfigurationOption Include="System.Text.Json.Serialization.RespectNullableAnnotationsDefault" Value="true" />
  <RuntimeHostConfigurationOption Include="System.Text.Json.Serialization.RespectRequiredConstructorParametersDefault" Value="true" />
</ItemGroup>
```

Bu tür proje genelindeki açma/kapama ayarlarına **feature switch** deniyor.

**Neden varsayılan kapalı:** .NET 9'dan önce yazılmış uygulamalar bozulmasın diye. Bugün açılsaydı, eksik alanlarla sorunsuz çalışan eski projeler bir anda hata fırlatmaya başlardı.

## zod → C# çeviri tablosu

İki ayar da açıkken:

| zod | C# record parametresi | Anlamı |
|---|---|---|
| `z.string()` | `string Name` | Olmak zorunda, null olamaz |
| `z.string().nullable()` | `string? Name` | Olmak zorunda, null olabilir |
| `z.string().optional()` | `string? Name = null` | Hiç olmayabilir |

Kural: **`?` "null olabilir" demek, `= varsayılan` "olmayabilir" demek.**

## Sınırlar: kontrolün göremediği yerler

`RespectNullableAnnotations` sadece "dış katmana" bakar. Şunları kontrol etmez:

- **Koleksiyonların elemanları.** Çalışma anında `List<string>` ile `List<string?>` ayırt edilemez, yani `["a", null]` geçer. Dünkü record eşitliğindeki "sığ" desenin aynısı: Kontrol dış katmanda kalıyor.
- **Generic alanlar.** Örneğin `record Box<T>(T Value)` içindeki `Value`.
- **En dıştaki tip.** JSON'un tamamı `null` ise `Deserialize<Person>` hata fırlatmaz, `null` döndürür. Bu yüzden `JsonSerializer.Deserialize<T>`'nin dönüş tipi `T?`; derleyici sonucu null kontrolü yapmadan kullanmana izin vermez.

**Bizim projedeki karşılığı:**
- `distribution` bir `int` dizisi. `int`'e null yazılamadığı için JSON'da null eleman olursa okuma zaten hata verir; burada risk yok.
- Kategori, öğe ve yorum listelerinde `[null]` gibi bir eleman geçebilir. Bizim `db.json`'da böyle bir durum yok, ama kontrolün sınırının burası olduğunu bil.
- Dosyanın tamamı null dönebileceği için okuyucuda bunu ele almak gerekecek. Capstone'da `TreatWarningsAsErrors` açık olduğu için, ele almazsan build zaten geçmeyecek.

## Zorunluluğun diğer yolları (tanıma seviyesi)

Kısa yazımlı record'da zorunluluk constructor parametresinden gelir. Property'leri ayrı yazılan bir `class`'ta ise:

- **`required` anahtar kelimesi:** `public required string Name { get; set; }`. C#'ta olağan yol.
- **`[JsonRequired]` attribute'u:** Zorunluluk sadece JSON okurken geçerli olsun istendiğinde.

Gün 46'da entity'ler `class` olarak yazılacak; bu ikisini orada tekrar göreceksin.

---

## Benim cevabım

> `record Person(string Name)` örneğinde JSON `{"Name": null}` olarak gelirse, `RespectNullableAnnotations` ayarı tip güvenliğini koruyarak hata fırlatır. Ancak JS'teki `undefined` mantığına benzer şekilde alan JSON'da hiç yoksa (`{}`), sistem bunu eksik (opsiyonel) veri sayar ve hata vermeden `null` atar. Bu eksik veri durumunu engellemek ve constructor parametrelerini zorunlu kılmak için `RespectRequiredConstructorParameters` ayarı kullanılmalıdır.

**Değerlendirme:** Üç tespit de doğru. Eklenenler:

- İki ayar da **varsayılan olarak kapalı**; record'u `string Name` diye yazmak tek başına hiçbir şeyi korumaz.
- Eksik alanın hata vermemesinin sebebi sadece C#'ta `undefined` olmaması değil. C# "olmak zorunda mı?" ile "null olabilir mi?" sorularını ayrı tutuyor; TS'teki `name?:` ile `| null` ayrımının aynısı. Çalışma anında ikisinin değeri tek bir `null`'a indiği için iki ayrı ayar gerekiyor.

## Capstone bağlantısı

Bugün `db.json`'u okuyan `SeedDataReader`'da iki ayar da açık olacak. Bir öğede zorunlu bir alan eksikse ya da null olmaması gereken bir yerde null varsa, sunucu daha açılırken hata verecek (fail fast). Ayarların okuyucunun options nesnesinde mi yoksa `.csproj`'da mı açılacağı capstone'da kararlaştırılacak. Gün 50'de `POST /comments` isteğinin gövdesi okunurken aynı soru tekrar gelecek.
