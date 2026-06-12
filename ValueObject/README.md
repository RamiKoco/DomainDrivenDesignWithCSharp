# Value Object Nedir?

## Giriş

DDD'nin en faydalı ama en az kullanılan yapı taşlarından biri **Value Object** (değer nesnesi)'dir. Giriş yazısında kısaca değinmiştik; burada derinleşiyoruz.

Çoğu kod, "para", "e-posta", "tarih aralığı" gibi anlamlı kavramları `decimal`, `string` gibi ham tiplerle taşır. Value Object, bu kavramları kendi tipleri hâline getirerek koda anlam, güvenlik ve davranış kazandırır.

---

## Value Object Nedir?

Value Object, **kimliği olmayan**, yalnızca **değeriyle** anlam taşıyan bir nesnedir. İki value object, tüm değerleri aynıysa eşittir — "hangisi" diye sormak anlamsızdır.

100 TL ile 100 TL birbirinden ayırt edilemez. Bir adres ile aynı bilgileri taşıyan başka bir adres, aynı adrestir. İşte bunlar value object'tir.

Üç temel özelliği vardır:

* **Kimliksizdir** — eşitlik değerle belirlenir, Id ile değil
* **Değişmezdir (immutable)** — bir kez oluşturulunca değiştirilemez
* **Kendini doğrular** — geçersiz bir value object hiç oluşturulamaz

---

## Problem: Primitive Obsession

İş kavramlarını ham tiplerle (primitive) temsil etmeye **primitive obsession** denir. Sık görülür ve sinsi sorunlar doğurur:

```csharp
public class Siparis
{
    public decimal Tutar { get; set; }     // hangi para birimi? negatif olabilir mi?
    public string Eposta { get; set; }     // geçerli format mı? her yerde tekrar kontrol
}
```

Sorunlar:

* `Tutar` bir `decimal` — para birimi bilgisi yok, negatif değer atanabilir
* `Eposta` bir `string` — geçerli olup olmadığını her kullanan kodun ayrı ayrı kontrol etmesi gerekir
* Doğrulama kuralları kod tabanına dağılır; biri unutulursa geçersiz veri sisteme sızar

Value Object bu kavramları kendi tipine taşıyarak sorunu kökten çözer.

---

## Örnek: Para Value Object

Bir `Para` value object'i kuralım. Dikkat: property'ler salt-okunur (değişmez), constructor doğrular (geçersiz Para oluşamaz):

```csharp
public class Para
{
    public decimal Tutar { get; }
    public string ParaBirimi { get; }

    public Para(decimal tutar, string paraBirimi)
    {
        if (tutar < 0)
            throw new ArgumentException("Tutar negatif olamaz");
        if (string.IsNullOrWhiteSpace(paraBirimi))
            throw new ArgumentException("Para birimi zorunlu");

        Tutar = tutar;
        ParaBirimi = paraBirimi;
    }

    // Değer eşitliği — iki Para, tutar ve birimi aynıysa eşittir
    public override bool Equals(object obj)
        => obj is Para p && Tutar == p.Tutar && ParaBirimi == p.ParaBirimi;

    public override int GetHashCode() => HashCode.Combine(Tutar, ParaBirimi);
}
```

Artık `Para`, sadece bir sayı değil; para birimini taşıyan, negatif olamayan, kendini doğrulayan bir kavram.

---

## C#'ta record ile

Yukarıdaki `Equals`/`GetHashCode` kodunu elle yazmak yorucu. C#'ta `record`, **değer eşitliğini ve değişmezliği otomatik** verir — value object için biçilmiş kaftandır:

```csharp
public record Para(decimal Tutar, string ParaBirimi);
```

Bu tek satır, değer eşitliğini hazır getirir:

```csharp
var a = new Para(100, "TL");
var b = new Para(100, "TL");

Console.WriteLine(a == b);   // True  (değer eşitliği otomatik)
```

Doğrulama eklemek istersen, record'a açık bir constructor verirsin (yine değer eşitliği record'dan gelir):

```csharp
public record Para
{
    public decimal Tutar { get; }
    public string ParaBirimi { get; }

    public Para(decimal tutar, string paraBirimi)
    {
        if (tutar < 0) throw new ArgumentException("Tutar negatif olamaz");
        Tutar = tutar;
        ParaBirimi = paraBirimi;
    }
}
```

---

## Değer Eşitliği Nasıl Çalışır?

