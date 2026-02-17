---
layout: post
title: "Bir Kulaklığın Anatomisi - Bölüm 4: Firmware Dağıtım Kanalının Analizi"
categories: [reverse-engineering, mobile-security]
date: 2026-02-17 12:00:00 +0300
tags: [pki, firmware, aws, flutter]
toc: true
image:
  path: /assets/img/headset-reverse/part4-img01.png
  alt: Firmware Analysis
---

# Firmware Dağıtım Kanalının Analizi

Bu bölümde, mobil uygulamanın firmware güncelleme sürecini nasıl yönettiğine odaklanacağız. Özellikle uygulamanın hangi güncelleme uç noktalarıyla iletişim kurduğu, firmware paketlerini nasıl tanımladığı ve bu dosyaları hangi indirme kanalı üzerinden temin ettiği incelenecektir.

## Dağıtım Altyapısı ve Yetkilendirme

Firmware indirme sürecinde kullanılan istemci, gerekli kimlik ve yetkilendirme bilgilerini uygulama içindeki merkezi bir yapıdan dinamik olarak almaktadır. Bu bilgiler, sabit olarak kodlanmak yerine, konfigürasyon sağlayıcı bir nesne üzerinden (`a.f35480a.a()`) elde edilmektedir.

Firmware güncelleme kanalının analizi, uygulama içinde “firmware” referanslarının izlenmesiyle başlatılmış ve güncelleme sürecinde kullanılan kritik kimlik bilgisinin düz metin olarak değil, obfuske edilmiş bir `byte[]` dizisi halinde saklandığı tespit edilmiştir.

Bu dizinin çalışma anında çözümlendiği metot zinciri takip edildiğinde:
1. Dönüşümün standart kriptografik API’ler kullanılarak gerçekleştirildiği;
2. Anahtar materyali ve başlangıç vektörünün uygulama içindeki sabitlerden üretildiği;
3. Çözümleme sonrası çıktının ek bir Base64 decode adımından geçirilerek okunabilir metne dönüştürüldüğü görülmüştür.

> [!CAUTION]
> **Güvenlik Notu:** Analiz sırasında, decompiler’ın sabit bayt dizilerini bazı durumlarda hatalı temsil edebildiği fark edilmiş; özellikle IV tarafında Base64 alfabesi dışı değerler içeren diziler nedeniyle doğrulamalı şifrelemenin (AES-GCM) titizlikle incelenmesi gerekmiştir.

![Şifreleme Analizi](/assets/img/headset-reverse/part4-img01.png)

![Deobfuscation Sonucu](/assets/img/headset-reverse/part4-img02.png)

## Firmware İndirme Mantığı

Cihaz için indirilecek yazılım paketi rastgele veya serbest keşif yoluyla değil, uygulama içinde tanımlı bir mantık zinciri üzerinden belirlenir:
- Uygulama, BLE tarama sırasında elde edilen üretici verileri, cihaz adı ve model kodlarını kullanarak cihazın hangi ürün ailesine ait olduğunu sınıflandırır.
- Bu sınıflandırma sonucu, firmware sunucusundaki dizin yapısının hangi dalının kullanılacağını belirler.

Yetkilendirme için gereken kimlik bilgileri, uygulama içinde düz metin olarak saklanmamakta; bunun yerine obfuske edilmiş sabitlerden çalışma anında çözümlenen geçici kimlik materyali ile erişim sağlanmaktadır. İndirme tamamlandıktan sonra, dosyanın bütünlüğü MD5/SHA-256 hash kontrolleri ile doğrulanır.

---

## Final: Kendi İstemcimizi Geliştirme (PoC)

Elde edilen teknik bilgiler temel alınarak, **Flutter** kullanımıyla platformdan bağımsız bir istemci uygulama geliştirilmesi hedeflenmiştir. Bu uygulamanın amacı, cihazla kurulan iletişimi ve güncelleme süreçlerini kontrollü ve kullanıcı odaklı bir arayüz üzerinden yönetmektir.

