# Gün 42 · Harita 1 — Dersler 396–401

> Etiketler: R 396–399 · T 400 · Ç 401. Senin haritanın düzeltilmiş ve tamamlanmış hâli. ✏️ işaretli maddeler düzeltme.

## Günün çerçevesi

Bu derslerin iki yarısı var:

- **396–399 (Razor):** Sunucunun HTML sayfasını nasıl ürettiği. Capstone'da karşılığı yok, çünkü bizim projede HTML'i React üretiyor; ASP.NET Core sadece JSON döndürüyor. Yine de iki yerde değeri var: React ile karşılaştırınca iki dünyanın mantığı netleşiyor, ve staj/ilk işte bir MVC projesiyle karşılaşacaksın.
- **400–401 (C#):** Dilin kendisi başlıyor. Bugün capstone'da yazacağın record'ların alan tipleri (`int`, `string`, listeler) tam bu konu.

---

## 396–399 · Razor (R)

### Tag Helper: `asp-controller` / `asp-action`

**Ne yaptı:** Navbar'da `<a href="/course">` yerine `<a asp-controller="Products" asp-action="List">` yazdı. Bunun çalışması için `_ViewImports.cshtml`'e `@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers` ekledi.

**Mantık:** HTML etiketine benzeyen ama sunucuda C# çalıştıran özelliklere **Tag Helper** deniyor ("HTML gibi yazılan sunucu yardımcısı"). Sunucu `asp-controller="Products" asp-action="List"`'i routing tablosuna sorar ve `href="/Products/List"`'e çevirir. Tarayıcı `asp-*`'ı hiç görmez, sadece sonuçtaki `href`'i görür.

**Neden elle yazmıyor:** ✏️ Kazanç "linke çevirmek" değil, **adresi routing tablosundan üretmek**. Route kalıbı değişirse (`/Products/List` → `/urunler/liste`), `asp-*` ile yazılan bütün linkler kendiliğinden güncellenir; elle yazılan `href="/course"` ise sessizce kırılır. Bu, dünkü `[Route("[controller]")]` sorusunun tersi: Orada içerideki bir isim değişikliği dış adresi sessizce değiştiriyordu; burada dış adres değişince iç linkler kendini düzeltiyor. Hocanın navbar'ında ikisi yan yana duruyor: `/course` elle yazılmış, `Products` üretilmiş.

**`_ViewImports.cshtml`:** Bulunduğu klasördeki ve alt klasörlerdeki bütün view'lara uygulanır. İçindeki `@using` satırları, dünkü `using` gibi sadece ad kısaltır. `@addTagHelper *, …` o kütüphanedeki bütün (`*`) tag helper'ları açar.

**Capstone'daki karşılığı:** Link yok, çünkü API HTML döndürmüyor. Ama "adresi elle yazma, routing'den ürettir" mantığı Gün 50'de geri gelecek: 201 cevabıyla birlikte yeni kaydın adresi (`Location` başlığı) aynı mekanizmayla üretilebiliyor (`CreatedAtAction`).

### Layout ve `RenderBody()`

**Ne yaptı:** Her sayfada tekrar eden üst ve alt kısmı (navbar, footer) tek bir `_Layout.cshtml`'e taşıdı. Sayfaya özel içerik `@RenderBody()`'nin bulunduğu yere yerleşiyor.

**Mantık:** Ortak çerçeveye **Layout** deniyor. React'te bunu bir layout bileşeni ve `children` (ya da React Router'da `<Outlet />`) ile çözüyordun; `RenderBody()` onların karşılığı.

**Eksikler:**

- Hangi view'ın hangi layout'u kullanacağını genelde `Views/_ViewStart.cshtml` belirler: `@{ Layout = "_Layout"; }`. Her view'a tek tek yazılmaz.
- Layout'un yeri `Views/Shared/`. MVC bir view'ı önce controller'ın kendi klasöründe arar, bulamazsa `Shared`'a bakar.
- ✏️ `_` ile başlayan ad (`_Layout`, `_ViewImports`, `_ViewStart`) bir dil kuralı değil, gelenek: "Bu dosya tek başına bir sayfa değil." MVC bunu zorlamaz, sadece okuyana söyler.

**Capstone'daki karşılığı:** Yok.

### Statik dosyalar

**Ne yaptı:** CSS, JS ve resim gibi dosyaların dışarıdan erişilebilmesi için `Program.cs`'e bir satır gerektiğini gösterdi.

