# Kaynak — Gün 52: Geliştirmede sırları saklamak (Secret Manager)

**Doküman:** "Safe storage of app secrets in development in ASP.NET Core" (learn.microsoft.com/aspnet/core/security/app-secrets)
**Soru:** *Where does the Secret Manager store secrets, and why is it not meant for production?*

**Kısa cevap:** Secret Manager sırları proje klasörünün dışında, kullanıcının profilindeki bir JSON dosyasında saklar: `%APPDATA%\Microsoft\UserSecrets\<UserSecretsId>\secrets.json`. Canlı ortam için uygun değil, çünkü:
1. Dosya şifresiz.
2. Dosya sadece geliştiricinin bilgisayarında durur, sunucuya hiç gitmez.
3. Uygulama bu dosyayı varsayılan olarak sadece Development ortamında okur.

## Çözdüğü sorun

- Bağlantı dizesi ya da API anahtarı `appsettings.json`'a yazılınca depoya girer. Hocanın Gün 51'de yaptığı buydu.
- Git geçmişi unutmaz: Sonraki bir commit'te silinen parola eski commit'te durmaya devam eder. Public depolar otomatik botlarla sürekli taranır.
- Secret Manager'ın asıl fikri şifreleme değil, **konum**. Değer proje ağacının dışında durduğu için ne `git add` ile depoya ne de `dotnet publish` çıktısına girebilir.

## Nasıl çalışıyor

- `dotnet user-secrets init` → csproj'a `<UserSecretsId>` (bir GUID) yazar.
  - Bu kimlik sır değil, depoya girer. Sadece "bu projenin sır klasörü hangisi" sorusunun cevabı.
- `dotnet user-secrets set "ConnectionStrings:Azure" "..."` → değeri `secrets.json`'a yazar. Dosya ilk `set` komutunda oluşur.
- Diğer komutlar:
  - `list`: listele
  - `remove "<anahtar>"`: tek anahtarı sil
  - `clear`: hepsini sil
- VS'de: projeye sağ tık → **Manage User Secrets** → aynı dosyayı açar.
- Anahtarlar `:` ile iç içe yazılır. `ConnectionStrings:Azure`, JSON'daki `{ "ConnectionStrings": { "Azure": "..." } }` ile aynı yere denk gelir. Dizi elemanları sıra numarasıyla yazılır: `Cors:AllowedOrigins:0`.
- Linux ve macOS'ta yol: `~/.microsoft/usersecrets/<UserSecretsId>/secrets.json`.

## Neden canlıda kullanılmaz

1. **Şifresiz:** O Windows hesabına erişen her program ve kişi dosyayı okuyabilir. Doküman açıkça "güvenilir bir depo değil" diyor.
2. **Sunucuya gitmez:** Dosya geliştiricinin profilinde durur, deploy edilen paketin içinde yer almaz.
3. **Varsayılan olarak sadece Development'ta okunur:** `WebApplication.CreateBuilder` bu kaynağı yalnızca ortam Development ise ekler. Production'da dosya olsa bile okunmaz.

**Canlıda bunun yerine kullanılanlar:**
- **Ortam değişkenleri:** App Service'te girilen ayarlar uygulamaya ortam değişkeni olarak geçer.
- **Merkezi sır kasası:** Sırra kimin erişebileceğini belirleyen ve kimin ne zaman okuduğunu kaydeden ayrı bir servis. Sırrı yenilemeyi (rotation) de kolaylaştırır. Azure'daki adı **Key Vault**.
- **Parolasız bağlantı:** Uygulama Azure'daki kendi kimliğiyle (managed identity) bağlanır; ortada saklanacak bir parola kalmaz (Blok H).

## Kim kimi ezer

ASP.NET Core ayarları katman katman okur ve **sonra okunan kazanır**:

1. `appsettings.json`
2. `appsettings.{Environment}.json`
3. User secrets (sadece Development)
4. Ortam değişkenleri
5. Komut satırı argümanları

- **Ezme anahtar bazında olur.** Secrets'taki tek bir anahtar sadece o anahtarı değiştirir; dosyanın geri kalanı yerinde kalır.
- **Ortam değişkenleri secrets'ı da ezer.** Bilgisayarında unutulmuş bir `ConnectionStrings__Default` değişkeni, secrets'taki değeri sessizce geçersiz kılar.
- **Ortam değişkeninde `:` yerine `__` (çift alt çizgi) yazılır:** `ConnectionStrings__Default`. `:` her platformda çalışmıyor; örneğin Linux kabuğu kabul etmiyor.
- App Service'in "Connection strings" bölümüne `Default` adıyla girilen değer, uygulamaya `ConnectionStrings:Default` olarak ulaşır.

**Capstone kararı buradan çıkıyor:** Azure bağlantı dizesinin secrets'taki anahtarı `ConnectionStrings:Azure` olacak. Anahtar `Default` olsaydı:
- 3. katman, 2. katmandaki LocalDB dizesini ezerdi.
- F5 canlı veritabanına bağlanırdı.
- Test yorumların canlıya yazılırdı.

## Sektör boyutu

- **Sızan sır silinmez, değiştirilir.** Commit'ten silmek yetmez, çünkü değer geçmişte durur. Önce parola yenilenir ve eskisi geçersiz olur; sonra gerekirse geçmiş temizlenir.
- **GitHub sır taraması:** Public depolarda bilinen sır biçimlerini tarar, bazılarında push'u durdurur. Her biçimi tanımaz; son kapı yine senin alışkanlığın.
- **Ekipte düzen:**
  - Sır olmayan ortak yerel ayarlar (LocalDB dizesi gibi) depodaki `appsettings.Development.json`'da durur.
  - Kişiye özel sırlar herkesin kendi `secrets.json`'unda durur.
  - Yeni gelen biri hangi anahtarı `set` etmesi gerektiğini README'den öğrenir.
- **Büyük ekiplerde** geliştirme ortamı da çoğu zaman Key Vault'tan okur. `secrets.json` kişisel kullanımda ve küçük ekiplerde yaygın.

## Bugün kanıtlananlar

- `dotnet user-secrets init` çıktısı ve csproj'a eklenen satır: ⟨doldur⟩
- `dotnet user-secrets list` (set'ten önce): ⟨doldur⟩
- Deney: Secrets'taki `Cors:AllowedOrigins:0`, `appsettings.Development.json`'daki listeyi ezdi mi? `Access-Control-Allow-Origin` başlığı kayboldu mu, `remove`'dan sonra geri geldi mi? ⟨doldur⟩
