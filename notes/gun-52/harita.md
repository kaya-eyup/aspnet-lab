# Gün 52 — Formlar I: GET formu, arama, hata ayıklama, yönetim paneline giriş

**Dersler:** 437–443 (Bölüm 4: Forms) · **Tarih:** 20 Ekim 2026

## Bölümün yeri

- Bölüm 4'ün akışı: Form nedir → GET formu → Debug → Arama formu → Yönetim paneli → Kayıt ekleme → listeleme → güncelleme → silme → Form Models → Form Validations → File Upload.
- Hoca, daha önce HTML/CSS ile yazılan ShopApp projesine bir yönetim paneli ekliyor. Kategoriler, ürünler ve slider'lar için ekleme, listeleme, güncelleme ve silme yapılacak.
- Bugün işlenenler: formun ne olduğu, GET ile arama, adresten parametre okumak, adım adım kurulan sorgu, VS'de hata ayıklama, yönetim panelinin iskeleti ve ilk liste.
- Capstone'a taşınanlar: arama (Gün 53–54) ve doğrulama (Gün 55). Razor formları capstone'a taşınmıyor.

## 1. Form ve HTTP

- **Form**, kullanıcının girdilerini toplayıp tarayıcıya bir HTTP isteği kurduran HTML elemanı. Üç ayarı var:
  - `action`: istek nereye gidecek
  - `method`: GET mi POST mu
  - her alanın `name`'i: sunucuda okunacak anahtar (`name="q"` → `q=...`)
- **GET formu** alanları URL'ye ekler: `/kurslar?q=aspnet`.
- **POST formu** alanları gövdeye yazar: `username=ali&password=...` (`Content-Type: application/x-www-form-urlencoded`).
  - Bizim React formu da POST gövdesi gönderiyor, ama JSON olarak (`application/json`). Paketleme farklı, mantık aynı.
- **Düzeltme: gövde "gizli" değil.**
  - HTTP'de URL de gövde de açık metin olarak gider. HTTPS ikisini birlikte şifreler.
  - Parolanın URL'ye konmamasının asıl sebebi, URL'nin kaydedildiği yerler: tarayıcı geçmişi, sunucu ve proxy logları, kopyalanıp paylaşılan linkler.
- **HTML formu sadece GET ve POST bilir.**
  - `method="put"` yazılırsa tarayıcı GET gönderir.
  - Bu yüzden MVC'de güncelleme ve silme POST ile yapılır (önümüzdeki dersler).
  - API'de durum farklı: `fetch` PUT ve DELETE de gönderebilir.
- **GET veri değiştirmemeli.**
  - Silmeyi bir linke bağlamak (`<a href="/kategori/sil/5">`) tehlikeli. Tarayıcının sayfaları önceden yüklemesi, linkleri gezen botlar, hatta başka bir sitedeki gizli bir resim etiketi bu adresi senin oturumunla çağırabilir.
  - Silme her zaman POST (MVC) ya da DELETE (API) ile yapılır.
- Cevabın gövdesi MVC'de HTML, API'de JSON (slaytın alt kısmı). Aradaki boru hattı (routing, parametre doldurma, controller, EF) aynı.

## 2. Route parametresi ve query string (bugün kafa karıştıran konu)

URL: `http://localhost:5162/urunler/telefon?q=apple`

- `?`'den önceki kısım **path** (`/urunler/telefon`), sonrası **query string** (`?q=apple`).
- Route tablosu sadece path'e bakar. Query string eşleşmeye katılmaz; action seçildikten sonra parametreler doldurulurken okunur.
- Kalıp `urunler/{url?}`:
  - `urunler` sabit kelime.
  - `{url}` bir yuva; o konumdaki kelime `url` değerine yazılır.
  - `?` yuvanın boş kalabileceğini söyler.
  - Varsayılanlar `controller = "Urun", action = "List"`, yani bu kalıba uyan her istek `UrunController.List`'e gider.

| İstek | `url` | `q` | Ne gösterir |
|---|---|---|---|
| `/urunler/telefon` | `"telefon"` | `null` | telefon kategorisi |
| `/urunler?q=apple` | `null` | `"apple"` | bütün kategorilerde arama |
| `/urunler/telefon?q=apple` | `"telefon"` | `"apple"` | telefonlarda "apple" araması |

- **`?` olmasaydı** `/urunler?q=apple` ilk kalıba uymazdı, çünkü yuvanın dolu olması gerekirdi. İstek `default` kalıbına düşer. Orada `urunler` controller adı sanılır; `UrunlerController` olmadığı için 404 döner. Hocanın "url nullable olmalı" demesinin sebebi bu.
- **Parametreleri doldurmak:** ASP.NET, action'ın parametrelerini adlarıyla eşleştirerek isteğin parçalarından doldurur. Sırayla form alanlarına, route değerlerine ve query string'e bakar. `url` route'tan, `q` query string'den gelir. Hiçbir yerde karşılığı olmayan parametre boş kalır. Bu işe **model binding** deniyor.
- **Capstone'da aynısını zaten yazdın:**
  - `GET /items/kebap` → `[HttpGet("{id}")]` → route parametresi
  - `GET /items?categoryId=yemek-kulturu` → `GetAll(string? categoryId)` → query string
