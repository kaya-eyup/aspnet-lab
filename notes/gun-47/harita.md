# Gün 47 · Kurs haritası (423–430)

**Bugünün iki konusu:**
1. Veritabanının yapısını kurmak (migration).
2. Kurulan tablolara ilk veriyi koymak (seed).

Ardından hoca ilk kez veritabanından okuyan bir controller yazdı ve listeyi süzdü.

Hoca `dotnet-ef` aracını global olarak güncelledi. Sende de global kurulu, sürümü 10.0.12.

## 1. Migration nedir? (Ç)

**Sorun:** Entity sınıfların zamanla değişir: bir sütun eklenir, yeni bir tablo eklenir. Veritabanı bu değişikliği kendiliğinden öğrenemez. Her değişiklik için elle SQL yazmak ve onu her veritabanında (senin LocalDB'n, ileride Azure) aynı sırayla çalıştırmak hataya açık bir iş.

**Çözüm:** Her model değişikliği bir dosya olarak kaydedilir. Dosya şunu söyler: "Veritabanını bir önceki hâlinden bu hâline getirmek için şunları yap." Bu dosyalar sıralıdır, depoda durur ve her veritabanına aynı sırayla uygulanır. Bu dosyaların her birine **migration** deniyor. Git benzetmesi: commit kodun bir adımıdır, migration veritabanı yapısının bir adımıdır.

**Düzeltme:** Tanımın, migration'ın "entity şemalarını veritabanına haber etmek" olduğunu söylüyordu. Haber etmek, migration'ın kendisinin değil `database update`'in işi. Bunlar iki ayrı adım:
1. **Üret:** `migrations add`. Değişikliği bir dosyaya yazar, veritabanına dokunmaz.
2. **Uygula:** `database update`. Dosyalardaki adımları veritabanında çalıştırır.

**Slaytı okumak:**
- **Soldaki Migration 1-2-3:** Projendeki dosyalar. Her biri SQL'e çevrilebilen bir adım (veritabanını oluştur, sütun ekle, tablo ekle). Slayttaki "SQL Scripts" etiketi bunu anlatıyor.
- **Sağdaki liste:** Veritabanının içinde tutulan bir kayıt tablosu: `__EFMigrationsHistory`. Hangi adımların bu veritabanına uygulandığını yazar.
- **`database update` ne yapar:** İki listeyi karşılaştırır. Sadece veritabanında henüz olmayanları uygular, sırayla. Bunlara bekleyen, yani **pending** migration deniyor. Bu yüzden aynı komutu iki kez çalıştırmak zarar vermez; ikinci seferde uygulanacak bir şey kalmamıştır.

## 2. `migrations add` ne üretir? (Ç)

Dosyalar "kendi kendine" oluşmuyor. EF, modelin şimdiki hâlini son migration'da kaydettiği hâliyle karşılaştırıyor ve aradaki farkı C# kodu olarak yazıyor.

| Dosya | İçinde ne var | Kim kullanır |
|---|---|---|
| `2026…_InitialCreate.cs` | `Up`: ileri gitme adımları (`CreateTable`…). `Down`: aynı adımların geri alınması (`DropTable`…) | `database update` |
| `2026…_InitialCreate.Designer.cs` | O migration anındaki modelin kaydı | EF |
| `AppDbContextModelSnapshot.cs` | Modelin en son hâlinin fotoğrafı. Buna **snapshot** deniyor | Bir sonraki `migrations add` farkı buna göre çıkarır |

İlk migration'da karşılaştırılacak bir fotoğraf yok. Bu yüzden her şey yeni sayılır ve bütün tablolar `CREATE` olarak yazılır.

## 3. Komutlar ve kurallar (K + Ç)

| Komut | Ne yapar | Veritabanına dokunur mu? |
|---|---|---|
| `dotnet tool update --global dotnet-ef` | `dotnet ef` komutunu kurar ya da günceller | hayır |
| `dotnet add package Microsoft.EntityFrameworkCore.Design` | Aracın projenle konuşmasını sağlayan paketi ekler | hayır |
| `dotnet build` | Projeyi derler | hayır |
| `dotnet ef migrations add <Ad>` | Migration dosyalarını üretir | hayır |
| `dotnet ef migrations script` | Migration'ların SQL hâlini ekrana yazar | hayır |
| `dotnet ef migrations remove` | Son migration'ı dosyalarıyla ve snapshot'la birlikte geri alır. Migration uygulanmışsa durur | sadece kontrol için |
| `dotnet ef database update` | Bekleyen migration'ları uygular. Veritabanı yoksa önce onu oluşturur | evet |
| `dotnet ef database update <Ad>` | Veritabanını o migration'a kadar geri götürür (`Down`'ları çalıştırır) | evet |

