# Domain-Driven Design (DDD) Nedir?

## Giriş

Karmaşık iş yazılımlarında en büyük sorun çoğu zaman teknik değildir; **iş ile kod arasındaki kopukluktur**. İş uzmanı "sipariş onaylanır" der, kodda ise `order.Status = 3` yazar. Aradan zaman geçince bu ikisi birbirinden uzaklaşır ve kimse kodun iş kurallarını gerçekten yansıttığından emin olamaz.

**Domain-Driven Design (DDD)**, yazılımı tam da bu iş alanının (domain) etrafında tasarlama yaklaşımıdır. Amacı, kodun iş dünyasının dilini ve kurallarını sadık biçimde konuşmasını sağlamaktır.

---

## Domain-Driven Design Nedir?

DDD, yazılımı **iş alanını (domain) merkeze alarak** tasarlama yaklaşımıdır. Eric Evans tarafından ortaya konmuştur ve temel fikri şudur: yazılımın kalbi, kullandığı teknoloji değil, çözdüğü iş problemidir.

Bu yaklaşımın amacı:

* Kodu, iş uzmanlarının konuştuğu dile yaklaştırmak
* İş kurallarını dağıtmak yerine, ait oldukları yerde toplamak
* Karmaşık alanları yönetilebilir sınırlara bölmek
* Yazılım ile iş dünyası arasındaki kopukluğu kapatmak

---

## DDD Bir Desen mi?

Hayır. DDD tek bir tasarım deseni ya da bir framework değildir; içinde **birçok deseni barındıran bir yaklaşımdır (metodoloji).** İki yarımdan oluşur:

```text
Domain-Driven Design
│
├── Stratejik Tasarım (büyük resim)
│     ├── Ubiquitous Language (Ortak Dil)
│     ├── Bounded Context (Sınırlı Bağlam)
│     └── Context Map (Bağlam Haritası)
│
└── Taktiksel Tasarım (yapı taşları)
      ├── Entity
      ├── Value Object
      ├── Aggregate (+ Aggregate Root)
      ├── Repository
      ├── Domain Service
      ├── Domain Event
      └── Factory
```

Yani "DDD nedir?" sorusunun en kısa cevabı: **iş alanını merkeze alan, stratejik ve taktiksel iki katmandan oluşan bir tasarım yaklaşımı.**

---

## Problem: Anemik Model

DDD'nin savaştığı asıl şey, **anemik (kansız) domain model**'dir. Burada nesneler yalnızca veri taşır; iş mantığı ise dağınık servislerin içine yayılır:

```csharp
// Anemik model — sadece veri kabı, davranış yok
public class Siparis
{
    public Guid Id { get; set; }
    public int Durum { get; set; }
    public decimal Tutar { get; set; }
}

// İş kuralı başka yerde, servisin içinde
public class SiparisServisi
{
    public void Onayla(Siparis s)
    {
        if (s.Tutar <= 0) throw new Exception("Geçersiz tutar");
        s.Durum = 2;
    }
}
```

Sorun: `Siparis` nesnesine bakan biri, bir siparişin ne yapabildiğini (onaylanabilir mi, iptal edilebilir mi) anlayamaz. Kurallar nesnenin dışında olduğu için, herhangi biri `s.Durum = 2` yazıp kuralı atlayabilir. DDD bu mantığı nesnenin **içine** taşır.

---

## Stratejik Tasarım

### Ubiquitous Language (Ortak Dil)

Geliştiriciler ile iş uzmanlarının **aynı kelimeleri** kullanmasıdır. İş uzmanı "sipariş onaylanır" diyorsa, kodda da `siparis.Onayla()` olmalıdır — `UpdateStatus(2)` değil. Kod, iş dilini birebir yansıtır.

### Bounded Context (Sınırlı Bağlam)

Büyük bir sistemde aynı kelime farklı yerlerde farklı şey ifade edebilir. "Müşteri", satış bağlamında farklı, muhasebe bağlamında farklıdır. **Bounded Context**, bir modelin tutarlı olduğu sınırı çizer: o sınırın içinde "Müşteri" tek bir net anlama gelir. Sistem, bu bağlamlara bölünerek yönetilebilir hale gelir.

---

## Taktiksel Tasarım: Yapı Taşları

### Entity (Varlık)

Bir **kimliği (identity)** olan nesnedir. İki Entity'nin eşitliği değerleriyle değil, **kimlikleriyle** belirlenir. Bir müşterinin adı değişse bile o hâlâ aynı müşteridir.

```csharp
public class Siparis
{
    public Guid Id { get; private set; }   // kimlik
    // İki Siparis, aynı Id'ye sahipse aynıdır.
}
```

### Value Object (Değer Nesnesi)

Kimliği **olmayan**, yalnızca değeriyle anlam taşıyan nesnedir. İki value object, tüm değerleri aynıysa eşittir. Değişmez (immutable) olmalıdır. C#'ta `record` bunun için biçilmiş kaftandır:

```csharp
public record Para(decimal Tutar, string ParaBirimi);

var a = new Para(100, "TL");
var b = new Para(100, "TL");
// a == b  →  true  (değer eşitliği, record sayesinde otomatik)
```

