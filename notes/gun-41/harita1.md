# Gün 41 · Harita 1 — Dersler 390–395: Controller, action, varsayılan route, View

> Hoca VS Code + `dotnet watch` ile anlatıyor, ben VS 2026 kullanıyorum. Farklar ilgili maddede yazıyor.
> İşaretler: 🔧 düzeltilen yanlış model · ➕ eklenen eksik · ⚠️ karşılaşabileceğin durum

---

## 0. Bugünkü derslerin yolculuk şemasındaki yeri

```
Tarayıcı ── GET /home/about ──▶ sunucu
  → Program.cs istasyonları
      UseRouting → adresi route kalıbına oturtur              ← §4 (varsayılan route)
                   home → HomeController, about → About()
  → seçilen action çalışır: HomeController.About()            ← §1–3 (controller, action)
  → dönüş: "about" (düz metin)  ya da  View() → About.cshtml → HTML   ← §5–6
◀── response ──
```

Bugünkü derslerin hepsi şemanın iki kutusunda: **"bu adres hangi action'a gidiyor?"** (routing) ve **"action ne döndürüyor?"** (controller + View).

---

## 1. Controller

**Düz mantık:** Restoranda garson. Siparişi o alır, mutfaktan ister, tabağı masaya o getirir. Yemeği kendisi pişirmez.

**Tanım:** Gelen request'i karşılayan sınıf. Gereken işi başka parçalara yaptırır ve hangi cevabın döneceğini seçer. Cevap MVC'de bir View (HTML), Web API'de JSON + status code olur.

```csharp
using Microsoft.AspNetCore.Mvc;       // Controller sınıfının adını kısaltır (§2)

namespace dotnet_basics.Controllers;  // bu dosyadaki sınıfların "soyadı" (§2)

public class HomeController : Controller   // ":" = TS'teki extends
{
}
```

- **`: Controller` kalıtımdır.** `HomeController`, `Controller` sınıfının hazır yeteneklerini miras alır: `View()`, `Ok()`, `Json()`, `Redirect()` ve gelen request'e erişim (`Request`). Bunlar olmasaydı her cevabı elle kurman gerekirdi. C#'ta `:` hem `extends` hem `implements` yerine kullanılır.
- 🔧 **"Sadece yönlendirme yapmalı"** → Yönlendirme, yani adrese bakıp hangi action'ın çalışacağını seçmek, routing'in işi (`UseRouting`). Request controller'a geldiğinde bu seçim zaten yapılmıştır. Aklındaki ilke doğru, kelime yanlış: controller bir **aracı** olmalı ve iş kuralı barındırmamalı.
  - Bu ilkenin sektördeki adı **thin controller**: veri alma, hesaplama ve kural kontrolünü servislere bırakan, kendisi sadece "servisi çağır → cevabı seç" yapan controller.
  - Capstone'da: bu akşam `CategoriesController` sabit listeyi kendi içinde tutacak. Bu bilinçli ve geçici. Gün 49'da iş bir servis sınıfına taşınacak; o gün kursta 520 numaralı "Neden servise ihtiyacımız var?" dersi var.
- ➕ **Controller nesnesi her request için yeniden oluşturulur**, cevap gidince atılır.
  - ⚠️ Bir field'da (sınıfın içinde tutulan değişken) sayaç tutarsan, sayaç her request'te sıfırdan başlar. Veri request'ler arasında controller'da yaşamaz.

**Capstone karşılığı:** Aynı yapı, iki farkla:
1. Sınıf `ControllerBase`'ten türer; `View()` gibi HTML metotları yoktur.
2. Üstünde `[ApiController]` yazar (§7).

---

## 2. namespace ve using

**Düz mantık:** Projende iki farklı klasörde `index.ts` adında iki dosya olabilir, çünkü tam yolları farklıdır: `src/app/index.ts` ve `src/shared/index.ts`. C#'ta sınıf adları için de aynı ihtiyaç var. Senin `Category` sınıfın, kullandığın bir kütüphanedeki `Category` sınıfıyla çakışmamalı.