Normal bir `class`'ta eşitlik **referansa** dayanır: iki ayrı nesne, aynı değerlere sahip olsa bile eşit sayılmaz. Value object'te ise eşitlik **değere** dayanmalıdır.

* `record` kullanırsan, derleyici `Equals` ve `GetHashCode`'u senin yerine yazar → değer eşitliği bedava gelir.
* `class` kullanırsan, bu ikisini elle override etmen gerekir (yukarıdaki `Para` örneğindeki gibi).

Bu yüzden modern C#'ta value object'ler genelde `record` ile yazılır.

---

## Davranış ve Değişmezlik

Value object yalnızca veri tutmaz; **davranış da taşıyabilir.** Ama değişmez olduğu için, bir işlem nesneyi değiştirmez — **yeni bir nesne döndürür:**

```csharp
public record Para(decimal Tutar, string ParaBirimi)
{
    public Para Ekle(Para other)
    {
        if (other.ParaBirimi != ParaBirimi)
            throw new InvalidOperationException("Farklı para birimleri toplanamaz");

        return this with { Tutar = Tutar + other.Tutar };   // yeni Para üretir
    }
}
```

```csharp
var fiyat = new Para(100, "TL");
var toplam = fiyat.Ekle(new Para(50, "TL"));   // toplam = 150 TL, fiyat hâlâ 100 TL
```

`fiyat` değişmedi; `Ekle` yeni bir `Para` verdi. Değişmezlik, value object'i öngörülebilir ve yan etkisiz kılar. Üstelik "farklı para birimleri toplanamaz" gibi bir kural da burada, kavramın kendi içinde korunur.

---

## Ham Tip ve Value Object Arasındaki Fark

| Ham tip (decimal / string)        | Value Object (Para / Eposta)        |
| --------------------------------- | ----------------------------------- |
| Anlam taşımaz, sadece sayı/metin  | Bir iş kavramını temsil eder        |
| Doğrulama her kullanımda tekrarlar| Doğrulama tek yerde (constructor)   |
| Geçersiz değer tutabilir          | Geçersiz durum oluşturulamaz        |
| Davranış eklenemez                | Kendi davranışını taşır (Ekle vb.)  |

---

## EF ile Saklama: Owned Types

Value object'lerin kendi kimliği olmadığı için veritabanında ayrı bir tabloya değil, sahibinin tablosuna yazılması gerekir. EF Core'da bu, **owned type** ile yapılır:

```csharp
modelBuilder.Entity<Siparis>().OwnsOne(s => s.ToplamTutar);
```

Yani EF makalesinde gördüğün owned types, tam olarak value object'leri saklamak içindir. (Senin `SqlServerCache`'indeki `CacheItem` da `OwnsOne(...).ToJson()` ile böyle saklanıyordu.)

---

## Avantajları

* **Anlamı koda taşır** — `decimal` yerine `Para`, `string` yerine `Eposta`
* **Doğrulamayı tek yere toplar** — geçersiz bir değer hiç oluşturulamaz
* **Geçersiz durumu imkansız kılar** — değişmez + doğrulanmış
* **Yan etkisizdir** — değişmez olduğu için test etmesi ve akıl yürütmesi kolaydır
* **Primitive obsession'ı önler** — kavramlar kendi tipine kavuşur

---

## Dezavantajları

* **Küçük kavramlar için aşırıya kaçılabilir** — her `string`'i value object yapmak gerekmez
* **EF ile saklama ek konfigürasyon ister** — owned type / value converter ayarı gerekir
* **Daha fazla tip** — kod tabanında sınıf sayısı artar (karşılığında her biri anlamlıdır)

---

## Sonuç

Value Object, kimliği olmayan, değeriyle eşit, değişmez ve kendini doğrulayan bir DDD yapı taşıdır. Ham tiplerle temsil edilen iş kavramlarını (para, e-posta, adres) kendi tiplerine taşıyarak koda anlam ve güvenlik kazandırır.

C#'ta `record`, değer eşitliği ve değişmezliği hazır verdiği için value object yazmanın en doğal yoludur. Davranış eklerken nesneyi değiştirmek yerine yeni bir nesne döndürmek, değişmezliği korur.

Primitive obsession'ın sızdırdığı geçersiz durumları kökten engellediği için, Value Object çoğu zaman bir DDD projesinde en hızlı kazanç sağlayan yapı taşıdır.