100 TL ile 100 TL birbirinden ayırt edilemez; "hangi 100 TL" diye sormak anlamsızdır. İşte value object budur.

### Aggregate ve Aggregate Root

Birbirine bağlı Entity ve Value Object'lerin, **tek bir bütün** olarak ele alındığı kümedir. Bu kümenin dışarıya açılan tek kapısı **Aggregate Root**'tur. Dışarıdan iç nesnelere doğrudan erişilemez; her şey root üzerinden yapılır. Böylece iş kuralları (invariant'lar) tek bir yerde korunur:

```csharp
public class Siparis   // Aggregate Root
{
    private readonly List<SiparisKalemi> _kalemler = new();
    public IReadOnlyList<SiparisKalemi> Kalemler => _kalemler.AsReadOnly();

    public void KalemEkle(Guid urunId, int adet)
    {
        if (adet <= 0)
            throw new ArgumentException("Adet pozitif olmalı");   // invariant korunuyor

        _kalemler.Add(new SiparisKalemi(urunId, adet));
    }
}
```

Dikkat: dışarıdaki kod bir `SiparisKalemi`'ni doğrudan listeye ekleyemez — `Kalemler` salt-okunur. Eklemek için `KalemEkle`'den geçmek zorundadır; kural orada korunur. Aggregate'in özü budur.

### Repository

Bir aggregate'i saklamak ve geri getirmek için kullanılan soyutlamadır. **Aggregate düzeyinde** çalışır; tek tek satır değil, bütün aggregate'i alır/verir:

```csharp
public interface ISiparisRepository
{
    Siparis Getir(Guid id);
    void Ekle(Siparis siparis);
    void Kaydet(Siparis siparis);
}
```

Domain, verinin nasıl saklandığını (SQL, NoSQL) bilmez; yalnızca bu arayüzü tanır.

### Domain Service

Tek bir Entity'ye doğal olarak ait olmayan iş mantığı, bir **domain service**'e konur. Örneğin iki hesap arasında para transferi ne "gönderen hesaba" ne de "alan hesaba" tek başına aittir:

```csharp
public class ParaTransferServisi
{
    public void Transfer(Hesap kaynak, Hesap hedef, Para tutar) { ... }
}
```

### Domain Event

Domain'de **önemli bir şey olduğunu** ifade eder ("Sipariş oluşturuldu", "Ödeme alındı"). Sistemin başka parçaları bu olaya tepki verebilir:

```csharp
public record SiparisOlusturuldu(Guid SiparisId, DateTime Tarih);
```

---

## Entity ve Value Object Arasındaki Fark

Bu ayrım DDD'nin kalbidir ve en çok karıştırılan noktadır:

| Entity                          | Value Object                       |
| ------------------------------- | ---------------------------------- |
| Kimliği (Id) vardır             | Kimliği yoktur                     |
| Eşitlik kimlikle belirlenir     | Eşitlik tüm değerlerle belirlenir  |
| Zamanla değişebilir             | Değişmezdir (immutable)            |
| Örnek: Müşteri, Sipariş         | Örnek: Para, Adres, Tarih aralığı  |

Pratik soru: "İki tanesi aynı değerlere sahipse, bunlar aynı şey midir?" Cevap **evet** ise value object, **hayır** ise entity'dir.

---

## Ne Zaman Kullanmalı, Ne Zaman Kullanmamalı?

**Kullan, eğer:**

* İş alanı karmaşıksa ve çok sayıda iş kuralı/invariant varsa
* Proje uzun ömürlüyse ve zamanla evrilecekse
* İş uzmanlarıyla yakın çalışıyorsan (ortak dil kurulabilir)

**Kullanma, eğer:**

* Uygulama büyük ölçüde basit CRUD ise (veri ekle-oku-güncelle-sil)
* Veri-merkezli, raporlama ağırlıklı bir sistemse
* Proje küçükse — DDD'nin getirdiği yapı, kazançtan fazla yük olur

DDD güçlü bir yaklaşımdır ama bedeli vardır. Basit bir sistemde katmanlı/işlem-script (transaction script) tarzı bir mimari çoğu zaman daha sade ve yeterlidir. DDD'yi, karmaşıklık onu **hak ettiğinde** kullan.

---

## Sonuç

Domain-Driven Design, yazılımı iş alanının etrafında tasarlama yaklaşımıdır. Stratejik tarafı (ubiquitous language, bounded context) büyük resmi; taktiksel tarafı (entity, value object, aggregate, repository, domain event) yapı taşlarını verir.

Özü, iş mantığını dağıtmak yerine ait olduğu yerde — domain nesnelerinin içinde — toplamaktır. Anemik bir veri kabını, davranışı ve kuralları kendi içinde taşıyan zengin bir modele dönüştürür.

Karmaşık, kural yoğun ve uzun ömürlü domain'lerde DDD, kod ile iş dünyası arasındaki kopukluğu kapatır; ancak basit sistemlerde gereğinden fazla yapı getirebileceği için, her zaman değil, ihtiyaç onu çağırdığında tercih edilmelidir.
