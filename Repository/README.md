# Repository Nedir?

## Giriş

Önceki yazılarda Repository'ye birkaç kez değindik; özellikle Aggregate'in "kaydedilip getirilen birim" olduğunu söylerken. Şimdi onu kendi başına ele alalım.

**Repository**, domain ile veri saklama (persistence) katmanı arasındaki köprüdür. Amacı, domain'in veritabanını hiç tanımamasını sağlamaktır: domain yalnızca "şu aggregate'i getir / kaydet" der, bunun SQL mi, EF mi, başka bir şey mi olduğunu bilmez.

---

## Repository Nedir?

Repository, **aggregate'leri saklamak ve geri getirmek** için kullanılan bir soyutlamadır. Domain'e, sanki bellekteki bir aggregate koleksiyonuymuş gibi görünür: "ekle", "getir", "kaydet".

İki temel niteliği vardır:

* **Aggregate düzeyinde çalışır** — tek tek child nesneleri değil, bütün aggregate'i alır/verir
* **Domain'i persistence'tan ayırır** — veri erişiminin detayları domain'e sızmaz

---

## Problem: Domain'in Veritabanını Tanıması

Repository olmadan, iş mantığı doğrudan veritabanı koduna bağlanır:

```csharp
public class SiparisServisi
{
    public void Onayla(Guid id)
    {
        using var ctx = new SiparisDbContext();        // EF detayı domain'e sızdı
        var siparis = ctx.Siparisler.Find(id);
        siparis.Onayla();
        ctx.SaveChanges();
    }
}
```

Sorunlar:

* İş mantığı, EF ve `DbContext` gibi altyapı detaylarını tanımak zorunda kalıyor
* Test etmek zorlaşıyor — gerçek bir veritabanı olmadan çalıştırılamıyor
* Veri erişimini değiştirmek (EF'ten başka bir şeye geçmek) domain'i de etkiliyor

İhtiyacımız: domain'in "nasıl saklandığını" değil, yalnızca "ne yapmak istediğini" bilmesi.

---

## Repository Soyutlaması

Çözüm, **domain katmanında bir arayüz** tanımlamaktır. Domain yalnızca bu arayüzü tanır:

```csharp
// Domain katmanında — sadece sözleşme
public interface ISiparisRepository
{
    Siparis Getir(Guid id);
    void Ekle(Siparis siparis);
    void Kaydet(Siparis siparis);
}
```

Bunun **uygulaması (implementation)** ise altyapı (infrastructure) katmanında yaşar. Domain, arkasında EF mi yoksa başka bir şey mi olduğunu bilmez:

```csharp
public class SiparisServisi
{
    private readonly ISiparisRepository _repository;
    public SiparisServisi(ISiparisRepository repository) => _repository = repository;

    public void Onayla(Guid id)
    {
        var siparis = _repository.Getir(id);
        siparis.Onayla();
        _repository.Kaydet(siparis);
    }
}
```

Artık domain temiz: ne EF görüyor, ne `DbContext`.

---

## Aggregate Düzeyinde Çalışır

Önemli bir kural: **her aggregate root'un bir repository'si olur**, child entity'lerin değil. `Siparis` için `ISiparisRepository` vardır, ama `SiparisKalemi` için ayrı bir repository **yoktur** — ona her zaman `Siparis` üzerinden ulaşılır.

Ayrıca repository, aggregate'i **bütün olarak** getirir; yani siparişi getirirken kalemlerini de getirir:

```csharp
public class SiparisRepository : ISiparisRepository
{
    private readonly SiparisDbContext _context;
    public SiparisRepository(SiparisDbContext context) => _context = context;

    public Siparis Getir(Guid id)
        => _context.Siparisler
            .Include(s => s.Kalemler)        // aggregate'i bütün getir (eager loading)
            .FirstOrDefault(s => s.Id == id);

    public void Ekle(Siparis siparis) => _context.Siparisler.Add(siparis);
    public void Kaydet(Siparis siparis) => _context.SaveChanges();
}
```

Buradaki `Include`, EF makalesinde gördüğümüz eager loading'tir — aggregate'i parçalı değil, tam getirmek için.

---

## Önemli Ayrım: Generic Repository vs DDD Repository

Sık karşılaşılan bir tuzak var. Bazı projelerde her entity için aynı tip CRUD sunan bir `Repository<T>` yazılır:

```csharp
public interface IRepository<T>
{
    T GetById(int id);
    void Add(T entity);
    void Delete(T entity);
}
```

Bu genelde **gereksizdir**, çünkü EF Core'un `DbSet<T>`'i zaten bir repository gibi davranır — onu ince bir sınıfla sarmalamak değer katmaz.

DDD'deki repository farklıdır: **aggregate root'a özeldir** ve domain için anlamlı sorgular sunar:

```csharp
public interface ISiparisRepository
{
    Siparis Getir(Guid id);
    IReadOnlyList<Siparis> MusteryeGoreGetir(Guid musteriId);   // domain anlamlı sorgu
    void Ekle(Siparis siparis);
}
```

Yani repository'yi her şeye genel bir CRUD katmanı olarak değil, **aggregate'leri domain dilinde saklayan/getiren** bir soyutlama olarak kullan.

---

## Doğrudan Veri Erişimi ve Repository Farkı

| Doğrudan veri erişimi (EF/SQL)       | Repository                            |
| ------------------------------------ | ------------------------------------- |
| Domain, veritabanını tanır           | Domain yalnızca arayüzü tanır         |
| İş mantığı altyapıya bağlıdır        | İş mantığı persistence'tan ayrıdır    |
| Test etmek zordur (gerçek DB gerekir)| Sahte (mock) repository ile test kolay|
| Veri erişimini değiştirmek domain'i etkiler | Sadece implementasyon değişir   |

---

## Avantajları

* **Domain'i persistence'tan ayırır** — iş mantığı EF/SQL'den habersiz kalır
* **Test edilebilirliği artırır** — arayüz sahte bir uygulamayla değiştirilebilir
* **Aggregate sınırını korur** — veri her zaman bütün aggregate olarak alınır/verilir
* **Domain dilini yansıtır** — `MusteryeGoreGetir` gibi anlamlı sorgular

---

## Dezavantajları ve Dikkat

* **Gereksiz generic katman** — `DbSet`'i saran ince bir `Repository<T>` çoğu zaman değer katmaz
* **Aşırı soyutlama** — basit projelerde doğrudan `DbContext` kullanmak yeterli olabilir
* **Yanlış granülerlik** — child entity'lere ayrı repository açmak aggregate fikrini bozar

---

## Sonuç

Repository, aggregate'leri saklamak ve geri getirmek için kullanılan, domain'i veri erişiminden ayıran bir DDD soyutlamasıdır. Domain yalnızca arayüzü tanır; veritabanı detayları altyapı katmanında kalır.

Doğru kullanımı **aggregate düzeyindedir**: her aggregate root'un bir repository'si olur, aggregate bütün olarak alınıp verilir ve sorgular domain dilini yansıtır. Her entity için genel bir CRUD sarmalayıcısı yazmak ise genelde gereksizdir; çünkü EF'in `DbSet`'i bu işi zaten görür.

Repository, domain'in "nasıl saklandığını" değil "ne yapmak istediğini" konuşmasını sağlayarak, iş mantığını temiz ve test edilebilir tutar.