JS'te her dosya zaten kendi kapsamıdır. C#'ta ise projedeki bütün sınıflar tek bir havuzda durur. Klasörler sadece düzen içindir, isim çakışmasını önlemez. Çakışmayı önleyen şey, sınıf adının önüne konan "soyad": **namespace**.

- `namespace dotnet_basics.Controllers;` yazınca bu dosyadaki sınıfın tam adı `dotnet_basics.Controllers.HomeController` olur.
- Kural: namespace = proje adı + klasör yolu. VS yeni dosya açınca bunu kendisi yazar.
- **Neden `dotnet_basics` (alt çizgiyle)?** Proje adı `dotnet-basics`. C# adlarında `-` kullanılamadığı için `_`'ye çevrildi. Capstone projesinin adını `TurkiyeSimulasyonu.Api` seçmemizin bir sebebi de bu.
- `namespace X;` (noktalı virgüllü) yazımı C# 10 ile geldi ve bütün dosyayı kapsar. Eski kodlarda süslü parantezli `namespace X { … }` hâlini görürsün. Anlamı aynı, sadece bir girinti seviyesi fazla.

**`using`:**
- 🔧 **"Yukarıda bir kütüphane eklendi"** → `using` kütüphane eklemez. Kütüphane zaten projede: ASP.NET Core, `.csproj`'daki `Sdk="Microsoft.NET.Sdk.Web"` satırıyla geliyor. `using Microsoft.AspNetCore.Mvc;` sadece "bu dosyada `Microsoft.AspNetCore.Mvc.Controller` yerine kısaca `Controller` yazacağım" demektir.
  - JS'teki `import` iki işi birden yapar: modülü yükler ve adı getirir. C#'ta bu iki iş ayrıdır. Yükleme `.csproj`'da olur (NuGet paketleri; npm'deki `package.json` gibi), ad kısaltma `using`'de.
- ⚠️ **`Program.cs`'nin tepesinde hiç `using` yok.** Sık kullanılan namespace'ler projeye otomatik ekleniyor; bunu `.csproj`'daki `<ImplicitUsings>enable</ImplicitUsings>` satırı sağlıyor. `Microsoft.AspNetCore.Mvc` bu listede değil, o yüzden her controller dosyasında elle yazılır.
- ⚠️ **Routing namespace'e bakmaz, sınıf adına bakar.** İki farklı namespace'te iki `HomeController` olursa `/home` isteği "birden fazla yere uyuyor" hatası verir (`AmbiguousMatchException`).

---

## 3. Action method

**Düz mantık:** Controller garsonsa, action'lar garsonun bildiği tek tek işlerdir: "menüyü getir", "hesabı getir". Her adres bir işe denk gelir.

**Tanım:** Controller'ın içindeki her `public` metot bir **action**, yani bir adrese cevap veren metottur.

```csharp
public class HomeController : Controller
{
    // localhost:5158/home/about
    public string About()
    {
        return "about";   // string dönünce cevap düz metin olur (text/plain, 200)
    }
}
```

React benzetmen yerinde: controller, aynı ön eki paylaşan bir route grubu (`/course/...`); action, o grubun altındaki tek bir route'un işini yapan fonksiyon.

### "Sihir" sorusu: adres metodu nasıl buluyor?

İki ayrı eşleşme var ve ikisi de **isim kuralına** dayanıyor.

