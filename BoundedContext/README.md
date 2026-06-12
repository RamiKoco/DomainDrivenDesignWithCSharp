# Bounded Context Nedir?

## Giriş

Şimdiye kadarki yazılar **taktiksel** taraftaydı: Entity, Value Object, Aggregate, Repository — yani yapı taşları. **Bounded Context** ise DDD'nin **stratejik** tarafındadır; büyük resmi, sistemi nasıl böleceğini ele alır.

Büyük bir sistemde her şeyi tek bir devasa model altında toplamaya çalışmak çöker. Bounded Context, bu büyük alanı yönetilebilir, tutarlı sınırlara bölmenin yoludur.

---

## Bounded Context Nedir?

Bounded Context (sınırlı bağlam), bir domain modelinin **tutarlı olduğu ve her terimin tek bir net anlam taşıdığı sınırdır.** Bu sınırın içinde "Müşteri" tek bir şey ifade eder; ortak dil (ubiquitous language) o sınır içinde belirsizlikten arınmıştır.

Temel fikri:

* Büyük bir domain'i, kendi içinde tutarlı parçalara bölmek
* Her parçanın (bağlamın) kendi modeline sahip olması
* Bir terimin, ait olduğu bağlam içinde tek anlama gelmesi

---

## Problem: Tek Büyük Model

Bounded Context olmadan, her departmanın ihtiyacı tek bir sınıfa tıkıştırılır ve o sınıf giderek şişen bir "god-object"e döner:

```csharp
public class Musteri
{
    // Satış için
    public decimal KrediLimiti { get; set; }
    // Kargo için
    public Adres TeslimatAdresi { get; set; }
    // Muhasebe için
    public string VergiNo { get; set; }
    // Destek için
    public List<DestekTalebi> Talepler { get; set; }
    // ... her departman ekledikçe büyür, kimse güvenle dokunamaz
}
```

Sorunlar:

* Tek `Musteri` sınıfı, ilgisiz onlarca sorumluluğu taşır
* Bir departmanın yaptığı değişiklik, diğerlerini etkileme riski taşır
* "Müşteri" kelimesi herkes için farklı anlama gelir ama model bunu görmezden gelir

---

## Aynı Kelime, Farklı Anlam

Mesele tam da şudur: **"Müşteri" her departmanda farklı bir şeydir.**

* **Satış**'ta müşteri = kredi limiti ve siparişler
* **Kargo**'da müşteri = bir isim ve teslimat adresi
* **Muhasebe**'de müşteri = vergi no ve fatura geçmişi
* **Destek**'te müşteri = açtığı talepler

Aynı kelime, dört farklı model. Bounded Context der ki: "Benim sınırımın içinde 'Müşteri' **şu** demektir." Her bağlam, terimi kendi ihtiyacına göre tanımlar.

---

## Çözüm: Sınırlara Bölmek

Tek bir `Musteri` yerine, her bağlamın **kendi** `Musteri`'si olur — küçük ve o bağlama odaklı:

```csharp
// Satış bağlamı
namespace Satis;
public class Musteri
{
    public Guid Id { get; set; }
    public decimal KrediLimiti { get; set; }
    public List<Siparis> Siparisler { get; set; }
}
```

```csharp
// Kargo bağlamı — bambaşka bir "Müşteri"
namespace Kargo;
public class Musteri
{
    public Guid Id { get; set; }
    public string Ad { get; set; }
    public Adres TeslimatAdresi { get; set; }
}
```

Gerçek dünyadaki aynı kişi (aynı `Id`), iki ayrı bağlamda iki ayrı model olarak yaşar. Her biri küçük, odaklı ve kendi takımının sahip olduğu bir modeldir.

---

## Ubiquitous Language ile İlişkisi

Ortak dil (ubiquitous language) **yalnızca bir bounded context içinde geçerlidir.** "Müşteri" kelimesi Satış'ın dilinde bir şey, Kargo'nun dilinde başka bir şey ifade eder. Yani bir bağlamın sınırı, aynı zamanda bir **dil sınırıdır**. Bu yüzden ortak dil "tüm sistem için tek dil" değil, "her bağlam için kendi dili" demektir.

---

## Bağlamlar Nasıl Konuşur?

Bounded context'ler modellerini **paylaşmaz**; birbirleriyle iyi tanımlanmış sözleşmeler üzerinden (API, mesaj/olay, paylaşılan ID) entegre olur. Bu bağlamlar arası ilişkilerin haritasına **Context Map** denir (ayrı bir konu). Aralarındaki ilişki türleri (shared kernel, customer-supplier, anti-corruption layer gibi) bu haritada tanımlanır.

Önemli olan şu: bir bağlam diğerinin iç modeline doğrudan dokunmaz; aralarında net bir sınır ve sözleşme vardır.

---

## Aggregate ile İlişkisi

Aggregate'ler bir bounded context'in **içinde** yaşar. Bir bounded context, bir veya birden çok aggregate içerir. Yani taktiksel yapı taşları (Entity, Value Object, Aggregate) her zaman bir bağlamın sınırının içinde çalışır. Bounded Context büyük resmi çizer; aggregate o resmin içindeki tutarlılık adacıklarını kurar.

---

## Tek Büyük Model ve Bounded Context Farkı

| Tek büyük model                      | Bounded Context'ler                   |
| ------------------------------------ | ------------------------------------- |
| "Müşteri" her yerde aynı sınıftır    | Her bağlamın kendi "Müşteri"si vardır |
| Şişen god-object                     | Küçük, odaklı modeller                |
| Terimler belirsizdir                 | Her terim, bağlamında tek anlamlıdır  |
| Takımlar birbirini engeller          | Her takım kendi bağlamının sahibidir  |

---

## Avantajları

* **Karmaşıklığı böler** — devasa bir model yerine yönetilebilir parçalar
* **Terimleri netleştirir** — her bağlamda her kelime tek anlama gelir
* **Takımları bağımsızlaştırır** — her bağlam kendi modelinin sahibi olur
* **Değişimi güvenli kılar** — bir bağlamdaki değişiklik diğerlerini etkilemez

---

## Dezavantajları ve Dikkat

* **Sınırı yanlış çizmek** — bağlamları yanlış yerden bölmek en kritik hatadır
* **Aşırı bölmek** — çok sayıda küçük bağlam, gereksiz entegrasyon karmaşıklığı doğurur
* **Entegrasyonu ihmal etmek** — bağlamlar arası sözleşmeleri (context map) tanımlamamak
* **Modelleri sızdırmak** — bir bağlamın iç modelini diğerine doğrudan paylaştırmak sınırı bozar

---

## Sonuç

Bounded Context, bir domain modelinin tutarlı olduğu ve her terimin tek anlam taşıdığı sınırdır. DDD'nin stratejik tarafının temelidir: büyük bir sistemi, her biri kendi modeline ve diline sahip yönetilebilir bağlamlara böler.

Asıl fikri şudur: "Müşteri", "Ürün" gibi kelimeler farklı yerlerde farklı şeyler ifade eder; bunu tek bir devasa modelde birleştirmeye çalışmak yerine, her bağlamın kendi modelini kurmasına izin vermek. Aggregate gibi taktiksel yapı taşları da her zaman bir bağlamın sınırının içinde çalışır.

Doğru çizilmiş bounded context'ler, karmaşık sistemleri hem teknik hem de takım açısından sürdürülebilir kılar — bu yüzden DDD'nin en stratejik kararıdır.