**Neden hem araç hem paket gerekiyor?** Araç sadece bir başlatıcı. Önce projeni derler, sonra `Program.cs`'yi `Build()` satırına kadar çalıştırır ve `AppDbContext`'i servis kutusundan alır. Modeli okuyup kod üretme işini ise projenin içindeki Design paketi yapar. csproj'da bu paketin altındaki `PrivateAssets` / `IncludeAssets` satırları iki şey söylüyor: senin kodun bu paketi kullanamaz ve paket, bu projeyi kullanan başka projelere geçmez.

**Neden migration'dan önce `dotnet build`?** `migrations add` de projeyi derler. Ama derleme hata verirse sadece "Build failed." yazar, hatanın kendisini göstermez. Hocanın önerisi doğru: önce derle, hata varsa doğru yerde görürsün.

**Ortam:** `dotnet ef` çalıştırılırken ortam belirtilmemişse araç Development'ı varsayar (EF kaynak kodunda doğrulandı). Bağlantı adresi bu yüzden `appsettings.Development.json`'dan okunur ([gun46] ortam maddesi).

**Sektörde uyulan kurallar:**
- Başka bir veritabanına uygulanmış bir migration düzenlenmez ve silinmez. Düzeltme gerekiyorsa bu yeni bir migration'dır. Git'te push edilmiş bir commit'i yeniden yazmamak gibi.
- Migration dosyası elle silinmez, `migrations remove` ile geri alınır. Elle silersen snapshot geride kalır ve sonraki migration yanlış farkla üretilir.
- EF'in ürettiği dosyalar elle biçimlendirilmez, olduğu gibi commit'lenir.
- Migration'ın adı yapılan değişikliği anlatır: `InitialCreate`, `AddSearchColumn`. Türkçe klavyeye dikkat: notunda `InıtialCreate` yazıyor, doğrusu `InitialCreate`.
- Canlı veritabanı: v1'de kendi bilgisayarından `database update` ile uygulanacak (Gün 52). Kurumsal şirketlerde migration'lar çoğunlukla SQL script'e çevrilip veritabanı yöneticisine verilir.

**Capstone'daki karşılığı:** Bugünkü Adım 1–3.

## 4. Veritabanının içine bakmak (K)

- **Hoca:** SQLite bütün veritabanını tek bir dosyada tutar (`store.db`). O dosyayı açmak için ayrı bir program indirdi.
- **Capstone:** LocalDB bir dosya değil, bilgisayarında çalışan bir sunucu. İçine Visual Studio'nun SQL Server Object Explorer penceresinden bakılır, ayrıca bir şey indirmen gerekmez. Orada `__EFMigrationsHistory` tablosunu da göreceksin.

## 5. Seed: `HasData` (Ç)

Tablolar kurulduğunda boştur. Uygulamanın anlamlı açılması için içinde bazı verilerin hazır olması gerekir. Bu ilk veriye **seed** deniyor (ilk ekilen tohum).