**Mantık:** ✏️ İki koşulun ikisi birden gerekiyor: (1) Dosya `wwwroot` klasöründe olmalı. (2) `Program.cs`'te statik dosya istasyonu açılmış olmalı. Hoca muhtemelen `app.UseStaticFiles()` yazdı. Senin .NET 10 sandbox'ında aynı işi büyük ihtimalle `app.MapStaticAssets()` yapıyor; bu yenisi dosyaları ayrıca sıkıştırıyor ve tarayıcının önbellekte güvenle tutabileceği hâle getiriyor. Sandbox'ın `Program.cs`'inde hangisinin olduğuna bak.

**Neden varsayılan kapalı:** Proje klasörünün tamamı dışarı açık olsaydı, biri `/appsettings.json` isteyip veritabanı bağlantı bilgini okuyabilirdi. Bu yüzden varsayılan kapalı ve sadece `wwwroot` vitrin. Bu ilkenin adı **secure by default** ("güvenli olan varsayılandır, açmak bilinçli bir karardır").

**Masa kağıdıyla bağlantısı:** `/css/site.css` isteği controller'a hiç uğramaz. Bir istasyon cevabı kendisi verip bantı orada kesebilir; HttpsRedirection'ın 307'si de buydu. Bantı kesip cevabı kendisinin dönmesine **short-circuit** deniyor. Kağıda bir kural daha: *Bir istasyon cevabı kendisi verip bantı kesebilir.*

**Capstone'daki karşılığı:** Yok. API'nin `wwwroot`'u olmayacak; frontend'in dosyalarını Vercel sunuyor.

---

## Ara bölüm · React ↔ Razor

Tek temel fark: **HTML nerede ve ne zaman üretiliyor.**

- **Razor:** Her istek sunucuya gelir, C# çalışır ve tam bir HTML sayfası üretilip gönderilir. Sayfa gittikten sonra donar; değişiklik yeni bir istek demektir. HTML'i sunucunun üretmesine **server-side rendering (SSR)** deniyor.
- **React (SPA):** Sunucu bir kez boş bir HTML ve bir JS paketi gönderir. Sonra tarayıcı API'den JSON çeker ve HTML'i kendisi üretir; state değişince sayfa yenilenmeden yeniden çizer. Buna **client-side rendering (CSR)** deniyor.

Aynı iş, iki yazım (ürün listesi):

```cshtml
@* Razor: HTML'in içine C# *@
@model List<Product>
<ul>
  @foreach (var p in Model)
  {
    <li><a asp-action="Details" asp-route-id="@p.Id">@p.Name</a></li>
  }
</ul>
```

```tsx
// React: JS'in içine HTML (JSX)
function ProductList({ products }: { products: Product[] }) {
  return (
    <ul>
      {products.map((p) => (
        <li key={p.id}>
          <Link to={`/products/${p.id}`}>{p.name}</Link>
        </li>
      ))}
    </ul>
  );
}
```

**Razor'da `key` neden yok?** React sayfayı yerinde günceller ve hangi `<li>`'nin hangisi olduğunu bilmek için `key`'e ihtiyaç duyar. Razor ise hiçbir şeyi güncellemez: Sayfayı bir kez üretir, gönderir ve bir sonraki istekte baştan üretir. Güncellenecek bir şey olmadığı için kimliğe de gerek yok.

| Konu | Razor (MVC) | React (SPA) |
|---|---|---|
| Nerede çalışır | Sunucuda, C# | Tarayıcıda, JS |
| Sunucu ne döner | HTML | JSON; HTML'i tarayıcı üretir |
| Yazım yönü | HTML'in içine C# (`@foreach`, `@p.Name`) | JS'in içine HTML (`{products.map(...)}`) |
| Veri sayfaya nasıl gelir | Controller → `View(model)` → view'da `@model` + `Model` | props; veri `fetch` + state (TanStack Query) |
| Ortak çerçeve | Layout + `RenderBody()` | Layout bileşeni + `children` / `<Outlet />` |
| Tekrar eden parça | Partial view: `<partial name="_Card" model="p" />` | Component: `<Card item={p} />` |
| Link | Tag helper; adresi routing tablosundan üretir | `<Link to="...">`; adres elle yazılır |
| Kullanıcı tıklayınca | Yeni istek → yeni sayfa (sayfa yenilenir) | State değişir → yeniden çizim (sayfa yenilenmez) |
| Durum (state) nerede | Sunucuda kalmaz; her istek sıfırdan başlar | Tarayıcının belleğinde |
| Tip kontrolü | `@model List<Product>`; view'lar build'de derlenir, `@p.Nmae` build'i kırar | TS + sınırda zod |
| XSS koruması | `@` çıktıyı otomatik kaçışlar; kaçış kapısı `@Html.Raw` | `{}` otomatik kaçışlar; kaçış kapısı `dangerouslySetInnerHTML` |
| İlk açılış | HTML hazır gelir; hızlı ilk görüntü, arama motoru dostu | Önce JS inmeli ve çalışmalı |
| Güçlü olduğu yer | İçerik siteleri, formlar, admin panelleri, kurumsal iç uygulamalar | Yoğun etkileşimli uygulamalar |

