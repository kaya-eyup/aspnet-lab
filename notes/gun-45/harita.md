# Gün 45 · Dersler 410–413 (harita)

11 Ekim 2026 · Ç: 410, 412, 413 · R: 411

**Bitmiş hâl:** Kurs listesindeki her kartta bir "İncele" linki var. Linke tıklanınca o kursun detay sayfası açılıyor. Sayfanın bazı kısımları bir koşula bağlı: koşul sağlanırsa görünüyor, sağlanmazsa hiç üretilmiyor.

Capstone karşılıkları: `notes/mvc-capstone-esleme.md`

---

## 1. Koşullu gösterim: `@if`

- **Ne yaptı:** Ürünün stokta olup olmamasına göre (`stok > 0` benzeri bir koşul) farklı çıktı üretti.
- **Neden:** Razor, koşula göre HTML'in yalnızca bir dalını üretir. Seçilmeyen dal tarayıcıya hiç gitmez.
- **Capstone (web):** React'te `{koşul && …}` ya da ternary. Yorum listesindeki "Henüz yorum yok" yazısı bunun örneği (yazıldı).
- **Senior:**
  - Butonu gizlemek, kuralı uygulamak demek değil. Stok 0 iken "Sepete ekle" gizlense bile biri isteği doğrudan atabilir. Kural sunucuda da kontrol edilir (capstone: Gün 55, doğrulama).
  - "Henüz yorum yok" yalnızca istek başarıyla `[]` döndüğünde gösterilir. Yükleniyor, hata ve boş, birbirinden ayrı üç durum.

## 2. Adres kalıbı: bütün uygulama için tek kalıp (conventional routing)

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}")
    .WithStaticAssets();
```

- **Ne yaptı:** Bütün uygulama için tek bir adres kalıbı tanımladı. `=` varsayılan değer, `?` isteğe bağlı demek.

| Adres | controller | action | id |
|---|---|---|---|
| `/` | Home | Index | yok |
| `/Course` | Course | Index | yok |
| `/Course/Details/5` | Course | Details | `"5"` |

- **Capstone (api):** Bu kalıp capstone'da kullanılamaz. Orada her controller kendi adresini attribute ile yazıyor (`[Route("items")]`, `[HttpGet("{id}")]`) ve `Program.cs`'te `MapControllers()` var. Buna attribute routing deniyor. `[ApiController]` attribute routing'i zorunlu kılar; kalıptan gelen bir adres `[ApiController]`'lı bir controller'a ulaşamaz.

## 3. Detay action'ı: adresten metoda

```csharp
public ActionResult Details(int id)
{
    Course? kurs = kurslar.Where(a => a.Id == id).FirstOrDefault();
    return View(kurs);
}
```

- **Ne yaptı:** Adresteki id'ye göre kursu buldu ve aynı View'a her seferinde farklı bir model verdi.
- **id'nin yolculuğu:** Routing adresten `id = "5"` değerini metin olarak çıkarır. Model binding bu metni `int id = 5` yapar.
- **Hocanın kodundaki hata:** `FirstOrDefault()` kayıt bulamazsa `null` döner. Bu değer kontrol edilmeden View'a gider. Kartta `@Model.Title` çalışınca exception oluşur ve sonuç 500 olur. Bu iki adreste yaşanır:
  - `/Course/Details/999`: böyle bir kurs yok.
  - `/Course/Details`: id isteğe bağlı, ama `int` null olamadığı için 0 gelir (anlatılan, ölçülmedi).
  - Hocanın yazdığı `Course?` zaten "bu değer null olabilir" diyor; bir sonraki satır bunu dikkate almıyor.
- **Capstone (api):** `GET /items/{id}` içinde `if (item is null) return NotFound();` var, sonuç 404 (Gün 43).
- **Kısa yazım:** `kurslar.FirstOrDefault(a => a.Id == id)` aynı işi yapar.

## 4. Kart parçası ve link üreten özellikler: `_CourseCard`

```cshtml
@model Course
...
<a asp-controller="Course" asp-action="Details" asp-route-id="@Model.Id" ...>İncele</a>
```

- **Ne yaptı:** Kartı ayrı bir dosyaya aldı. Dosya adının başındaki `_`, bunun tek başına bir sayfa olmadığını, başka bir View'ın içine konan bir parça olduğunu gösterir. Buna partial view deniyor. Kart model olarak **bütün bir `Course` nesnesi** alıyor. Listeyi basan View her kurs için kartı ayrı ayrı çağırıyor.
- **Link:** `asp-*` özellikleri, HTML etiketine eklenen ve sunucuda çalışıp çıktıyı üreten özelliklerdir; bunlara tag helper deniyor. Burada aynı adres kalıbını ters yönde kullanıp `href="/Course/Details/5"` üretiyorlar. Yani routing iki yönlü çalışıyor: gelen istekte adresi okur, linkte adresi yazar.
- **ID'nin tam yolculuğu:**

```
Liste isteği → controller listeyi View'a verir → her kart kendi Model.Id'sini href'e yazar
→ kullanıcı tıklar → YENİ istek: /Course/Details/5
→ Routing: id = "5" → model binding: int id = 5 → Details(5) → View(kurs)
```

- **Neden tag helper:** `href="/Course/Details/@Model.Id"` da çalışırdı. Ama kalıp değişirse elle yazılan linkler hiçbir hata vermeden bozulur. Tag helper'ın ürettikleri ise kendiliğinden güncellenir.
- **Neden ayrı parça:** Aynı kart HTML'i her listede tekrar tekrar yazılmasın, bir değişiklik tek yerden yapılsın diye.
- **Capstone:** MVC'de tek bir adres hem sayfayı hem veriyi getiriyor. Capstone'da iş iki adrese, iki routing'e bölünüyor:
  - **web:** öğe kartı bileşeni (props) + `<Link to={...}>`, öğe sayfasının React Router yolu
  - **api:** sayfanın veri istediği `GET /items/{id}`

## Düzeltilen yanlış modeller

- **"CourseCard'a ID'yi dışarıdan aldırdı"** → Kart bütün bir `Course` alır ve Id'sini linke **yazar**. ID adrese oradan girer; yeni istekte routing onu geri **okur**.
- **"Detay sayfası = capstone'da öğe döndürme"** → Kısmen doğru. Veri kısmı api'deki `GET /items/{id}`; sayfa kısmı web'deki React Router yolu.
- **"Hoca koşula göre çıktıyı manipüle etti"** → Koşula göre HTML'in yalnızca bir dalını üretti.
