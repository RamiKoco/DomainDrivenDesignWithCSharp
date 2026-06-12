# Entity Nedir?

## Giriş

**Entity** (varlık), Value Object'in ikizidir. Value Object'i değeriyle tanımlıyorduk; Entity'yi ise **kimliğiyle (identity)** tanımlarız. Bu ayrım DDD'nin kalbidir.

Bazı nesneler zaman içinde takip edilmeli ve değişebilmelidir. Adı, adresi değişen bir müşteri hâlâ aynı müşteridir — onu özellikleri değil, kimliği "aynı" yapar. İşte Entity budur.

---

## Entity Nedir?

Entity, **benzersiz bir kimliği (Id)** olan nesnedir. İki entity'nin eşitliği değerleriyle değil, **kimlikleriyle** belirlenir. Özellikleri zamanla değişse bile, kimliği aynı kaldığı sürece o hâlâ aynı entity'dir.

Üç temel özelliği:

* **Kimliği vardır** — her entity'nin benzersiz bir Id'si olur
* **Eşitlik kimlikle belirlenir** — değerler değil, Id karşılaştırılır
* **Zamanla değişebilir** — özellikleri evrilir, ama kimliği sabit kalır

---

## Problem: Neden Kimlik Gerekir?

Bir müşteriyi düşün. Onu değerleriyle ayırt etmeye çalışırsak sorun çıkar:

```csharp
var m1 = new Musteri("Ahmet Yılmaz", "İstanbul");
var m2 = new Musteri("Ahmet Yılmaz", "İstanbul");
// Bunlar aynı kişi mi? Aynı ada ve şehre sahip iki FARKLI müşteri olabilir.
```

Aynı ada sahip iki ayrı müşteri olabilir; ya da bir müşterinin adı/adresi değişebilir ama o yine aynı kişidir. Değerler kimlik için yeterli değildir. Bu yüzden entity'ye, değerlerinden bağımsız, **kalıcı bir kimlik** veririz.

---

## Kimlik (Identity)

Her entity'nin, onu benzersiz kılan bir `Id`'si vardır. Bu kimlik, nesnenin "bu spesifik olan" olmasını sağlar:

```csharp
public class Musteri
{
    public Guid Id { get; private set; }     // kimlik — değişmez
    public string Ad { get; private set; }
    public Adres Adres { get; private set; } // Adres bir value object

    public Musteri(string ad, Adres adres)
    {
        Id = Guid.NewGuid();
        Ad = ad;
        Adres = adres;
    }
}
```

Kimliğin **değişmemesi** kritiktir; bir entity'nin Id'sini değiştirirsen, o artık başka bir entity olur.

---

## Eşitlik: Kimlikle, Değerle Değil

İki entity, **Id'leri aynıysa** eşittir — diğer özellikleri farklı olsa bile. Id'leri farklıysa, tüm değerleri aynı olsa bile eşit değildir:

```csharp
public override bool Equals(object obj)
    => obj is Musteri m && Id == m.Id;       // sadece Id karşılaştırılır

public override int GetHashCode() => Id.GetHashCode();
```

Bu, Value Object'in tam tersidir: orada eşitlik tüm değerlere bakardı, burada yalnızca kimliğe bakar.

---

## Zamanla Değişim (Mutability)

Value Object değişmezdi (immutable); Entity ise **değişebilir.** Ama bu değişim başıboş değildir — kontrollü metotlardan geçer ve kurallar korunur. Kimlik sabit kalır, özellikler evrilir:

```csharp
public void AdresDegistir(Adres yeniAdres)
{
    Adres = yeniAdres;   // adres değişti, ama Id aynı → hâlâ aynı müşteri
}
```

```csharp
var musteri = new Musteri("Ahmet", istanbulAdres);
musteri.AdresDegistir(ankaraAdres);
// Aynı müşteri, yeni adres. Kimlik değişmedi.
```

---

## Entity ve Value Object Arasındaki Fark

DDD'nin en temel ayrımı budur:

| Entity                          | Value Object                       |
| ------------------------------- | ---------------------------------- |
| Kimliği (Id) vardır             | Kimliği yoktur                     |
| Eşitlik kimlikle belirlenir     | Eşitlik tüm değerlerle belirlenir  |
| Zamanla değişebilir (mutable)   | Değişmezdir (immutable)            |
| "Bu spesifik olan"              | "Bu değere sahip olan"             |
| Örnek: Müşteri, Sipariş         | Örnek: Para, Adres, Tarih aralığı  |

Pratik test: *"İki tanesi aynı değerlere sahipse, bunlar aynı şey midir?"* Cevap **hayır** ise (ayrı kimlikleri olabilir) → Entity. Cevap **evet** ise → Value Object.

---

## Kimlik Nasıl Atanır?

Entity'ye kimlik atamanın birkaç yolu vardır:

| Strateji            | Nasıl                          | Not                                            |
| ------------------- | ------------------------------ | ---------------------------------------------- |
| **Guid**            | Uygulamada (`Guid.NewGuid()`)  | Nesne, veritabanına gitmeden kimliğe kavuşur   |
| **DB tarafından**   | Veritabanı üretir (auto-increment) | Insert'e kadar Id bilinmez (0 / boş)       |
| **Doğal anahtar**   | İş dünyasından (TC, ISBN)      | Asla değişmemesi şartıyla kullanılabilir       |

`Guid` çoğu zaman en esnek seçenektir; entity'yi daha oluşturulurken kimliklendirir, böylece kayıt edilmeden önce de onunla çalışabilirsin.

---

## Entity ve Aggregate

Entity'ler aggregate'lerin içinde yaşar. Aslında bir **Aggregate Root'un kendisi de bir entity'dir** — hem de en önemlisi. Aggregate yazısındaki `Siparis` bir entity (kimliği var, zamanla değişir); içindeki `SiparisKalemi` de bir child entity'dir ama ona her zaman root üzerinden ulaşılır.

Yani Entity, DDD'nin temel kimlik taşıyıcısıdır; Aggregate ise bu entity'leri kurallarıyla bir arada tutan yapıdır.

---

## Avantajları

* **Kimlik üzerinden takip** — bir nesneyi zaman içinde, değerleri değişse de izleyebilirsin
* **Doğru eşitlik** — iki kaydın "aynı" olup olmadığı net biçimde kimlikle belirlenir
* **Kontrollü değişim** — özellikler metotlar üzerinden, kurallar korunarak evrilir

---

## Dikkat Edilecek Noktalar

* **Eşitliği değerle yazmak yanlıştır** — entity eşitliği yalnızca Id'ye bakmalı
* **Kimliği değiştirilebilir yapmak yanlıştır** — Id bir kez atanır, sonra sabit kalmalı (`private set`)
* **Her nesneyi entity yapmaya çalışmak** — kimliğe ihtiyaç duymayan kavramlar (para, adres) value object olmalı, entity değil

---

## Sonuç

Entity, benzersiz bir kimliği olan, eşitliği bu kimlikle belirlenen ve zamanla değişebilen bir DDD yapı taşıdır. Value Object'in tam karşıtıdır: biri değeriyle, diğeri kimliğiyle tanımlanır.

Bir entity'nin özü, değerleri ne kadar değişirse değişsin "aynı" kalmasıdır — bunu sağlayan da sabit kimliğidir. Değişimi kontrollü metotlardan geçirmek, bu değişim sırasında iş kurallarının korunmasını sağlar.

"Bu spesifik olan mı, yoksa bu değere sahip olan mı?" sorusu, bir kavramı Entity mi yoksa Value Object mi yapacağını belirleyen en pratik pusuladır.