**Sektör gözüyle:** İkisi rakip değil, farklı ihtiyaçlar için. Sektör bir süre her şeyi SPA'ya taşıdı, şimdi sarkaç geri dönüyor: Next.js'in sunucu bileşenleri, Blazor'ın SSR modu ve htmx gibi araçlar HTML'i yeniden sunucuda üretiyor. Türkiye'deki kurumsal projelerde MVC hâlâ çok yaygın. Bizim mimaride MVC'nin V'si React; ASP.NET Core sadece M ve C'yi yapıp JSON döndürüyor.

---

## 400 · Ne öğreniyoruz (T)

Hoca dinamik veriye geçiyor: değişkenler, veri türleri, diziler, model ve class, döngüler, koşullar, dinamik partial view'lar, route parametreleri, detay sayfası.

**Capstone eşlemesi:** Kursun önümüzdeki dersleri ile capstone'un Gün 42–43'ü aynı işi yapıyor. Tek fark: Hoca HTML döndürecek, sen JSON.

| Kursta | Capstone'da |
|---|---|
| Model / class | Bugünkü `Item` ve `Comment` record'ları |
| Diziler, döngüler | `List<T>` + LINQ (`Where`, `OrderBy`) |
| Liste sayfası | `GET /categories`, `GET /items` |
| Kategoriye göre süzme | `GET /items?categoryId=` (Gün 43) |
| Route parametresi + detay sayfası | `GET /items/{id}` (Gün 43) |

**Gün 43 için not:** API'lerde standart şu: Kaynağın **kimliği** adreste (`/items/kebap`); **süzme, sıralama ve sayfalama** query string'de (`/items?categoryId=yemek`). Hoca süzmeyi route parametresiyle yaparsa bu MVC'de görülen bir yol, ama bizim sözleşmemiz query string diyor.

---

## 401 · Değişken tanımlama (Ç)

**Ne yaptı:** `int sayi1 = 10;` kalıbını (`<tip> <ad> = <değer>;`) anlattı. Action'ın içinde iki sayı tanımladı, birini güncelledi (`sayi1 = 30`) ve toplamı hesapladı (`50`).

**Neden:** Dinamik sayfanın verisi önce C# değişkenlerinde yaşayacak; sonraki derslerde `View(model)` ile sayfaya taşınacak.

**Capstone'daki karşılığı:** Record'ların alanları (`int Order`, `string Name`, dağılım dizisi) ve controller'daki değişkenler. Bugün yazacakların.

### Senin notların: doğrular ve düzeltmeler

- ✅ **"Değişken, bellekte veri saklayan depo."**
- ✅ **"Her değişken kendi kapsamında geçerli; aynı kapsamda aynı ad iki kez tanımlanamaz."** C# burada JS'ten bir adım daha katı: İç blokta, dıştakiyle aynı adı da kullandırmaz.

  ```csharp
  int count = 1;
  if (true) { int count = 2; }   // C#: hata (CS0136). JS'te let ile serbest.
  ```

- ✏️ **"Her satırın sonunda noktalı virgül."** Her satır değil, her **ifade** (statement). `if (...)` satırı, `{` `}` ve sınıf bildirimi noktalı virgül almaz. JS'teki otomatik noktalı virgül ekleme C#'ta yok.
- ✏️ **"Tipi tanımlarken vereceksin."** Ya da derleyiciye bırakırsın: `var count = 10;` yazınca derleyici tipi `int` olarak çıkarır. Tipin bağlamdan çıkarılmasına **type inference** deniyor ve C#'ta da var. `var`, JS'teki `var` değil: Tip bir kez çıkarılır ve bir daha değişmez. Sektörde, tipin sağ taraftan belli olduğu yerlerde `var` yaygın kullanılıyor.
- ✏️ **"Kullanmayacağın boş veriyi tanımlamasan iyi olur."** Burada iki ayrı kural var:
  - Değer vermeden tanımlayabilirsin (`int count;`), ama **değer vermeden okuyamazsın**: `Console.WriteLine(count)` hata verir (CS0165). JS'teki `undefined` gibi bir ara durum yok.
  - Tanımlanıp hiç kullanılmayan değişken uyarı verir (CS0168, CS0219). Capstone'da `TreatWarningsAsErrors` açık, yani orada bu uyarı **build'i kırar**.