- **Fark, kalıbın yazıldığı yerde:**
  - Hoca kalıpları `Program.cs`'te merkezi bir tabloya yazıyor (`MapControllerRoute`). Buna **conventional routing** deniyor. Adresler bir kurala göre üretildiği için çok sayfalı MVC sitelerinde pratik.
  - Biz kalıbı her controller'ın ve action'ın üstüne yazıyoruz (`[Route("items")]`, `[HttpGet("{id}")]`). Buna **attribute routing** deniyor.
  - Web API'lerde standart attribute routing, çünkü her ucun adresi tek tek tasarlanır ve kodun hemen üstünde okunur. MVC projelerinde ikisi de yaygın.
- **`string q` mi, `string? q` mi?** Null güvenliği açıkken `?`'siz bir `string` parametre zorunlu sayılır.
  - Hocanın MVC controller'ında eksik `q` için hata ModelState'e yazılır, ama action yine çalışır. Kimse ModelState'e bakmadığı için fark edilmez.
  - Bizim `[ApiController]`'ımızda aynı durum otomatik 400 döner: `GET /comments` (itemId'siz) → 400 `errors.itemId`.
  - Gün 54'te `q` isteğe bağlı olacağı için `string? q` yazılacak.
- Formdaki `asp-controller="Urun" asp-action="List"`, formun `action` adresini route tablosundan Razor'a ürettirir; adres elle yazılmaz. Kalıp değişirse adres de kendiliğinden değişir. Bu `asp-` ile başlayan yardımcılara **tag helper** deniyor (R; capstone'da yok).

## 3. Arama: sorgu adım adım kurulur

```csharp
var query = _context.Urunler.Where(i => i.Aktif);   // veritabanına henüz bir şey gitmedi
if (!string.IsNullOrEmpty(url)) query = query.Where(i => i.Kategori.Url == url);
if (!string.IsNullOrEmpty(q))   query = query.Where(i => i.UrunAdi.ToLower().Contains(q.ToLower()));
return View(query.ToList());                         // tek SQL burada gider
```

- `query` bir sonuç değil, bir tarif.
  - Her `Where` tarife bir koşul ekler.
  - `ToList()` (bizde `ToListAsync`) tarifi tek bir SQL'e çevirip gönderir; koşullar `AND` ile birleşir.
  - Sorgunun son ana kadar çalışmamasına **deferred execution** deniyor.
- **Bizde aynısı var:** `ItemService.GetAllAsync` (`IQueryable<Item> query = _db.Items; if (...) query = query.Where(...)`). Gün 54'te aynı tarife arama koşulu, toplam sayı (`CountAsync`) ve sayfa (`Skip`/`Take`) eklenecek.
- **Tuzak:** `ToList()` araya girerse (`_context.Urunler.ToList().Where(...)`) bütün tablo belleğe iner ve süzme C#'ta yapılır. 24 satırda fark edilmez; 1 milyon satırda sunucuyu yorar.
- `Where(i => i.Kategori.Url == url)` için `Include` gerekmez; EF bunu JOIN'e çevirir. `Include` sadece ilişkili satırları cevapla birlikte geri getirmek için kullanılır.

**Hocanın arama satırı ve üç sorunu:** `i.UrunAdi.ToLower().Contains(q.ToLower())`

1. **Türkçe I:**
   - C#'ın `ToLower()`'ı bilgisayarın dil ayarına göre çalışır. Türkçe Windows'ta `"IPHONE".ToLower()` → `"ıphone"` (noktasız ı); veritabanındaki küçük harfli "iphone" ile eşleşmez.
   - `q.ToLower()` satıra değil kullanıcının kelimesine uygulandığı için EF onu SQL'e göndermeden önce C#'ta hesaplıyor. Sorun tam bu noktada çıkıyor.
   - Kod bir makinede çalışır, dili farklı bir sunucuda bozulur. Sektördeki adı **Turkish I problem**.
2. **Türkçe karakter:**
   - "cay" yazan kullanıcı "Çay"ı bulamaz, çünkü `ToLower` aksanları silmez.
   - SQL Server'da (bizde) karşılaştırma zaten büyük/küçük harfe duyarsız (KEBAP kararı); orada `ToLower` gereksiz iş.
3. **Ölçek:**
   - `Contains` SQL'de `LIKE '%apple%'` olur. Baştaki `%` yüzünden veritabanı index kullanamaz ve her satırı okur.
   - Bin satırda fark edilmez. Milyonlarca satırda, veritabanının kelimeleri ayrıca indeksleyen arama özelliği ya da ayrı bir arama servisi gerekir. Veritabanındaki bu özelliğin adı **full-text search**.

