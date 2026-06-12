# DomainDrivenDesignWithCSharp

> Domain-Driven Design concepts, explanations and real-world examples using C# and .NET.

![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)
![C#](https://img.shields.io/badge/C%23-.NET-512BD4.svg)
![Language](https://img.shields.io/badge/lang-T%C3%BCrk%C3%A7e-orange.svg)

---

## Hakkında

Bu repo, **Domain-Driven Design (DDD)** yaklaşımını C# ile, sade ve örneğe dayalı bir dille adım adım anlatır. DDD'nin amacı, yazılımı kullandığı teknolojinin değil, çözdüğü **iş alanının (domain)** etrafında tasarlamaktır.

Her konu kendi klasöründe, ayrı bir doküman olarak ele alınır ve aynı yapıyı izler.

---

## DDD İki Yarımdan Oluşur

DDD tek bir desen değil, içinde birçok deseni barındıran bir yaklaşımdır. İki katmanı vardır:

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

---

## Başlangıç

Yeni başlıyorsan buradan başla:

* **[Introduction — DDD Nedir?](./Introduction)** — DDD'nin ne olduğu, anemik modelin neden sorun olduğu, stratejik ve taktiksel tasarımın genel görünümü.

---

## Konular

Aşağıdaki başlıklar ayrı dokümanlar olarak ele alınır (yazıldıkça linklenir).

### 🧭 Stratejik Tasarım

| Konu | Açıklama |
| ---- | -------- |
| Ubiquitous Language | Geliştirici ve iş uzmanının aynı dili konuşması; kodun iş dilini birebir yansıtması. |
| Bounded Context | Bir modelin tutarlı olduğu sınır; aynı kelimenin farklı bağlamda farklı anlama gelmesi. |
| Context Map | Bounded context'lerin birbiriyle ilişkisini gösteren harita. |

### 🧱 Taktiksel Tasarım (Yapı Taşları)

| Konu | Açıklama |
| ---- | -------- |
| Entity | Kimliği (Id) olan, eşitliği kimlikle belirlenen nesne. |
| Value Object | Kimliği olmayan, değeriyle eşit ve değişmez (immutable) nesne. |
| Aggregate (+ Root) | Bütün olarak ele alınan nesne kümesi; root tek giriş kapısı, kuralları korur. |
| Repository | Aggregate'leri saklayıp geri getiren soyutlama. |
| Domain Service | Tek bir entity'ye ait olmayan iş mantığı. |
| Domain Event | Domain'de önemli bir şeyin olduğunu bildiren olay. |
| Factory | Karmaşık nesne oluşturma sorumluluğu. |

---

## Temel Ayrım: Entity ve Value Object

DDD'yi anlamanın anahtarı bu ayrımdır:

| Entity                          | Value Object                       |
| ------------------------------- | ---------------------------------- |
| Kimliği (Id) vardır             | Kimliği yoktur                     |
| Eşitlik kimlikle belirlenir     | Eşitlik tüm değerlerle belirlenir  |
| Zamanla değişebilir             | Değişmezdir (immutable)            |
| Örnek: Müşteri, Sipariş         | Örnek: Para, Adres, Tarih aralığı  |

Pratik test: *"İki tanesi aynı değerlere sahipse, bunlar aynı şey midir?"* Cevap **evet** ise value object, **hayır** ise entity.

---

## Her Doküman Neler İçeriyor?

Tüm konular tutarlı bir şablonla yazılmıştır:

* **Nedir ve neden gerekir**
* **Problem** — kavram olmadan kodun nasıl göründüğü
* **Örnek kod** — gerçek bir domain (Sipariş) üzerinden
* **Karşılaştırma** — yakın kavramlarla farkı
* **Ne zaman kullanmalı / kullanmamalı**
* **Sonuç**

---

## Ne Zaman DDD?

DDD güçlüdür ama bedeli vardır. **Karmaşık, kural yoğun ve uzun ömürlü** domain'lerde kazandırır. Basit CRUD veya veri-merkezli sistemlerde ise katmanlı/transaction-script tarzı bir mimari çoğu zaman daha sade ve yeterlidir. DDD'yi, karmaşıklık onu hak ettiğinde kullan.

---

## Lisans

Bu proje [MIT Lisansı](./LICENSE) ile lisanslanmıştır.
