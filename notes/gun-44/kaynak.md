# Gün 44 — Kaynak: LINQ sıralama — eşitlik olunca sırayı kim belirler?

Kaynaklar:
- https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/sorting-data
- https://learn.microsoft.com/en-us/dotnet/api/system.linq.enumerable.orderbydescending (Remarks)

**Soru:** İki yorumun `createdAt`'i aynıysa `OrderByDescending` onları hangi sırada döndürür? İlk kuralı bozmadan ikinci bir sıralama kuralı nasıl eklenir?

**Kısa cevap:** Bellekte LINQ eşit anahtarlı elemanların kaynaktaki sırasını korur; buna stable sort deniyor. İkinci kural `ThenBy` / `ThenByDescending` ile eklenir. İkinci bir `OrderBy` önceki sıralamayı birincil kural olmaktan çıkarır. Bellekteki bu güvence veritabanında yok. Bu yüzden sektörde her sıralamanın son anahtarı benzersiz bir alandır.

## 1. Neye göre söylersin, nasıl karşılaştırılacağını LINQ bilir

JS'te `toSorted((a, b) => a.x - b.x)` ile karşılaştırmayı sen yazıyordun. LINQ'te sadece anahtarı veriyorsun (`c => c.CreatedAt`); karşılaştırmayı anahtarın tipinin varsayılan karşılaştırıcısı yapıyor.

| Metot | Ne yapar |
|---|---|
| `OrderBy` / `OrderByDescending` | Birincil kural: artan / azalan |
| `ThenBy` / `ThenByDescending` | Birincil kuralda eşit kalanları kendi aralarında sıralar |
| `Reverse` | Mevcut sırayı ters çevirir; sıralamaz |

- `OrderBy` kaynağa dokunmaz, yeni bir sıra üretir (JS'teki `sort` değil, `toSorted` gibi).
- Ertelenmiş çalışır (Remarks'ın ilk paragrafı): sorgu, gezilince ya da `ToList()` çağrılınca çalışır.
- **Dikkat, metin sıralaması:** `string` için varsayılan karşılaştırıcı kültüre duyarlı, yani programın çalıştığı makinenin dil ayarına göre sıralıyor. Gün 43'teki `==` ise harf harf (ordinal) karşılaştırıyordu. Türkçe karakterler (`İ/ı`, `Ç`, `Ş`) bilgisayarında bir sırada, dil ayarı farklı olan Azure sunucusunda başka bir sırada çıkabilir. Veritabanında ise sıralamayı collation belirler. Metne göre sıralarken hangi kuralın geçerli olduğunu açıkça bil.

## 2. Eşit anahtar: stable sort

Remarks'ta yazdığı gibi: anahtarları eşit iki elemanın kaynaktaki sırası korunur. Bunu sağlayan sıralamaya stable sort deniyor. Bu, LINQ to Objects'in, yani bellekteki listelerin bir güvencesi.

Gizli bir bedeli var. Eşitlik durumunda sonucu kaynağın sırası belirliyor. Bugün kaynak `db.json`'daki dosya sırası. Biri dosyadaki iki yorumun yerini değiştirirse API'nin cevabı da değişir. Hiçbir kod satırı değişmeden davranış değişmiş olur.

## 3. İkinci kural: `ThenBy`, ikinci `OrderBy` değil

```csharp
comments.OrderByDescending(c => c.CreatedAt).ThenByDescending(c => c.Id)   // doğru
comments.OrderByDescending(c => c.CreatedAt).OrderByDescending(c => c.Id)  // tuzak
```

`OrderBy`'ın dönüş tipi `IOrderedEnumerable<T>`, yani "zaten sıralanmış" etiketi taşıyan bir sıra. `ThenBy` sadece bu tipin üzerinde çağrılabiliyor. İkinci bir `OrderBy` ise Remarks'taki ifadeyle yeni bir birincil sıra kuruyor ve öncekini yok sayıyor.

Tuzak satırın iki dünyada farklı sonucu var (Gün 43'teki "aynı satır, iki dünya" konusunun devamı):
- **Bellekte:** `OrderBy` stable olduğu için ilk kural, ikinci kuralda eşit kalanlar arasında gizlice hayatta kalıyor. Sonuç "önce Id, sonra tarih" gibi çıkıyor, yani öncelik ters.
- **Veritabanında:** EF Core yeni bir `OrderBy` gördüğünde öncekini SQL'e hiç yazmıyor; sadece son kural kalıyor.

Kısacası aynı kod bellekte bir sonuç, veritabanında başka bir sonuç veriyor. Bellekteki testten geçmesi bir şey kanıtlamıyor.

## 4. Sektör boyutu: veritabanında eşitlik bozulmuyor

SQL Server `ORDER BY CreatedAt DESC` sorgusunda tarihi eşit iki satırın sırasını garanti etmiyor. Aynı sorgu iki çalıştırmada farklı sıra dönebilir. Hiç `ORDER BY` yoksa hiçbir sıra garanti değil; Gün 43'te kategori öğelerinin sırasının Gün 48'e borç yazılmasının sebebi de buydu.

Sayfalama gelince (Gün 54) bu bir görünüm sorunu olmaktan çıkıp bir veri hatasına dönüşüyor. 1. sayfanın sonu ile 2. sayfanın başında tarihi eşit iki yorum varsa, biri iki sayfada birden görünebilir, diğeri hiçbirinde görünmeyebilir.

**Sektör kuralı:** Her sıralamanın son anahtarı benzersiz bir alandır, genellikle `Id`. Böylece sıra her çalıştırmada aynı çıkar. Eşitliği bozan bu son kurala tie-breaker deniyor. Veritabanında `Id` sırayla artan bir sayı olduğunda, `ThenByDescending(c => c.Id)` "aynı saniyede yazılanlardan sonra ekleneni önce göster" anlamına da geliyor.

## 5. Capstone karşılığı

- **Bugün, `GET /comments?itemId=`:** `OrderByDescending(c => c.CreatedAt).ThenByDescending(c => c.Id)`.
- **Sorgu kurma kalıbıyla çakışan nokta:** Kalıptaki değişken `IEnumerable<Comment> query`. `query = query.OrderByDescending(...)` derlenir, ama sonraki satırda `query.ThenBy(...)` derlenmez, çünkü değişkenin tipi "sıralanmış" etiketini taşımıyor. Çözüm: Süzgeçler `if`'lerle adım adım eklenir, sıralama ise en sonda tek bir zincir olarak yazılır.
- **Gün 48:** Kategori ve öğe listelerine açık sıralama gelir ve tie-breaker eklenir. Bellekteki stable sort güvencesi orada biter.
- **Gün 54:** Sayfalama, ancak sıra her çalıştırmada aynıysa doğru çalışır.
- **Gün 56:** "En çok yorum alan 4 öğe" sorgusunda 4. sırada eşitlik çıkabilir. Hangi öğenin listeye gireceğini tie-breaker belirler.
