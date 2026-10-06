# Gün 40 · Harita 1 — Backend'e giriş (dersler 382–389)

> Senin notlarının üstüne kuruldu: yanlışlar düzeltildi, eksikler tamamlandı. Düzeltilen yerler **Düzeltme** diye işaretli. Her bölümün sonundaki **Bizde** satırı, kavramın `turkiye-simulasyonu`'da nereye denk geldiğini söylüyor.

---

## 0. Büyük resim

Web'deki her etkileşim aynı döngü: tarayıcı sunucuya bir şey ister, sunucu bir şey geri gönderir. Giden pakete **request**, dönen pakete **response** deniyor. Değişen tek şey response'un içeriği:

- Sunucu sayfanın **hazır HTML'ini** üretip gönderir, tarayıcı sadece gösterir. Kursun %88'i bu: **MVC**.
- Sunucu sadece **veriyi (JSON)** gönderir, HTML'i tarayıcıdaki JS kurar. Senin React projen ve bu ay yazacağın API bu: **Web API**.

Kursun ilk slaytı bunu söylüyor: ASP.NET Core aynı temelden iki tür çıktı üretebilir, HTML ya da JSON.

---

## 1. Katmanlar: C#, .NET, ASP.NET Core

| .NET dünyası | Ne iş yapar | Frontend'deki karşılığı |
|---|---|---|
| C# | Kodu yazdığın dil | TypeScript |
| .NET | Kodu çalıştıran platform + hazır kütüphaneler (dosya, JSON, tarih…) | Node.js |
| ASP.NET Core | .NET'in üstündeki web çatısı: request karşılama, yönlendirme, response üretme | Express (sunucu tarafı) |
| NuGet | Paket deposu | npm |
| `.csproj` | Projenin ayar + bağımlılık dosyası | `package.json` |
| `dotnet` komutu | Proje oluşturma, derleme, çalıştırma | `npm` / `npx` |

**Kurulum iki parçadan oluşur.** Kod geliştirmek için gereken araç setinin (derleyici + `dotnet` komutları + çalıştırıcı) adı **SDK**. Hazır bir uygulamayı sadece çalıştırmaya yeten parçanın adı **runtime**. Geliştirici olarak SDK kurarsın; canlıdaki sunucuya runtime yeter.
`dotnet --list-sdks` SDK'ları, `dotnet --list-runtimes` runtime'ları listeler.

**Düzeltme — isim karmaşası** (notunda ".NET Core kurulu" yazıyordu; iş ilanlarını okurken de işine yarar):

| İsim | Dönem | Ne |
|---|---|---|
| .NET Framework | 2002–2019 | Sadece Windows'ta çalışan eski platform, son sürüm 4.8. Türkiye'de eski kurumsal projelerin çoğu hâlâ bunun üstünde ("ASP.NET MVC 5" gibi) |
| .NET Core | 2016–2019 | Baştan yazılmış, her işletim sisteminde çalışan yeni platform. Slayttaki ".NET Core" bu |
| .NET 5, 6 … 10 | 2020 → | "Core" kelimesi platformun adından düştü, tek isim ".NET" oldu. Web çatısının adında kaldı: **ASP.NET Core** |

- "Bende .NET Core kurulu" cümlesinin bugünkü doğru hâli: "Bende .NET SDK kurulu, sürümü X." Sürüm önemli, biz 10 kullanıyoruz.
- İlanda ".NET Framework" görürsen eski Windows projesi; ".NET Core" ya da ".NET 6+" görürsen bu ay öğrendiğin dünya.
- Çift numaralı sürümler (8, 10) **LTS**, yani 3 yıl güvenlik yaması alır. Tek numaralılar (9) daha kısa süre destek alır. Hoca 9 kullanıyor, biz 10.

**Bizde:** Sandbox ve capstone .NET 10. Eklenecek Microsoft paketlerinin ana sürümü 10 olacak.

---

## 2. ASP.NET Core ile web uygulaması yapmanın üç yolu

| Yol | Response ne? | HTML'i kim üretir? | Nerede görürsün? | Bizde |
|---|---|---|---|---|
| **MVC** | HTML | Sunucu (View) | Kurumsal ve eski projeler, admin panelleri | Kursun projesi, `Sandbox` |
| **Razor Pages** | HTML | Sunucu | Az sayfalı basit siteler. Her sayfa kendi dosyası + kendi kodu, Controller yok | Kullanmıyoruz |
| **Web API** | JSON | Tarayıcıdaki JS (React, mobil uygulama…) | Ayrı frontend'i olan her proje | `turkiye-simulasyonu-api` |