**Bizim yolumuz (Gün 53–54):**
- Frontend'deki `normalizeForSearch` (`toLocaleLowerCase("tr")` + ı→i + aksan silme) C#'a taşınacak.
- Öğenin aranabilir metni, normalize edilmiş hâliyle ayrı bir sütunda tutulacak. Kullanıcının kelimesi de aynı fonksiyondan geçecek.
- Dil ayarına bağlı hiçbir çağrı kalmayacak.
- Hocanın kodu sadece `UrunAdi`'nda arıyor; bizim sözleşmemiz ad + özet.

## 4. Hata ayıklama (Visual Studio)

- Debug, kodu durdurup o anda ne olduğunu görmek demek. Hata olmadan da kullanılır: hoca `List`'e hangi değerlerin geldiğine baktı.
- **Kısayollar:**
  - F5: hata ayıklamayla başlat ya da devam et
  - F9: satıra breakpoint koy ya da kaldır
  - F10: satırı çalıştır, metodun içine girme
  - F11: metodun içine gir
  - Shift+F11: metottan çık
  - Shift+F5: durdur
- **Değerleri görmek:**
  - Locals penceresi o andaki değişkenleri gösterir. Watch penceresi sürekli izlemek istediğin ifadeyi tutar (sağ tık → Add Watch).
  - Değişkenin üstüne fareyle gelince de değeri görünür.
- **Koşullu breakpoint:** Kırmızı noktaya sağ tık → Conditions → `q == "apple"`. Program sadece o koşulda durur; çok çağrılan bir metotta işe yarar.
- **EF sorgusunun SQL'ini çalıştırmadan görmek:** Breakpoint'te Watch'a `query.ToQueryString()` yaz. Gün 49'daki log yöntemine göre daha hızlı, çünkü sadece o sorguyu gösterir.
- **Dikkat:** Locals'ta bir `IQueryable`'ın "Results View"ını açmak sorguyu veritabanına gönderir. SQL log'unda sebebi belli olmayan bir sorgu görürsen kaynağı bu olabilir.

## 5. Yönetim paneli ve CRUD'un yeri

- Hoca hazır bir admin şablonunu `_AdminLayout.cshtml` olarak ekledi; bu, admin sayfalarının ortak çerçevesi (R).
- **`KategoriController.Index`:** `_context.Kategoriler.Include(i => i.Uruns).ToList()`
  - `Include` her kategorinin bütün ürünlerini satır satır getirir.
  - Tabloda ürünlerin sadece sayısı gösteriliyorsa bu gereksiz bir yük. `Select(k => new { k.KategoriAdi, UrunSayisi = k.Uruns.Count })` sayımı SQL'de `COUNT` ile yapar ve her kategori için tek bir sayı gelir. Bizim projection kararımızın sebebi bu.
  - `ToList()` senkron; asenkron sürüm kursta 524'te geliyor, bizde baştan `ToListAsync`.
- **CRUD controller'da mı, serviste mi?** Notundaki hüküm "yanlış yaptı". Daha isabetli hüküm: bu bir takas.
  - **Controller'da `DbContext`:** Küçük projede en kısa yol. Microsoft'un eğitimleri ve VS'nin otomatik ürettiği controller'lar da böyle. Kursta da mantıklı: her derste tek yeni şey. Hoca servisi 520'de ("Neden servise ihtiyacımız var?") getiriyor.
  - **Servisin değer kazandığı yerler:** Aynı iş birden fazla yerden çağrılınca (MVC admin + API), iş kuralları büyüyünce, iş mantığını HTTP olmadan test etmek gerekince.
  - **Bizim tutarlılığımız:** Repository katmanını "`DbSet` zaten bu işi yapıyor, bir şey kazandırmıyor" diye eklemedik. Her katman için sorulacak soru aynı: "Bu katman burada neyi kazandırıyor?" Kalıbın kendisi doğru ya da yanlış değil.
- **Capstone'da admin paneli: Blok G'de yok.**
  - Admin için giriş ve rol gerekir; Hafta 4'te sadece giriş var.
  - Razor formları capstone'a taşınmaz: capstone'un admin'i React'te olur ve API'nin PUT/DELETE uçlarını çağırır. Kurstan taşınan kısım CRUD uçları, doğrulama ve yetkilendirme.
  - Yeri Blok H. Razor formlarını yazma seviyesine çıkaracak yer, müfredattaki MVC mini projesi.

## Kurs → capstone

| Bugünkü konu | Capstone'daki karşılığı |
|---|---|
| GET formu + `?q=` | Gün 54: `GET /items?q=&page=`. Frontend'de arama zaten URL'de (`?q=`, `?page=`). |
| Route parametresi / query string | `/items/{id}` ve `?categoryId=` (var) |
| `string q` / `string? q` | Gün 54: `string? q`; `[ApiController]`'da eksik zorunlu değer 400 |
| Adım adım `Where` | `ItemService.GetAllAsync` (var); Gün 54: arama + `CountAsync` + `Skip`/`Take` |
| `ToLower().Contains` | Gün 53: `normalizeForSearch` C#'a + normalize sütun |
| `Include` ile liste | Projection (var) |
| Debug | `ToQueryString()` Gün 54'ün sorgusunda |
| Yönetim paneli / CRUD | Blok G'de yok (Blok H); servis katmanı (var) |