- ✅ **"Sonradan içini güncelleyebilirsin."** Ama tipini değiştiremezsin: `sayi1 = "otuz";` hata verir. Bir fark daha: JS'te alışkanlığın `const`; C#'ta yerel değişkenler varsayılan olarak değiştirilebilir. Sabitlik `const` (sadece derleme anında bilinen değerler, ör. `const int MaxLength = 500;`) ve `readonly` (alanlar, dünkü `static readonly` gibi) ile sağlanır.

### Slayttaki ad kuralları

- ✅ Küçük harfle başlar. ✅ Sayıyla başlayamaz. ✅ `int`, `class`, `return` gibi anahtar kelimeler ad olamaz.
- ✏️ **"Birden fazla kelime `_` ile ayrılır."** Sektör standardı bu değil. Microsoft'un .NET ad kuralları şöyle:

  | Ne | Biçim | Örnek |
  |---|---|---|
  | Yerel değişken, parametre | camelCase | `totalCount` |
  | Sınıf, record, metot, property | PascalCase | `CategoriesController`, `GetAll`, `Order` |
  | Private alan | `_` + camelCase | `_categories` |

  `toplam_fiyat` biçimi (snake_case) C#'ta kullanılmaz. Kendi kodunda bu tabloya uy; dünkü kodun zaten böyle.
- ✏️ **"Türkçe karakter kullanamayız."** Aslında derlenir (`int sayı = 1;` çalışır). Kural dilden değil, pratikten geliyor: klavye, ekip, dosya kodlaması. Bizim kuralımız zaten daha sıkı: Kod İngilizce.

### TS'ten C#'a: "blöf attırmıyor" cümlesinin doğru hâli

Sezgin doğru yönde, ama bir sınırı var. Önce doğru kısmı:

1. **Tipler çalışma anında da var.** TS'te tipler derlemeden sonra silinir; Gün 20'de gördüğün gibi `as` sadece etiketi değiştirir. C#'ta her değer tipini çalışma anında da taşır: `count.GetType()` → `System.Int32`. Dünkü routing'in projedeki controller'ları tarayıp tablo kurabilmesi (reflection) bunun sonucu. TS'te bu imkânsız, çünkü çalışma anında taranacak tip kalmıyor.
2. **Yanlış dönüşüm sessiz değil, gürültülü.** TS'te yanlış bir `as` sessizce yanlış veriyle yürür. C#'ta yanlış bir dönüşüm (`(Category)obj`) çalışma anında hata fırlatır.
3. **Şekle değil isme bakar.** TS'te aynı şekle sahip iki tip birbirinin yerine geçebilir (Gün 20, **structural typing**). C#'ta geçemez: `record A(int Id)` ile `record B(int Id)` farklı tiplerdir. Buna **nominal typing** deniyor.
4. **Tek bir `number` yok.** TS'te her sayı `number`. C#'ta `int` (tam sayı), `long` (büyük tam sayı), `double` (ondalık) ve `decimal` (para) ayrı tipler. Tuzak: `int / int` tam sayı döner, yani `7 / 2 == 3`. Capstone'da dağılımdan ortalama hesaplarken bu karşına çıkacak.

Şimdi sınır. C#'ta da iki yerde blöf yapılabilir:

- **Null güvenliği sadece derleme anında.** `string` ile `string?` arasındaki fark, TS'teki gibi derleyicinin tuttuğu bir not; çalışma anında kontrol edilmiyor. `null!` yazmak, TS'teki `!` ile aynı blöf.
- **Dış dünyadan gelen veri.** JSON'dan okunan bir record'daki `string Name` alanı, dosyada o alan yoksa sessizce null olur. Derleyici dosyanın içini göremez.

Yani "veri akışında sorunu kökünde öldürür" cümlesi kodun **içi** için doğru; sistemin **kenarları** (JSON, HTTP gövdesi, veritabanı) için değil. Frontend'de kenarı zod tutuyordu. Backend'de bugünkü kaynak sorusu ve capstone'un 4. kararı tutacak.

---

## Düzeltilen yanlış modeller (özet)

- Tag helper'ın kazancı "linke çevirmek" değil, adresi routing tablosundan üretmek.
- Statik dosya için iki koşul var: `wwwroot` + `Program.cs`'teki istasyon.
- `_` ile başlayan ad bir kural değil, gelenek.
- C#'ta da type inference var (`var`). Fark, tipin bir kez çıkarılınca sabit kalması ve çalışma anında da var olması.
- Noktalı virgül her satıra değil, her ifadeye konur.
- Çok kelimeli adlar `_` ile değil, camelCase / PascalCase ile yazılır.
- C# kodun içinde blöf attırmaz; null'da ve dışarıdan gelen veride attırır.