Üçü de aynı temeli paylaşır: request karşılama, yönlendirme, veritabanı, kullanıcı yönetimi. Bu yüzden kursun MVC kısmının çoğu Web API'ye aynen taşınıyor.

---

## 3. HTML'i kim üretiyor? İki dünya

### Düzeltme: "derleniyor" değil, "üretiliyor"

Notunda "sunucuda derlenip gönderiyor, frontend'de tarayıcıda derleniyordu" yazıyordu. İki ayrı iş karışmış:

- Kodun **bir kez, request gelmeden önce** çalıştırılabilir hâle çevrilmesine **compile** deniyor. `dotnet build` C#'ı compile eder. React'te bunu Vite yapıyordu: JSX ve TS'yi JS'e çeviriyordu. Razor şablonları da build sırasında compile edilir.
- **Her request'te** olan şey başka: compile edilmiş kod o anki veriyle çalışır ve HTML'i çıkarır. Buna **render** deniyor, React'ten bildiğin kelime.

Yani mantığın doğru (HTML'i sunucu mu kuruyor, tarayıcı mı), kelime yanlış.

| | MVC | Senin React projen |
|---|---|---|
| Sayfayı açınca ilk gelen | İçi dolu HTML | Neredeyse boş `index.html` + JS dosyası |
| Veri | HTML'in içine yerleşmiş gelir | Sonradan ayrı request'le JSON olarak gelir |
| HTML'i kuran | Sunucudaki View | Tarayıcıda çalışan React |
| Başka sayfaya geçince | Yeni request, yeni tam HTML | Sayfa yenilenmez, sadece yeni veri istenir |
| Adı | **server-side rendering** | **client-side rendering** |

### Düzeltme: response tek parça değil

Notunda "response yine html css js olarak" yazıyordu. Sunucu her request'e **tek bir** response döner. MVC'de ilk response HTML'dir. Tarayıcı HTML'i okurken içinde `<link href="site.css">` ve `<script src="site.js">` görür ve **her biri için ayrı request** atar. Bu dosyalar her request'te değişmeden, olduğu gibi gönderilir; bunlara **static files** deniyor. ASP.NET Core'da `wwwroot/` klasöründe dururlar.
Dünkü şemada Vite'ın yaptığı iş de buydu: `index.html` ve JS dosyalarını olduğu gibi göndermek.

### Senin sorun: "Bizim frontend View'un yerini alamaz mı?"

Alır, ve Web API tam olarak bu. React View'un işini üstlenince sunucunun HTML üretmesine gerek kalmaz. Controller aynı kalır, sadece son satırı değişir:

```csharp
// MVC: HTML döner
public class CoursesController : Controller
{
    public IActionResult Index()
    {
        var courses = GetCourses();   // Model nesneleri: List<Course>
        return View(courses);         // View'a ver, HTML'i o üretsin
    }
}

// Web API: JSON döner
[ApiController]
[Route("courses")]
public class CoursesController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        var courses = GetCourses();   // aynı veri
        return Ok(courses);           // JSON'a çevrilip gönderilir
    }
}
```

Köşeli parantezli satırlar sınıfa ve metoda etiket yapıştırır: hangi adrese, hangi metotla gelen request'e cevap verdiğini söyler. Bunlara **attribute** deniyor. Gün 41'de kendin yazacaksın.

**Sektör hangisini ne zaman seçer:**

- **MVC:** Arama motorunda görünmesi gereken içerik siteleri, iç kurum uygulamaları, admin panelleri, tek ekibin hem arka hem ön yüzü yazdığı projeler. Tek proje, tek deploy.
- **SPA + Web API:** Etkileşimi yoğun uygulamalar, aynı API'yi web ve mobil uygulamanın ortak kullandığı ürünler, frontend ve backend'in ayrı ekiplerde olduğu şirketler. Bedeli: iki proje, iki deploy, CORS ve giriş için token.
- Arası da var: Next.js gibi çatılar React'i sunucuda HTML'e çevirip gönderiyor. Şimdilik bilmen yeterli.

**Bizde:** React = View. API sadece JSON döner. Bu kararın bedelini Gün 45'te (CORS) ve Hafta 4'te (token) ödeyeceğiz.

---

## 4. MVC: parçalar ve bir request'in yolculuğu

### Düzeltme: MVC ne için var?

Notundaki "uygulama büyüyünce dosyaların bölümlere göre sınıflandırılması" sonucu anlatıyor, sebebi değil. Asıl fikir şu: **her parçanın tek bir işi var ve diğerlerinin içini bilmiyor.** Klasörler bu ayrımın görünen yüzü. İşi tek olan bir parçayı değiştirmek diğerlerini bozmaz. Bu ilkeye **separation of concerns** deniyor.
Bunu zaten yaptın: frontend'de `fetch`'i sadece `client.ts` bilir, adresleri sadece `endpoints.ts` bilir, bileşenler ikisini de bilmez.

### Parçalar

| Parça | İşi | Klasörü | Not |
|---|---|---|---|
| **Model** | Veriyi taşıyan sınıflar, ör. `Course { Id, Title, Price }` | `Models/` | **Düzeltme:** "Modal" değil Model. Modal, frontend'deki açılır pencere |
| **Controller** | Request'i karşılar, ne yapılacağına karar verir, gereken veriyi ister, response'u seçer | `Controllers/` | Veritabanıyla kendisi konuşabilir, ama sektörde bu iş servise devredilir (aşağıda) |
| **View** | Model'i alıp HTML'e döker. Razor: HTML'in içine gömülü C# | `Views/<Controller>/<Action>.cshtml` | — |

Slaytta olmayan ama akışın ilk adımı olan parça:

- Gelen adrese bakıp hangi Controller'ın hangi metodunun çalışacağına karar veren mekanizmaya **routing** deniyor.
- Controller'ın içindeki, bir request'i karşılayan her metoda **action** deniyor.

### Akış

```
Tarayıcı ── GET /courses ──▶ Kestrel (uygulamanın içinde, bir portu dinleyen web sunucusu)
                                │
                                ▼
                         Routing: "/courses" → CoursesController.Index()
                                │
                                ▼
                         Controller: action çalışır
                           │  veriyi ister ──▶ servis / veritabanı
                           │  ◀── List<Course>   (Model nesneleri)
                           ▼
                         View: Views/Courses/Index.cshtml
                           (Model'i alır, HTML'e döker)
                                │
Tarayıcı ◀── 200 OK + HTML ─────┘
   │
   └── HTML'deki her <link> / <script> için ayrı request ──▶ wwwroot/ (static files)
```

Slayttaki `/kurslar` örneği de aynısı: `KurslarController.Index()` çalışır.

### Slayttaki oklar hakkında

Slayttaki Model → Controller → View okları **verinin** yolunu gösteriyor, request'in değil. Request her zaman önce routing üzerinden Controller'a gelir; Model ilk durak değil.

### Adres → kod eşlemesi

`Program.cs`'teki varsayılan kural: `{controller=Home}/{action=Index}/{id?}`
Yani adresin ilk parçası Controller'ın adı, ikincisi action'ın adı, üçüncüsü isteğe bağlı bir `id`. Yazılmayan parça için `=` sonrasındaki varsayılan kullanılır.

| Adres | Çalışan kod | Kullanılan View |
|---|---|---|
| `/` | `HomeController.Index()` (iki varsayılan) | `Views/Home/Index.cshtml` |
| `/Home/Privacy` | `HomeController.Privacy()` | `Views/Home/Privacy.cshtml` |
| `/Courses/Details/5` | `CoursesController.Details(5)` | `Views/Courses/Details.cshtml` |

Sınıf adı `Controller` ile biterse ve View dosyaları isme göre yerleşirse, hiçbir ayar yazmadan birbirlerini bulurlar. Ayar yazmak yerine isim kuralına güvenen bu yaklaşıma **convention over configuration** deniyor.
Bedeli: adı yanlış yazarsan hata compile sırasında değil, sayfa açılınca çıkar.

### "Model" kelimesinin iki anlamı

- Klasik MVC teorisinde (kitaplarda, Vikipedi'de) Model = veri **+ iş kuralları + veritabanı erişimi**.
- ASP.NET Core pratiğinde `Models/` klasörü sadece **veri taşıyan sınıflar**. İş kuralları **servis** sınıflarına, veritabanı erişimi **EF Core**'a gider. Hocanın "veri taşır" demesi bu yüzden.
- Büyüyen projede yerleşik düzen: Controller ince kalır (request'i alır, servise iletir, response'u döner), asıl işi servis yapar. Kurs bunu 520. derste anlatıyor; capstone'da Gün 49'da yapıyoruz.

---

## 5. URL'in parçaları

Slayttakine iki parça eksik: port ve query string.

```
https://localhost:7123/items?categoryId=yemek-kulturu
└─┬─┘   └───┬───┘ └┬─┘└─┬──┘└──────────┬───────────┘
protokol  alan adı port path       query string
```

- **Port:** Aynı bilgisayarda hangi programın kapısına gidildiği. Dünkü şemada Vite 5173'te, json-server 3000'de duruyordu. Yazılmazsa varsayılan: http 80, https 443.
- **Query string:** `?` sonrası, `&` ile ayrılan `anahtar=değer` çiftleri. Filtre, arama, sayfa numarası gibi isteğe bağlı ayarları taşır.
- Protokol + alan adı + port üçlüsüne **origin** deniyor. Üçünden biri farklıysa tarayıcı orayı "başka bir yer" sayar. `localhost:5173` ile `localhost:7123` farklı origin'ler. Gün 45'teki CORS ayarının sebebi bu.

---

## 6. HTTP: request ve response

### Ham hâli

Dünkü şemadaki request'e kablonun içinden bakınca:

```http
GET /items?categoryId=yemek-kulturu HTTP/1.1
Host: localhost:3000
Accept: application/json
Origin: http://localhost:5173
```

```http
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8

[{"id":"kebap","categoryId":"yemek-kulturu","name":"Kebap", ...}]
```

Body taşıyan bir request (yorum gönderme, Gün 50):

```http
POST /comments HTTP/1.1
Content-Type: application/json

{"itemId":"kebap","author":"misafir","body":"Adana mı Urfa mı?"}
```

Bu biçim tesadüf değil: capstone'da her uç için yazacağın `.http` dosyaları tam olarak böyle görünecek.

### Metotlar

Slayt sadece GET ve POST'u gösteriyor; API'de beşi de kullanılır.

| Metot | Ne ister | Body | Aynı request iki kez gelirse | Bizde |
|---|---|---|---|---|
| GET | Oku | Yok | Sonuç değişmez | Bütün okuma uçları |
| POST | Yeni kayıt oluştur | Var | İki kayıt oluşur | `POST /comments` |
| PUT | Kaydı verilen hâliyle baştan değiştir | Var | Sonuç değişmez | `PUT /surveys/rebirth/my-vote` |
| PATCH | Kaydın bir kısmını değiştir | Var | Duruma göre | — |
| DELETE | Sil | Genelde yok | Sonuç değişmez (zaten silinmiş) | — |

Aynı request'in iki kez gelmesi sonucu değiştirmiyorsa o metoda **idempotent** deniyor. Önemi: ağ koptuğunda request güvenle tekrarlanabilir mi? GET, PUT ve DELETE tekrarlanabilir; POST tekrarlanırsa çift yorum oluşur. TanStack Query'nin okumaları kendiliğinden yeniden denemesi, yazmaları (mutation) ise denememesi bu yüzden.

### Header örnekleri

| Header | Kim gönderir | Ne der |
|---|---|---|
| `Content-Type` | İkisi de | "Body'nin biçimi şu" (`application/json`, form verisi…) |
| `Accept` | İstemci | "Response'u şu biçimde isterim" |
| `Authorization` | İstemci | "Token'ım bu" (Hafta 4) |
| `Origin` | Tarayıcı, kendiliğinden | "Bu request'i şu siteden atıyorum" (CORS) |
| `Set-Cookie` / `Cookie` | Sunucu / tarayıcı | Çerez yaz / çerezi geri gönder |

### Body: form verisi mi, JSON mu?

Slaytta request body'nin yanında "Form Data" yazıyor. Bu MVC'ye özgü: HTML formu gönderilince body `name=Ali&age=20` biçiminde gider (`Content-Type: application/x-www-form-urlencoded`). API'de body JSON'dur (`application/json`). İkisini ayıran şey `Content-Type` header'ı; ASP.NET Core body'yi buna bakarak C# nesnesine çevirir.

### Status code aileleri

| Aile | Anlamı | Örnekler |
|---|---|---|
| 2xx | Tamam | 200 OK · 201 Created (yeni kayıt oluştu) · 204 No Content (başarılı, gövde yok) |
| 3xx | Başka yere git | 301/302 yönlendirme (MVC'de form gönderdikten sonra çok görürsün) |
| 4xx | **İstemcinin** hatası | 400 hatalı request · 401 giriş yok · 403 yetki yok · 404 yok · 429 çok fazla request |
| 5xx | **Sunucunun** hatası | 500 sunucuda beklenmeyen hata · 503 hizmet geçici olarak yok |

Bu ayrımı zaten kullanıyorsun: `queryClient.ts`'te 4xx'i tekrar denemiyor, 5xx'i deniyorsun, çünkü istemci hatası tekrar denemekle düzelmez. Backend'de bu kodları artık **sen seçeceksin**. Yanlış kod seçmek, örneğin her hataya 500 dönmek, frontend'in bu mantığını bozar.

### Stateless: ne demek, neyi doğurur?

Sunucu iki request arasında seni hatırlamaz; her request tek başına, kendi bilgisiyle gelir.

- **Sorunu:** Giriş yaptın, sonraki request'te sunucu "bu kim?" bilmez. Çözüm, kimlik bilgisinin her request'le birlikte gönderilmesi: ya tarayıcının otomatik geri yolladığı bir **çerez** ile (MVC'nin klasik yolu) ya da her request'e eklenen bir **token** ile (`Authorization` header'ı). Hafta 4'teki "token mı, çerez mi" kararı tam olarak bu.
- **Yararı:** Sunucu kimseyi hatırlamak zorunda olmadığı için aynı uygulamanın 10 kopyası yan yana çalıştırılabilir; request hangisine giderse gitsin response aynı olur. Milyonlarca kullanıcıya ölçeklemenin temeli bu.

---

## 7. Komutlar ve araçlar

### Hocanın komutu, parça parça

```
dotnet new mvc -o dotnet-basics -f net9.0
```

| Parça | Anlamı |
|---|---|
| `dotnet new` | Hazır şablondan yeni proje oluştur |
| `mvc` | Şablonun kısa adı. `dotnet new list` hepsini gösterir; Web API için `webapi` |
| `-o dotnet-basics` | Çıktı klasörü; proje adı da bu olur |
| `-f net9.0` | Hedef .NET sürümü → **biz `net10.0`** |

`dotnet -h` komut listesini verir; her komutta da çalışır: `dotnet new -h`.

### Kursta görmediysen de gerekecek

| Komut | Ne yapar | npm karşılığı |
|---|---|---|
| `dotnet --list-sdks` | Kurulu SDK'ları listeler | `node -v` |
| `dotnet build` | Compile eder, hata ve uyarıları gösterir | `npm run build` (+ `typecheck`) |
| `dotnet run` | Compile edip çalıştırır | `npm run dev` |
| `dotnet watch` | Çalıştırır, dosya değişince kendini yeniler | Vite'ın anlık yenilemesi |
| `dotnet add package X` | NuGet paketi ekler | `npm i X` |

### Visual Studio yolu

- **Create a new project** → "ASP.NET Core Web App (Model-View-Controller)" → ad, konum → Framework **.NET 10.0 (LTS)** (hoca 9.0 seçti) → Create.
- **Çalıştırma:** F5 = hata ayıklayıcıyla çalıştır, breakpoint'lerde durur. Ctrl+F5 = hata ayıklayıcısız çalıştır, daha hızlı açılır. Durdurmak: Shift+F5.
- **Düğmenin arkasında ne var?** VS'nin çalıştır düğmesi aslında `dotnet build` + `dotnet run` yapıyor. Komutları bilmek bu yüzden zorunlu: sunucuda ve otomatik derleme sisteminde (GitHub Actions) düğme yok, sadece komut var.
- **Port nereden geliyor?** `Properties/launchSettings.json`. Proje oluşturulurken rastgele seçilir (https için 7xxx, http için 5xxx). Uygulamanın hangi ortamda çalıştığını (`ASPNETCORE_ENVIRONMENT: Development`) da burası söyler.
- **Solution nedir?** Bir ya da birden fazla projeyi bir arada tutan kap (`.sln`, yeni biçimde `.slnx`). VS projeyi değil solution'ı açar. İleride capstone'a bir test projesi eklendiğinde ikisi aynı solution'da duracak.

---

## 8. Bu ay için çıkarım

- Kursun MVC'si: sunucu HTML üretir. Capstone: sunucu JSON üretir, HTML'i React kurar.
- Aynı kalan: routing, controller, model, servis, veritabanı, HTTP kuralları. Değişen: View yok, `Controller` yerine `ControllerBase`, `View()` yerine `Ok()` / `NotFound()`.
- Kursu izlerken her derste kendine sor: **"Bu, View'a mı özgü, yoksa View'dan önceki kısım mı?"** İlki tanıma seviyesinde kalır, ikincisi capstone'a taşınır.
