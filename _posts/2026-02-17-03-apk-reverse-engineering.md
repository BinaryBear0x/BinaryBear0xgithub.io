---
layout: post
title: "Bir Kulaklığın Anatomisi - Bölüm 3: Mobil Uygulama Reverse Engineering"
categories: [reverse-engineering, mobile-security]
date: 2026-02-17 12:00:00 +0300
tags: [android, jadx, apk, reversing]
toc: true
image:
  path: /assets/img/headset-reverse/part3-img01.png
  alt: JADX Analysis
---

# Mobil Uygulama Reverse Engineering

Bu noktadan itibaren analiz, cihaz tarafındaki gözlemleri doğrulamak ve BLE üzerindeki olay/komut akışını uygulama mantığıyla ilişkilendirmek amacıyla vendor mobil uygulamasının tersine mühendisliğine genişletilmektedir.

Çalışmada referans olarak kullanılan v2.1.0 sürümlü APK dosyası, statik analiz için **JADX** ortamına aktarılmış ve uygulama içindeki BLE etkileşimleri (servis/karakteristik erişimleri, payload formatları ve olay işleme mantığı) incelenmeye başlanmıştır.

Bu bölümün amacı; cihaz üzerinde gözlemlenen event akışını, uygulama tarafındaki kod yolu ve veri formatlarıyla kanıta dayalı biçimde eşleştirmek; böylece “gözlem → yorum” çizgisini “gözlem → uygulama doğrulaması” seviyesine taşımaktır.

> [!NOTE]
> **Not:** Bu bölümde yer alan paket adları ve tanımlayıcılar, çalışmanın teknik bütünlüğünü korumak amacıyla kısmen anonimleştirilmiştir.

## JADX ile Başlangıç

JADX ortamını açıp APK’yı içe aktardıktan sonra, daha önce cihaz üzerinde gözlemlediğimiz ANC ile ilişkili characteristic UUID’sini kod içinde arayarak başlangıç noktasını belirleyeceğiz. Bunun için global aramayı açıyoruz (`Ctrl + Shift + F`).

Arama kutusuna, sahada yakaladığımız UUID’yi (ör. `0000aa20` veya tam 128-bit formu) girerek uygulama içerisinde geçtiği sınıf ve metotları tespit ediyoruz.

> [!TIP]
> **İpucu:** Direkt tam UUID aratmak yerine `aa20` şeklinde arama yaparak daha net sonuçlar elde edilebilir.

![JADX Arama Ekranı](/assets/img/headset-reverse/part3-img01.png)

![JADX Arama Sonuçları](/assets/img/headset-reverse/part3-img02.png)

`0000aa20` UUID’sinin geçtiği yerde `ANC_CONFIGURATION` ve `BLE` anahtar kelimelerinin görülmesi, bu karakteristiğin uygulama tarafında ANC konfigürasyonu ile ilişkilendirildiğini doğrulamaktadır. Bu bulgu, cihaz gözlemleri ile APK kod yolu arasında doğrudan bir bağ kurmamızı sağlar.

Bu aşamada, `com.[XXXXX].bleprotocol` paketi altındaki ilgili class üzerinden BLE kontrol akışı analiz edilmektedir.

![Kod Analizi](/assets/img/headset-reverse/part3-img03.png)

Görülebileceği üzere, uygulama içindeki `BLECharacteristic` tanımları ile GATT envanterinin birleştirilmesi sonucunda, uzun süredir tespit etmeye çalıştığımız GATT haritası anlamlı bir şekilde ortaya çıkmaktadır. Bu noktadan sonra hedef, bu karakteristikleri davranışlarına göre sınıflandırmak ve hangi işlevleri temsil ettiklerini kanıta dayalı biçimde notlandırmaktır.

### GATT Profili ve İşlev Eşleşmeleri

Aşağıdaki tablo, JADX analizi sonucu doğrulanan servis ve işlevleri özetlemektedir:

| Servis | UUID | Örnek Karakteristikler | İşlev |
| :--- | :--- | :--- | :--- |
| **Device Information** | 0x180A | 0x2A29 / 0x2A24 / ... | Üretici, model, seri/firmware bilgileri (salt-okunur) |
| **Battery Service** | 0x180F | 0x2A19 | Batarya seviyesi (%) |
| **Generic Access** | 0x1800 | 0x2A00 | Cihazın görünen adı |
| **Kritik Kontrol Servisi** | 0xAA00 (Vendor) | 0xAA20 | ANC Kontrolü |

Diğer özel servisler:
- **Google Fast Pair (0xFE2C):** Android cihazlarla hızlı eşleşme akışını destekleyen servis ailesidir.
- **Airoha Proprietary (5052494d…):** Vendor-proprietary bir servis olup, üreticiye özgü düşük seviyeli yönetim işlevleri için kullanılan bir protokol yüzeyi olabilir.

### Mimari Özet ve Uygulama Geliştirme Notu

Elde edilen bulgular, cihazın genel olarak **state-driven** (durum odaklı) bir kontrol modeli benimsediğini göstermektedir:
1. **Girdi (Input):** Tek tıklama (single tap) gibi bazı kullanıcı etkileşimleri cihaz firmware’i içerisinde yorumlanır ve BLE tarafında her zaman ayrı bir “olay” olarak görünmeyebilir.
2. **Kontrol:** ANC gibi bazı işlevlerin, belirli bir karakteristiğe yazma yapılarak (örn. `aa20`) değiştirilebildiği gözlemlenmiştir.
3. **Olay (Event):** Uzun basma gibi etkileşimler, notify tabanlı bir event kanalında tek bayt veya kısa frame’ler halinde gözlemlenebilir.

## Kod Obfuscation ve Analiz

Daha önce doğrulanan ANC konfigürasyon UUID’si üzerinden JADX içinde *Find Usage* yaparak, bu karakteristiğin uygulama tarafındaki kullanım noktalarını incelemeye başlıyoruz.

Uygulama kodunda sınıf adları seviyesinde belirgin bir obfuscation (ad gizleme) gözlemlenmektedir. Bu tür isimlendirme kalıpları (`a.a.a`, `n7.a.a` vb.) genellikle R8/DexGuard benzeri araçlarla yapılan küçültme/obfuscation süreçlerinin bir sonucudur.

Kullanılan paketleyici/koruma katmanına dair göstergeler, **Detect It Easy (DIE)** gibi araçlarla yapılan statik incelemeyle desteklenebilir.

![Detect It Easy Analizi](/assets/img/headset-reverse/part3-img04.png)

![Detect It Easy Sonuçları](/assets/img/headset-reverse/part3-img05.png)

Bu durum analizi zorlaştırsa da, yapısal olarak bir engel değildir; çünkü BLE ile ilgili sınıflar çoğu zaman sabit stringler (UUID’ler), alan adları veya çağrı zincirleri üzerinden izlenebilir.

## Veri Yapıları ve Enum Analizi

### Cihazdan Uygulamaya: ANC Durumunun Çözülmesi (Read / Notify Yolu)

Cihaz tarafından uygulamaya gönderilen veri, aşağıdaki mantıkla işlenmektedir:
1. BLE karakteristiğinden gelen `byte[]` veri alınır.
2. Veri uzunluğu 1 byte değilse, durum geçersiz kabul edilir.
3. Geçersiz veya tanımsız durumda uygulama, güvenli bir varsayılan olarak `PLAYBACK_ONLY` moduna düşer.

Bu yaklaşım, bilinmeyen veya yeni firmware değerlerinin uygulamayı kırmaması için bilinçli bir tasarım tercihidir.

### Uygulamadan Cihaza: ANC Komutunun Kodlanması (Write Yolu)

Kullanıcı arayüzü üzerinden bir ANC modu seçildiğinde, ters yönde bir işlem gerçekleşir:
1. Seçilen ANC durumu, `AncConfigData` tipiyle temsil edilir.
2. Bu tip, kendi içinde sayısal bir karşılığa sahiptir.
3. Uygulama, bu sayısal değeri tek baytlık bir dizi hâline getirerek ilgili BLE karakteristiğine yazar.