**1. Adres → action (routing).** Uygulama açılırken (Program.cs, bir kez) ASP.NET Core projedeki sınıfları tarar:
- Adı `Controller` ile biten (ya da `Controller`'dan türeyen) `public` sınıflar controller sayılır.
- Bunların `public` metotları action sayılır.
- Sonuç bir tablodur: `home/about` → `HomeController.About()`, `course/list` → `CourseController.List()`. Controller adından `Controller` eki düşülür. Büyük/küçük harf fark etmez.

Request gelince `UseRouting` bu tabloya bakar. Sihir gibi görünen şey, programın çalışırken kendi sınıflarını ve metotlarını listeleyebilmesi; bir nevi kendine aynada bakması. Bu yeteneğe **reflection** deniyor.

**2. Action → View dosyası.** `return View();` isim verilmeden çağrılırsa dosyayı action'ın adıyla arar: `Views/{controller}/{action}.cshtml` (§5).

➕ **Döndürülen değerin içeriği eşleşmeye karışmaz.** `return "about";` yerine `return "merhaba";` yazsan adres yine `/home/about` olur. Eşleşme metodun **adıyla** yapılır, döndürdüğü şeyle değil.

**Bu yaklaşımın adı: convention over configuration.** Bir ayar dosyasına "şu adres şu metoda gitsin" diye yazmak yerine bir isim kuralına uyarsın, sistem gerisini kendisi bulur.
- Kazancı: yeni sayfa = yeni metot. Başka hiçbir yere dokunmazsın.
- Bedeli: kuralı bilmeyene sihir gibi görünür. Bir isim yanlış yazılırsa uyarı almazsın, sadece 404 görürsün.

**Kuralı ezmenin yolu, adresi metodun üstüne açıkça yazmak:**

```csharp
[Route("hakkimizda")]        // artık adres /hakkimizda; metot adı önemsiz
public IActionResult About() => View();
```

- Köşeli parantezli bu etiketlere **attribute** deniyor. TS'teki decorator'a benzer, ama kendisi kod çalıştırmaz; sistemin okuduğu bir etikettir.
- Adresi attribute ile yazmaya **attribute routing** denir. Sektörde Web API'ler neredeyse her zaman attribute routing kullanır. Capstone'da da öyle olacak (§7).

⚠️ **Her `public` metot dışarıdan çağrılabilir.** Controller'a yardımcı olsun diye `public void DeleteAll()` yazarsan, `/home/deleteall` adresi onu çalıştırır. Yardımcı metotlar `private` yazılır (ya da `[NonAction]` ile işaretlenir).

⚠️ **Hangi HTTP metodu?** Bugünkü action'larda metot belirtilmemiş, bu yüzden hem GET'e hem POST'a cevap verirler. Kısıtlamak için `[HttpGet]`, `[HttpPost]` gibi attribute'lar var. Bu akşam `GET /categories`'e `[HttpGet]` yazacağız. Yazmasaydık `POST /categories` de 200 dönerdi ve kapı kontrolündeki bozuk istek 405 vermezdi.

---

## 4. Varsayılan route ve Index

**Düz mantık:** Adres eksik gelirse boşluklar varsayılan değerlerle doldurulur. Restorana masa numarası söylemeden girersen seni giriş masasına oturturlar.

Program.cs'deki kalıp:

```
{controller=Home}/{action=Index}/{id?}
```

- `controller=Home` → adreste controller parçası yoksa `Home` kullanılır.
- `action=Index` → adreste action parçası yoksa `Index` kullanılır.
- `{id?}` → üçüncü parça isteğe bağlı (`?` bunu belirtiyor). Gelirse ve action'ın `id` adlı bir parametresi varsa değer oraya konur: `/course/details/5` → `Details(int id)`, `id = 5`.

| Adres | controller | action | Çalışan metot |
|---|---|---|---|
| `/` | (boş → Home) | (boş → Index) | `HomeController.Index()` |
| `/home` | Home | (boş → Index) | `HomeController.Index()` |
| `/course` | Course | (boş → Index) | `CourseController.Index()` |
| `/course/list` | Course | List | `CourseController.List()` |

- 🔧 **"Controller'ın default olarak döndürmesi gereken durum"** → `Index` controller'ın bir özelliği değil, route kalıbının varsayılan değeri. Kalıpta `action=Index` yazdığı için `Index` adlı metoda gidiliyor. Orayı `action=Main` yapsan varsayılan `Main` olurdu.
- ➕ **Slayt hatası:** Slaytın son satırında `/course/list` için `Course/Index` yazıyor. Doğrusu `Course/List`. Ya slaytta hata var ya da görüntü animasyonun ortasında alınmış.
- ⚠️ **`/course`, `List()`'e düşmez; `Index()` arar.** `CourseController`'da `Index` yoksa cevap 404 olur. Ekran görüntüsünde `List()`'in üstünde `// localhost:5158/course` yorumu da var; o adres List'e gitmez.

**"Onda var bende yok":** Büyük ihtimalle sürüm farkı değil, yazım farkı. Bu kalıp iki şekilde yazılabilir ve ikisi birebir aynı işi yapar:

```csharp
app.MapControllerRoute(name: "default", pattern: "{controller=Home}/{action=Index}/{id?}");
app.MapDefaultControllerRoute();   // yukarıdakinin kısa yazımı; kalıp içinde gizli
```

Senin Sandbox'ında (.NET 10 MVC şablonu) uzun yazım var, sonunda `.WithStaticAssets()` ile.

---

## 5. View ve .cshtml

**Düz mantık:** JSX'i düşün. HTML'e benziyor ama içinde kod var ve en sonunda gerçek HTML'e dönüşüyor. `.cshtml` aynı fikir; tek fark dönüşümün nerede olduğu. JSX tarayıcıda DOM'a dönüşür, `.cshtml` sunucuda HTML'e.

- 🔧 **"HTML'in .NET'teki karşılığı"** → `.cshtml` = HTML + C# (`cs` + `html`). Bu karışık yazımın adı **Razor**. Sunucu dosyayı çalıştırıp düz HTML üretir. Tarayıcı `.cshtml`'i hiç görmez, sadece sonucu görür.
- `return View();` önce `Views/Home/About.cshtml`'i arar. Bulamazsa `Views/Shared/About.cshtml`'e bakar; `Shared` bütün controller'ların ortak klasörü.
- İsim vermek de mümkün: `return View("Contact");` yazınca action'ın adı ne olursa olsun `Contact.cshtml` kullanılır.
- ⚠️ Dosya yoksa ya da adı yanlışsa şu hata gelir: *"The view 'About' was not found. The following locations were searched: …"* Mesaj aranan yerleri tek tek listeler; hatanın sebebini oradan okursun.

**Capstone'da yok:** API View döndürmez.

---

## 6. Dönüş tipi: `string` → `ActionResult`

- `public string About()` her zaman düz metin döndürür.
- `public ActionResult About()` "bir sonuç" döndürür. Sonucun hangisi olacağı çalışırken belli olur: View (HTML), JSON, 404, yönlendirme… Fikir olarak TS'teki birleşim tipine benzer: `ViewResult | JsonResult | NotFoundResult | …`. `View()`, `NotFound()`, `Ok()` gibi metotların her biri bu ailenin bir üyesini üretir.
- `IActionResult` diye bir yazım da görebilirsin; pratikte aynı amaca hizmet eder.
- **Capstone'da** tipli hâli kullanılacak: `ActionResult<List<Category>>`. Anlamı: "başarılıysa kategori listesi, değilse başka bir sonuç (404 gibi)". Başarılı cevabın şekli tipte yazdığı için hem kodu okuyan kişi hem de API belgesini üreten araçlar neyin döndüğünü görür.

---

## 7. Bu derslerden capstone'a (bu akşam) taşınanlar

| Bugünkü derste | Bu akşam API'de |
|---|---|
| `HomeController : Controller` | `CategoriesController : ControllerBase` |
| İsim kuralıyla routing (`{controller}/{action}`) | Attribute routing: sınıfa `[Route("categories")]`, metoda `[HttpGet]` |
| — | Sınıfa `[ApiController]`: API'ye özel davranışları açar. Attribute routing'i zorunlu kılar; Gün 55'teki doğrulamada da işe yarayacak |
| `return View();` | `return Ok(list);` → 200 + JSON |
| `string` / `ActionResult` | `ActionResult<List<Category>>` |

---

## 8. `dotnet watch` ve Hot Reload

- **`dotnet watch`:** Dosyayı kaydettikçe uygulamayı otomatik günceller; her değişiklikte durdur–derle–başlat yapmazsın. VS 2026'daki karşılığı **Hot Reload**: araç çubuğundaki alev simgesi. "Hot Reload on File Save" seçeneği açıksa kaydedince kendiliğinden uygulanır.
- ⚠️ **Her değişiklik canlı uygulanamaz.** Bir metodun içini değiştirmek uygulanır. `Program.cs`'deki bir değişiklik ise ancak uygulama yeniden başlayınca etki eder, çünkü `Program.cs` sadece açılışta bir kez çalışıyor (Gün 41 hatırlamanın 2. sorusu). Örneğin yeni bir middleware ekleyip yeniden başlatmazsan hiçbir şey değişmez.
- ⚠️ **Hocanın terminalindeki uyarı:** `warn: Failed to determine the https port for redirect`.
  - Sebep: `Program.cs`'de `UseHttpsRedirection` var ama uygulama sadece http'de çalışıyor. `dotnet run` ve `dotnet watch`, `launchSettings.json`'daki ilk profili (`http`) kullanır. Yönlendirilecek bir https portu olmadığı için uyarı çıkar.
  - Bu bir hata değil, sayfa yine açılır.
  - https ile başlatmak için: `dotnet watch run --launch-profile https`.
  - VS'de F5 uygulamayı `https` profiliyle açtığı için bu uyarıyı görmezsin.

---

## 9. MVC routing ve React Router

Gözlemin doğru: ikisi de "adres → onu karşılayan kod" eşlemesi yapıyor. Fark, eşlemenin nerede yapıldığında ve her geçişte ne olduğunda.

| | React Router (client-side) | MVC (server-side) |
|---|---|---|
| Eşleme nerede | Tarayıcıda | Sunucuda |
| Linke tıklayınca | Sunucuya gidilmez; bileşen değişir, sadece veri için istek atılır | Yeni request, sunucudan yeni HTML sayfası |
| Adres → kod | Route tablosu → bileşen | Route kalıbı → controller/action |
| Ekranı kuran | Tarayıcıda JS | Sunucuda Razor |

**"React'teki parça parça birleştirme daha verimli" iddiası kısmen doğru.**
- Razor'da da parça birleştirme var: ortak iskelet için layout, tekrar eden parçalar için partial view. Büyük ihtimalle sonraki R derslerinde göreceksin.
- "Verimli" ise neyi ölçtüğüne bağlı:

| Ölçüt | Sunucuda HTML (MVC) | Tarayıcıda (React SPA) |
|---|---|---|
| İlk sayfanın açılması | Hızlı; HTML hazır gelir | JS indirilip çalışana kadar boş |
| Arama motorları (SEO) | Kolay | Ek çözüm ister |
| Sayfa geçişleri, etkileşim | Her geçişte tam sayfa yenilenir | Akıcı; sayfa yenilenmez |
| Kurulum | Tek proje | İki proje (frontend + API) ve CORS |

Sektörde iki yaklaşım da yaşıyor. React dünyası son yıllarda render'ın bir kısmını sunucuya geri taşıdı bile (Next.js, React Server Components). Senin projen için React doğru seçim, çünkü oy verme ve anlık filtreleme gibi yoğun etkileşim var.

---

## 10. Kapalı kitap özeti (düzeltilmiş)

- **Ne kurdu:** Bir isteğin hangi controller'ın hangi action'ına gideceğini belirleyen routing'i ve action'ın cevap üretmesini kurdu. Cevap önce düz metindi (`string`), sonra `View()` ile `.cshtml`'den üretilen HTML oldu. Frontend'de React Router'ın tarayıcıda yaptığı eşlemeyi burada sunucu yapıyor.
- **Neden o yolu seçti:** İsim kuralı (convention over configuration). Adres ayarı yazmadan, sınıf ve metot adlarıyla eşleme yapılıyor; yeni sayfa = yeni metot. ➕ *Özetimde eksikti.*
- **Capstone'daki karşılığı:** Controller ve action aynen var. Routing attribute ile yazılacak, cevap View değil `Ok(...)` ile JSON olacak, Razor capstone'da yok. ➕ *Özetimde React'le karşılaştırdım, capstone'la değil.*
