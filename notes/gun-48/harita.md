# Gün 48 — Harita (431–435)

## Günün büyük resmi

Hoca üç şey yaptı:
1. Kategoriyi ve ürünle ilişkisini kurdu.
2. Her sayfada görünen menüyü ve slider'ı controller'dan bağımsız hâle getirdi.
3. Kategoriye göre ürün listeleyen ilk adresi yazdı.

Capstone'a en doğrudan değen üçüncüsü. Bugünkü capstone Adım 2'nin aynısı.

## 1. Kategori entity'si ve ilişki

- **Ne yaptı:**
  - `Kategori`: `Id`, `KategoriAdi`, `Url`, `List<Urun> Uruns`.
  - `Urun`'a `KategoriId` ve `Kategori` navigation eklendi.
  - Bir kategoride çok ürün var (1-N).
- **Neden:** Ürün tek bir kategoriye ait. Bağ sayıya (`KategoriId`) kurulu; navigation, C# tarafında karşı tarafa geçmeyi sağlıyor.
- **Capstone:**
  - `Category` ↔ `Item` aynı 1-N ilişki (Gün 46).
  - `Kategori.Url` = `Category.Slug`, yani adreste görünen ad.
- **Ekrandaki iki ayrıntı (senior gözüyle):**
  - `public Kategori Kategori { get; set; } = new();` EF dokümanı reference navigation'ın dolu bir değerle başlatılmamasını söylüyor, çünkü bu "olmayan bir kaydın var olduğunu iddia etmek" demek.
    - Ürünü kategorisi yüklenmeden okursan `null` değil, boş bir kategori görürsün. Hata gizlenir.
    - Yeni ürün eklerken bu boş nesne EF'e yeni bir kategori gibi görünebilir.
    - Capstone'daki `= null!` doğru olan.
    - Collection'ı (`= new()` / `= []`) başlatmak ise doğru: Boş liste "hiç ürün yok" demek, bu gerçek bir durum.
  - `double Fiyat`: Para `decimal` ile tutulur, `double`'da `0.1 + 0.2` tam `0.3` etmez. Capstone'da para yok, ama stajdaki e-ticaret projelerinde karşına çıkacak.

## 2. ViewComponent: menü ve slider

- **Ne yaptı:**
  - Önce `Index` action'ında `ViewData["Kategoriler"] = _context.Kategoriler.ToList()` yazdı ve listeyi `_Menu` partial'ına verdi.
  - Sonra menüyü, kendi verisini kendisi çeken bir ViewComponent'e çevirdi.
  - Slider'ı da aynı şekilde yaptı.
- **Neden:**
  - Menü her sayfada var. ViewData yolunda her sayfanın action'ı aynı satırı tekrarlamak zorunda.
  - ViewData tipsiz bir sözlük (TS'teki `Record<string, unknown>` gibi); veriyi alan taraf tipi kendisi dönüştürmek zorunda.
- **Dikkat (yarım model):** ViewComponent kodun tekrarını kaldırır, işin tekrarını kaldırmaz. Her sayfa çiziminde sorgusu yeniden çalışır; kendi başına bir önbelleği yok.
- **Slider:** Veri `Slider` tablosundan geliyor (`HasData` ile 3 kayıt). Bu sayede sonradan yönetim panelinden, kod değiştirmeden değiştirilebilir.
- **Capstone karşılığı:**
  - API'de yok, çünkü API HTML üretmiyor.
  - Frontend'deki karşılığı, kendi verisini kendisi çeken React bileşeni: `CategoryLink.tsx` (`useQuery(categoryQueries.detail(categoryId))`).
  - Fark: TanStack Query, aynı kategoriyi isteyen altı kart için tek istek atar (önbellek). ViewComponent her çizimde sorgular.
  - Capstone'un `NavBar`'ında kategori menüsü yok: logo, arama, oylarım, Hakkında, tema.

## 3. Kategoriye göre ürün listeleme (anlaşılmayan kısım)

**Mantık:** Adres `/urunler/telefon`. `Program.cs`'teki kural şunu söylüyor: "`urunler/` ile başlayan adresi `UrunController`'ın `List` metoduna gönder, sonraki parçayı `url` adıyla ver."
- ASP.NET, `List(string url)` parametresini adından eşleştirir ve içine `"telefon"` koyar.
- `Where(i => i.Aktif && i.Kategori.Url == url)`: Ürünün kategorisine geçer, onun slug'ını karşılaştırır. SQL'de bu bir JOIN.
- `url` adı kafa karıştırıcı. Tam adres değil, kategorinin slug alanı; hoca slug sütununa `Url` adını vermiş.

**Terimler:**
- Kuralı `Program.cs`'te merkezi olarak tanımlamaya **conventional routing** deniyor.
- Capstone'da kural controller'ın üstünde yazılı (`[Route("items")]`, `[HttpGet("{id}")]`). Buna **attribute routing** deniyor.
- Eşleştirme mekanizması aynı, sadece kuralın yazıldığı yer farklı. Web API'lerde standart attribute routing. MVC sitelerinde sayfa adresleri için conventional sık görülür.

**Capstone karşılığı (iki parça):**
- Sayfa adresi: Hocanın `/urunler/telefon`'u ↔ React'teki `kategori/:slug` route'u (`src/app/router.tsx`).
- Arkasındaki sorgu: ↔ `GET /items?categoryId=yemek-kulturu`. Bugünkü Adım 2 bu: `Where(i => i.Category.Slug == categoryId)`.
- MVC'de sayfa ve veri aynı sunucuda. Capstone'da sayfa adresi React'te, veri adresi API'de.

**Capstone'da neden yolda değil, query'de:** REST alışkanlığı şöyle. Tek bir kaydın kimliği yolda durur (`/items/kebap`). Bir listeyi süzmek query'de yapılır (`/items?categoryId=`).

## 4. Ekranda geçen diğer ayrıntılar

| Hocada | Capstone'da |
|---|---|
| `ToList()` (senkron) | `ToListAsync(cancellationToken)` (bugün) |
| Entity doğrudan view'a gidiyor (`@model List<Urun>`) | DTO (bugün) |
| `Where(urun => urun.Aktif && urun.Anasayfa)`: anasayfadaki ürünleri editörün işaretlediği bir bayrak belirliyor | Haftanın Seçilmişleri hesaplanıyor (Gün 56). Bayrakta kararı insan verir, hesapta veri. Capstone'da yönetim paneli yok ve liste her hafta kendiliğinden değişmeli, bu yüzden hesap. park [gun47] maddesinin yarısı bu. |
| `HasData` ile sabit Id'li 3 slider | Küçük, değişmeyen, anahtarı sabit veri için doğru araç. Capstone'un verisine uymuyor (Gün 47'deki üç gerekçe). |
| `base.OnModelCreating(modelBuilder)` | Düz `DbContext`'te bir şey yapmıyor. Gün 58'de `IdentityDbContext`'e geçince zorunlu olacak, çünkü Identity tablolarını base kuruyor. |
