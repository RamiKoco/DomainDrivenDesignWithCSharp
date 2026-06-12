# Aggregate Nedir?

## Giriş

**Aggregate**, DDD'nin en merkezi ve en sık yanlış anlaşılan yapı taşıdır. Entity ve Value Object'i bir araya getirip onları **tek bir bütün** olarak ele alır ve en önemlisi, iş kurallarının (invariant) hiçbir zaman bozulmamasını garanti eder.

Bir önceki yazılarda tek tek nesneleri gördük; Aggregate, bu nesneleri bir arada tutarlı tutmanın yoludur.

---

## Aggregate Nedir?

Aggregate, birbirine bağlı Entity ve Value Object'lerin oluşturduğu, **tek bir birim olarak değiştirilen** kümedir. Bu kümenin dışarıya açılan tek kapısı **Aggregate Root**'tur. Dışarıdan iç nesnelere doğrudan erişilemez; her değişiklik root üzerinden geçer.

Amacı:

* İlişkili nesneleri tutarlı bir bütün olarak yönetmek
* İş kurallarını (invariant) tek bir yerde korumak
* Dışarıdan geçersiz bir durumun oluşmasını imkansız kılmak

---

## Problem: Korumasız Nesne Kümesi

Aggregate olmadan, nesnelere dışarıdan serbestçe müdahale edilir ve kurallar hiçbir yerde korunmaz:

```csharp
var siparis = new Siparis();

siparis.Kalemler.Add(new SiparisKalemi(urunId, -5));   // negatif adet! kimse engellemiyor
siparis.Durum = SiparisDurumu.Onaylandi;               // boş sipariş onaylandı! kural yok
```

Sorunlar:

* Adet negatif olabiliyor — geçersiz veri
* Boş bir sipariş onaylanabiliyor — iş kuralı ihlali
* Kurallar nesnenin dışında olduğu için, herhangi biri herhangi bir yerden geçersiz bir duruma sokabilir

İhtiyacımız: bu kuralları nesnenin **içine** taşımak ve dışarıdan doğrudan erişimi kapatmak.

---

## Aggregate Root — Tek Kapı

Aggregate Root, dışarıdan etkileşilen tek nesnedir. İç nesnelere (örneğin sipariş kalemlerine) yalnızca root üzerinden, root'un izin verdiği şekilde ulaşılır. Root, bütün kuralların bekçisidir.

İki temel ilke:

* Dışarıdaki kod yalnızca root'u tutar ve root'un metotlarını çağırır
* İç nesneler salt-okunur açılır; doğrudan değiştirilemez

---

## Örnek: Sipariş Aggregate

`Siparis` aggregate root'tur; `SiparisKalemi` onun içindeki child entity'dir; `Para` ise value object. Kurallar tamamen root'un içinde korunur:

```csharp
public class Siparis   // Aggregate Root
{
    public Guid Id { get; private set; }
    public Guid MusteriId { get; private set; }            // başka aggregate'e ID ile referans
    public SiparisDurumu Durum { get; private set; }

    private readonly List<SiparisKalemi> _kalemler = new();
    public IReadOnlyList<SiparisKalemi> Kalemler => _kalemler.AsReadOnly();   // salt-okunur

    public void KalemEkle(Guid urunId, int adet, Para birimFiyat)
    {
        if (Durum != SiparisDurumu.Taslak)
            throw new InvalidOperationException("Sadece taslak siparişe kalem eklenebilir");
        if (adet <= 0)
            throw new ArgumentException("Adet pozitif olmalı");

        _kalemler.Add(new SiparisKalemi(urunId, adet, birimFiyat));
    }

    public void Onayla()
    {
        if (!_kalemler.Any())
            throw new InvalidOperationException("Boş sipariş onaylanamaz");   // invariant

        Durum = SiparisDurumu.Onaylandi;
    }

    public Para ToplamTutar()
        => _kalemler.Aggregate(new Para(0, "TL"), (acc, k) => acc.Ekle(k.AraToplam()));
}
```

Dikkat edilecek noktalar:

* `_kalemler` **private**; dışarıya yalnızca salt-okunur (`IReadOnlyList`) açılıyor — kimse listeye doğrudan ekleme yapamaz
* Kalem eklemenin tek yolu `KalemEkle`; kural orada korunuyor (adet > 0, sipariş taslak olmalı)
* `Durum`'un setter'ı **private**; sadece `Onayla` gibi metotlarla, kural kontrol edilerek değişiyor
* `Onayla`, "boş sipariş onaylanamaz" invariant'ını koruyor

Child entity ise dışarıdan oluşturulamaz; constructor'ı `internal`'dır:

```csharp
public class SiparisKalemi   // aggregate içindeki child entity
{
    public Guid UrunId { get; private set; }
    public int Adet { get; private set; }
    public Para BirimFiyat { get; private set; }

    internal SiparisKalemi(Guid urunId, int adet, Para birimFiyat)   // sadece aggregate üretir
    {
        UrunId = urunId;
        Adet = adet;
        BirimFiyat = birimFiyat;
    }

    public Para AraToplam() => new Para(BirimFiyat.Tutar * Adet, BirimFiyat.ParaBirimi);
}
```

---

## İnvariant Nedir?

**İnvariant**, aggregate için **her zaman doğru olması gereken** bir iş kuralıdır. "Onaylanmış bir siparişin en az bir kalemi olmalı", "kalem adedi pozitif olmalı" gibi. Aggregate'in asıl görevi, kendisini **hiçbir zaman** geçersiz bir duruma düşürmemektir.

Yukarıdaki örnekte her metot, değişiklik yapmadan önce ilgili invariant'ı kontrol eder. Böylece elinde bir `Siparis` varsa, onun her zaman geçerli olduğundan emin olabilirsin.

---

## Aggregate Sınırı: Küçük Tut

Bir aggregate'e neyi dahil edeceğin önemli bir karardır. Kural: **birlikte tutarlı kalması gereken** nesneleri aynı aggregate'e koy; gerisini koyma. Aggregate ne kadar küçükse o kadar iyidir.

Bunun en önemli sonucu şu: **bir aggregate, başka bir aggregate'i nesne referansıyla değil, ID ile tutar.** `Siparis`, bir `Musteri` nesnesini değil, yalnızca `MusteriId`'sini taşır:

```csharp
public Guid MusteriId { get; private set; }   // Musteri nesnesi DEĞİL, sadece ID
```

Böylece her aggregate kendi tutarlılık sınırının sahibi olur; biri diğerinin iç durumuna karışmaz.

---

## Aggregate ve Repository

Repository, **aggregate düzeyinde** çalışır: tek tek child nesneleri değil, bütün aggregate'i alır ve kaydeder. Her aggregate root'un genelde bir repository'si olur:

```csharp
public interface ISiparisRepository
{
    Siparis Getir(Guid id);      // bütün Siparis aggregate'ini getirir
    void Kaydet(Siparis siparis); // bütün aggregate'i kaydeder
}
```

`SiparisKalemi` için ayrı bir repository olmaz; ona her zaman `Siparis` üzerinden ulaşılır.

---

## Korumasız Nesne Kümesi ve Aggregate Farkı

| Korumasız nesne kümesi              | Aggregate                            |
| ----------------------------------- | ------------------------------------ |
| İç nesnelere doğrudan erişilir       | Sadece root üzerinden erişilir       |
| Kurallar dışarıda ve dağınıktır      | Kurallar root'un içinde korunur      |
| Geçersiz duruma düşebilir            | Her zaman geçerli (invariant) kalır  |
| Diğer nesneleri referansla tutar     | Diğer aggregate'leri ID ile tutar    |

---

## Avantajları

* **İnvariant'lar tek yerde korunur** — kurallar root'un içinde, dağınık değil
* **Tutarlılık sınırı nettir** — neyin birlikte değiştiği bellidir
* **Geçersiz durum imkansızdır** — dışarıdan iç nesnelere doğrudan dokunulamaz
* **Repository ve transaction sınırı netleşir** — aggregate, doğal bir kayıt/işlem birimidir

---

## Dezavantajları ve Dikkat

* **Aggregate'i çok büyük tutmak** — gereğinden fazla nesne dahil etmek performans ve kilitlenme sorunları doğurur
* **Sınırı yanlış çizmek** — birlikte tutarlı olması gerekmeyen şeyleri aynı aggregate'e koymak
* **Tek transaction'da birden çok aggregate değiştirmeye çalışmak** — genel kural, bir transaction'da bir aggregate'tir; aggregate'ler arası tutarlılık genelde domain event'lerle sağlanır

---

## Sonuç

Aggregate, birbirine bağlı Entity ve Value Object'leri tek bir tutarlı birim olarak ele alan DDD yapı taşıdır. Aggregate Root, bu birime açılan tek kapıdır ve bütün iş kurallarının (invariant) bekçisidir.

Özü şudur: iç nesneleri dışarıya kapatmak, değişiklikleri yalnızca root'un metotlarından geçirmek ve böylece aggregate'in **hiçbir zaman** geçersiz bir duruma düşmemesini sağlamak. Diğer aggregate'lere ID ile referans vererek de her birim kendi sınırının sahibi olur.

Doğru çizilmiş bir aggregate, iş kurallarının kod tabanına dağılmasını engeller ve sistemin tutarlılığını tek bir noktada güvence altına alır — bu yüzden DDD'nin kalbi sayılır.