- **Ne yaptı:** `modelBuilder.Entity<Urun>().HasData(...)` ile 6 ürün tanımladı; `Id`'leri elle verdi.
- **Düzeltme:** `OnModelCreating`'i yeni oluşturmadı. DbContext'te zaten var olan bu metodu **override** etti, yani kendi sürümüyle değiştirdi. Senin `AppDbContext`'inde uzunluk ve ilişki kurallarını yazdığın metot da bu.
- **Nasıl çalışır:** Veri modelin bir parçası olur ve migration dosyasının içine `InsertData` olarak yazılır. Veriyi değiştirirsen yeni bir migration üretmen gerekir. `Id`'yi elle vermek zorunludur, çünkü EF bir satırı bir migration'dan diğerine `Id`'sinden tanır.
- **Capstone'da neden kullanılmıyor:** Kaynak notunda. Kısaca: bizde numaraları veritabanı veriyor ve veri ~1000 öğeye büyüyecek.
- **`base.OnModelCreating(modelBuilder)`:** Düz bir DbContext'te bu satır boş bir metodu çağırır; silsen fark etmez. Gün 58'de Identity'nin context'ine geçince zorunlu olur, çünkü Identity kullanıcı ve rol tablolarını tam o satırda tanımlıyor.

## 6. Veritabanından okuyan controller (Ç)

- **Ne yaptı:** `UrunController` constructor'dan `DataContext` aldı ve `_context.Urunler` üzerinden ürünleri listeledi.
- **Neden:** Dün `AddDbContext` ile kaydedilen tarifi ilk kullanan yer burası. İstek gelir → servis kutusu bir context oluşturur → controller'a verir → istek bitince context atılır.
- **Capstone'daki karşılığı:** Yarın (Gün 48) controller'lar `SeedData` yerine `AppDbContext` alacak. Hoca sorguları senkron yazıyor (`ToList()`). Capstone'da `ToListAsync()` + `await` kullanılacak (müfredat kararı).

## 7. Süzme: `.Where(u => u.Aktif)` (Ç)

- **Ne yaptı:** `Aktif` alanı `true` olan ürünleri listeledi.
- **Mantık:** Bu `Where` C#'ta çalışmıyor. EF onu SQL'e çevirip veritabanına gönderiyor: `WHERE Aktif = 1`. Süzme veritabanında yapılır; bütün tablo C#'a çekilip orada süzülmez.
- **❌ Düzeltme, capstone eşlemesi:** Haftanın Seçilmişleri bir açık/kapalı alanına bakılarak seçilmiyor, hesaplanıyor: son 7 günde en çok yorum alan 4 öğe (Gün 56). Bir editörün elle işaretlediği bir alan yok; sıralama yorum sayısından çıkıyor. Bu hesap SQL'de gruplama (`GROUP BY`) gerektirir, tek bir `Where` yetmez.
- **Doğru eşleme:** `GET /items?categoryId=`. Bugün bellekteki listede çalışan `Where`, yarın aynı yazımla SQL'e çevrilecek. Kod aynı kalıyor, çalıştığı yer değişiyor.

## 8. `mvc_capstone_esleme.md`'ye eklenecek satırlar

| Hocanın projesi | Capstone | Neden farklı |
|---|---|---|
| `dotnet-ef` global | `dotnet-ef` global (10.0.12) | Aynı. Sürümü depoya bağlamak, otomatik yayınla birlikte Blok H'de |
| SQLite dosyası + ayrı görüntüleyici | LocalDB + SQL Server Object Explorer | Canlıyla aynı motor |
| `HasData` + elle verilen `Id` | `UseSeeding` + `db.json`; numarayı veritabanı veriyor | Veri büyüyecek, anahtarları veritabanı üretiyor |
| `ToList()` (senkron) | `ToListAsync()` + `await` | Müfredat kararı |
| `.Where(u => u.Aktif)` | `?categoryId=` süzgeci (Gün 48). Seçilmişler gruplama sorgusuyla (Gün 56) | Seçilmişler bir bayrak değil, hesaplanan bir liste |
