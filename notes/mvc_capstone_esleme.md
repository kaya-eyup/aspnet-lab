
| Hocanın projesinde (tek proje, MVC) | Capstone'da | Depo |
| :--- | :--- | :--- |
| Controller action | Controller action | api |
| Model sınıfı (`Urun`) | `Models/` sınıfları (`Item`, `Comment`) | api |
| `return View(liste)` | `return Ok(liste)` -> JSON | api |
| `@model List<Urun>` (View'ın beklediği şekil) | zod şeması (`schemas.ts`) | web |
| Razor View (`.cshtml`) | React bileşeni | web |
| `@foreach` | `.map()` | web |
| *(yok: controller modeli View'a bellekte, aynı makinede verir)* | `fetch` + `VITE_API_URL` + CORS | web + api |
|---|---|---|
| `Models/Urun.cs` | `Entities/Category.cs`, `Item.cs`, `Comment.cs` | entity ayrı klasörde; `Models/` dışarı giden şekiller için kalıyor |
| `Models/DataContext.cs` | `Data/AppDbContext.cs` (`sealed`) | veriyle ilgili her şey `Data/`'da |
| `UseSqlite`, `Data Source=store.db` | `UseSqlServer`, LocalDB | canlıyla aynı motor |
| EF Core 9.x | EF Core 10.x | paket kuralı |
| `= null!` (metin alanı) | `required` | eksik alanı derleyici yakalar |
| `double Fiyat` | — (para yok) | para `decimal` olur |
| adres lambda içinde okunuyor | lambda dışında, `?? throw` | fail fast |
| many-to-many (ürün ↔ kategori) | one-to-many (öğe → kategori) | öğenin tek kategorisi var |

Kural tek cümle: **Ekrana bir şey basan her şey web'de. API ekrana hiçbir şey basmaz, JSON döndürür ve işi orada biter.** Tablonun son satırı da capstone'daki bütün ek parçaların sebebini gösteriyor: MVC'de controller ile View arasında ağ yok, capstone'da var. JSON, fetch, zod ve CORS, o ağ yüzünden var.