![Flutter PoC Arayüzü](/assets/img/headset-reverse/part4-img03.png)

### Geliştirici Notları ve Kod Örnekleri

Aşağıda, analiz sonucu elde edilen ve kendi uygulamanızı geliştirirken kullanabileceğiniz kritik değerler ve akışlar paylaşılmıştır.

**1. Servis Tanımları**

```dart
// VENDOR SERVİSİ
static const String vendorService = "0000aa00-0000-1000-8000-00805f9b34fb";

// ANC KONTROL KARAKTERİSTİĞİ (READ + WRITE + NOTIFY)
static const String ancControl = "0000aa20-0000-1000-8000-00805f9b34fb";
/* ANC BYTE DEĞERLERİ:
   - 0x00 → Kapalı (Off/Neutral)
   - 0x01 → ANC Aktif (Noise Cancellation)
   - 0x02 → Şeffaflık/Farkındalık (Transparency/Awareness)
*/
```

![Kod Örnekleri 1](/assets/img/headset-reverse/part4-img04.png)

**2. Bağlantı Akışı (Connection Flow)**

Android tarafında stabil bir BLE bağlantısı için aşağıdaki adımlara dikkat edilmelidir. Özellikle MTU (Maximum Transmission Unit) isteği kritik önem taşır.

```dart
Future<bool> connect(BluetoothDevice device) async {
  try {
    // 1. Bağlan
    await device.connect(autoConnect: false);
    
    // 2. Android Bonding
    if (Platform.isAndroid) await ensureBonded(device);
    
    // 3. MTU Request - KRİTİK: 247 byte
    if (Platform.isAndroid) await device.requestMtu(247);
    
    // 4. Stabilizasyon
    await Future.delayed(const Duration(milliseconds: 300));
    
    // 5. Service Discovery
    await _discoverServices(device);
    
    return true;
  } catch (e) {
    return false;
  }
}
```

![Kod Örnekleri 2](/assets/img/headset-reverse/part4-img05.png)

**3. Batarya ve Sinyal İzleme**

Batarya servisi standart BLE profiline uygun olsa da, sağ/sol kulaklık ayrımı için birden fazla instance'ı kontrol etmek gerekebilir.

```dart
// Sol Kulaklık
if (_batteryLeftChar?.properties.notify == true) {
  await _batteryLeftChar!.setNotifyValue(true);
  _batteryLeftChar!.lastValueStream.listen((value) {
    onBatteryLeftChanged?.call(value[0]);
  });
}
```

![Kod Örnekleri 3](/assets/img/headset-reverse/part4-img06.png)

---

## Sonuç

Bu çalışma boyunca kapalı ve dokümantasyonsuz bir Bluetooth kulaklık ekosistemi çok katmanlı olarak çözümlendi ve ciddi teknik engeller sistematik biçimde aşıldı.

1.  **BLE Analizi:** `bluetoothctl`, `btmon` ve Wireshark kullanılarak vendor-specific servisler ve komut setleri çıkarıldı.
2.  **Mobil Tersine Mühendislik:** JADX ve Smali analizi ile uygulama içi mantık, gizli UUID'ler (`1337...beef`) ve firmware dağıtım URL'leri tespit edildi.
3.  **Kriptografik Çözümleme:** Firmware indirme yetkisi için kullanılan AES-GCM şifreli kimlik bilgileri deşifre edildi.
4.  **PoC Uygulama:** Elde edilen tüm bilgiler, çalışan bir Flutter uygulamasına dönüştürülerek "teori pratiğe" döküldü.

Bu noktada çalışma, belirsizlik içeren bir tersine inceleme olmaktan çıkmış; kullanılan araçlar, çözülen kriptografik yapılar ve elde edilen çıktılarla tamamlanmış, teknik olarak sağlam bir başarıya dönüşmüştür.

Okuduğunuz için teşekkürler.

**Özhan Yıldırım**

![Yazar / Sonuç](/assets/img/headset-reverse/part4-img07.png)