> [!NOTE]
> **Soru:** Bu tek baytlık değerlerin (0, 1, 2, …) hangi ANC modlarına karşılık geldiği ve bu eşlemenin nasıl tanımlandığı?

Bu sorunun yanıtı, `AncConfigData` sınıfında aranacaktır.

![AncConfigData Enum](/assets/img/headset-reverse/part3-img06.png)

`AncConfigData` içinde yapılan analiz sonucunda ANC modlarının sayısal karşılıkları açık biçimde tanımlanmış durumdadır:
- **PLAYBACK_ONLY (0)** → Kapalı
- **CANCELLING (1)** → ANC Açık
- **TRANSPARENCY (2)** → Şeffaflık Modu

### Cihaz Ayrımı (Sağ/Sol Kulaklık)

`k7.a` sınıfı araştırmasında, uygulamanın cihazdan gelen verinin hangi parçaya ait olduğunu `instanceId` üzerinden ayırt ettiği görülmüştür.

![k7.a Sınıfı](/assets/img/headset-reverse/part3-img07.png)

| Cihaz Parçası | Tam UUID Adresi | Değişken Adı |
| :--- | :--- | :--- |
| **Sağ Kulaklık** | `...63c33fc5d401` | `f41419i` (RIGHT) |
| **Sol Kulaklık** | `...63c33fc5d402` | `f41420j` (LEFT) |
| **Şarj Kutusu** | `...63c33fc5d403` | `f41421k` (MAIN_CASE) |

## UUID Üretim Mantığı ve GattHandler

Analiz sırasında, `aa20` gibi kısa UUID'lerin uygulamanın merkezinde nasıl tam 128-bit UUID'ye dönüştürüldüğü incelenmiştir.

![UUID Helper](/assets/img/headset-reverse/part3-img08.png)

Bu aşamada karşılaşılan yardımcı sınıf, UUID üretim mantığını `GattHandler` üzerine yıkar. `GattHandler`, uygulamanın Bluetooth tarafındaki tüm servis ve karakteristik UUID’lerini tek bir yerden üreten ve yöneten ana bileşendir.

![GattHandler Sınıfı](/assets/img/headset-reverse/part3-img09.png)

`GattHandler` sınıfı, üç farklı UUID üretim yolu tanımlar:

1. **Standart / Legacy UUID (a):** Bluetooth SIG tabanlı (`0000...-0000-1000-8000-00805f9b34fb`). Eski tip karakteristikler için.
2. **Vendor-Spesifik / Modern UUID (b):** Üreticiye özgü Base UUID (`0000... + g.f37787b`). Modern protokol için.
3. **OTA UUID (c):** Firmware güncelleme için (`...d102-11e1-9b23-00025b00a5a5`).

### Gizli Base UUID'nin Keşfi

Vendor’a özgü UUID üretiminde kullanılan `g.f37787b` değişkeninin içeriği, `g` sınıfında bulunmuştur.

![g Sınıfı ve Base UUID](/assets/img/headset-reverse/part3-img10.png)

```java
public static final String f37787b = "-1337-1dea-feed-c0ffee70c0de";
```

Bu "Magic String" (`1337-1dea-feed-c0ffee70c0de`), modern protokolün tam adresini oluşturmak için kullanılan anahtardır.

Bu "leet speak" (leetspeak) tarzı tanımlama, geliştiricilerin kod içine bıraktığı imzayı açıkça göstermektedir.

Bu verilerle artık tam bir iletişim haritası çıkarılmıştır:

| Fonksiyon | Tam UUID Adresi | Değerler |
| :--- | :--- | :--- |
| **Modern ANC Kontrol** | `00000013-1337-1dea-feed-c0ffee70c0de` | 00, 01, 02 |
| **Legacy ANC Kontrol** | `0000aa20-0000-1000-8000-00805f9b34fb` | 00, 01, 02 |
| **Düğme Olayları** | `0000aa13-0000-1000-8000-00805f9b34fb` | Notify |

Bu bölümle birlikte, cihazın kontrol protokolünü tamamen reverse engineer etmiş bulunuyoruz. Bir sonraki ve son bölümde, firmware güncelleme mekanizmasını inceleyeceğiz.
