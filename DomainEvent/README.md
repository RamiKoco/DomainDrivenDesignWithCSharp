# Domain Event Nedir?

## Giriş

Aggregate yazısında, "bir transaction'da bir aggregate değiştirilir, aggregate'ler arası tutarlılık genelde domain event'lerle sağlanır" demiştik. Şimdi o domain event'leri açalım.

**Domain Event**, domain'de **önemli bir şeyin olduğunu** bildiren bir olaydır: "Sipariş onaylandı", "Ödeme alındı". Bir şeyin gerçekleştiğini duyurur; o şeye kimin nasıl tepki vereceğini ise bilmez.

---

## Domain Event Nedir?

Domain Event, domain'de gerçekleşmiş, iş açısından anlamlı bir olayı temsil eder. **Geçmiş zamanda** ifade edilir, değişmezdir ve olayla ilgili veriyi taşır.

Temel nitelikleri:

* **Geçmiş zaman** — olmuş bir şeyi anlatır ("SiparisOnaylandi"), emir vermez
* **Değişmez (immutable)** — olan oldu, değiştirilemez
* **Veri taşır** — neyin olduğunu anlatan bilgiyi içerir (Id, tarih...)
* **Tepkiden bağımsızdır** — olayı çıkaran, kimin dinlediğini bilmez

---

## Problem: Doğrudan Çağrı

Domain event olmadan, bir şey olduğunda ona tepki vermesi gereken her yer doğrudan çağrılır:

```csharp
public void Onayla()
{
    Durum = SiparisDurumu.Onaylandi;

    _emailServisi.Gonder(...);      // sipariş, e-posta servisini tanıyor
    _stokServisi.Dus(...);          // ve stok servisini
    _kargoServisi.Bildir(...);      // ve kargo servisini
}
```

Sorunlar:

* `Siparis`, kendisine tepki veren tüm servisleri tanımak zorunda (sıkı bağ)
* Yeni bir tepki eklemek (örneğin SMS göndermek), bu metodu yeniden değiştirmeyi gerektirir
* İş mantığı (onaylama) ile yan etkiler (e-posta, stok) iç içe geçer

---

## Çözüm: Olayı Duyur

Bunun yerine aggregate yalnızca **olayı duyurur**; kimin dinlediğini bilmez:

```csharp
public record SiparisOnaylandi(Guid SiparisId, DateTime Tarih);

public class Siparis
{
    private readonly List<object> _olaylar = new();
    public IReadOnlyList<object> Olaylar => _olaylar.AsReadOnly();

    public void Onayla()
    {
        if (!_kalemler.Any())
            throw new InvalidOperationException("Boş sipariş onaylanamaz");

        Durum = SiparisDurumu.Onaylandi;
        _olaylar.Add(new SiparisOnaylandi(Id, DateTime.UtcNow));   // sadece duyur
    }
}
```

Tepki verecek olanlar ise ayrı **handler**'lardır; her biri olaya kendi tepkisini verir:

```csharp
public class SiparisOnaylandiEmailHandler
{
    public void Handle(SiparisOnaylandi olay) { /* e-posta gönder */ }
}

public class SiparisOnaylandiStokHandler
{
    public void Handle(SiparisOnaylandi olay) { /* stok düş */ }
}
```

Yeni bir tepki gerektiğinde `Siparis`'e hiç dokunmazsın — yeni bir handler eklersin.

---

## İsimlendirme: Geçmiş Zaman

Domain event, **olmuş bir şeyi** anlattığı için geçmiş zamanda adlandırılır:

* ✅ `SiparisOnaylandi`, `OdemeAlindi`, `StokTukendi`
* ❌ `SiparisOnayla`, `OdemeAl` (bunlar emir/komuttur, olay değil)

Bu ayrım önemlidir: **komut** bir şey yapılmasını ister (gelecek), **olay** bir şeyin olduğunu bildirir (geçmiş).

---

## Observer Pattern ile İlişkisi

Domain Event, aslında **Observer Pattern'in domain seviyesindeki uygulamasıdır.** Aggregate, olayı duyuran **subject**'tir; handler'lar ise ona tepki veren **observer**'lardır. Aggregate, observer'larının kim olduğunu bilmez; sadece "şu oldu" der.

(Tasarım deseni reposundaki Observer yazısını hatırla: orada `KahveMakinesi` subject, `event` ile abonelere duyuruyordu. Domain Event aynı fikrin iş diliyle ifade edilmiş hâlidir.)

---

## Aggregate'ler Arası Tutarlılık

Domain Event'in en önemli kullanımı budur. Aggregate yazısındaki kuralı hatırla: bir transaction'da bir aggregate değiştirilir. Peki bir sipariş onaylanınca **Stok** aggregate'inin de güncellenmesi gerekiyorsa?

Cevap: `Siparis` aggregate'i `SiparisOnaylandi` olayını yayar; bir handler bu olayı yakalayıp **Stok** aggregate'ini ayrı bir transaction'da günceller. Buna **eventual consistency** (nihai tutarlılık) denir — iki aggregate anında değil, olay üzerinden, kısa bir gecikmeyle tutarlı hâle gelir.

---

## Doğrudan Çağrı ve Domain Event Farkı

| Doğrudan çağrı                       | Domain Event                          |
| ------------------------------------ | ------------------------------------- |
| Aggregate tüm tüketicileri tanır     | Aggregate kimin dinlediğini bilmez    |
| Sıkı bağ                             | Gevşek bağ                            |
| Yeni tepki = metodu değiştir         | Yeni tepki = yeni handler             |
| Senkron, tek transaction             | Genelde ayrı transaction (eventual)   |

---

## Avantajları

* **Gevşek bağ** — aggregate, tepki verenleri tanımadan onları tetikler
* **Genişletilebilirlik** — yeni tepki, mevcut kodu değiştirmeden eklenir
* **İş mantığını temiz tutar** — yan etkiler (e-posta, stok) aggregate'in dışında kalır
* **Aggregate'ler arası tutarlılık** — olaylar üzerinden, sınırları bozmadan sağlanır

---

## Dezavantajları ve Dikkat

* **Olayı komut gibi kullanmak** — event olguyu bildirir, emir vermez; ikisini karıştırma
* **Aşırı event üretmek** — her küçük değişiklik için event çıkarmak akışı izlenemez kılar
* **Eventual consistency karmaşıklığı** — anlık değil gecikmeli tutarlılık, zamanlama sorunları doğurabilir
* **Olayı yaymayı unutmak** — toplanan olayların doğru noktada handler'lara iletilmesi gerekir

---

## Sonuç

Domain Event, domain'de gerçekleşmiş anlamlı bir olayı bildiren, geçmiş zamanda adlandırılmış, değişmez bir DDD yapı taşıdır. Bir şeyin olduğunu duyurur; o şeye kimin tepki vereceğini bilmez.

Özü, olayı çıkaran ile ona tepki vereni birbirinden ayırmaktır — bu, Observer Pattern'in domain diliyle ifade edilmiş hâlidir. Böylece iş mantığı yan etkilerden arınır, yeni tepkiler mevcut kodu bozmadan eklenir.

En değerli kullanımı aggregate'ler arası tutarlılıktır: "bir transaction bir aggregate" kuralını bozmadan, aggregate'lerin olaylar üzerinden, nihai tutarlılıkla haberleşmesini sağlar. Bu yüzden Domain Event, taktiksel yapı taşlarını birbirine bağlayan tutkaldır.
