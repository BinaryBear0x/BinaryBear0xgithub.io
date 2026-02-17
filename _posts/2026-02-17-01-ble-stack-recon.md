---
layout: post
title: "Bir Kulaklığın Anatomisi - Bölüm 1: BLE Stack Analizi"
categories: [reverse-engineering]
date: 2026-02-17 12:00:00 +0300
tags: [bluetooth, ble, firmware, reversing]
toc: true
image:
  path: /assets/img/headset-reverse/part1-img01.png
  alt: Headset Reverse Engineering Cover
---

# Bir Kulaklık Ekosisteminin Anatomisi: BLE Stack Analizi

**Cihaz–İstemci İletişim Katmanının Tersine Mühendisliği ve AWS Altyapısı Üzerinden Firmware Dağıtım Kanallarının Analizi**

## Giriş

Bu çalışma, modern bir Bluetooth kulaklık ekosisteminin görünmeyen tarafta nasıl iletişim kurduğunu ele alan bir tersine mühendislik incelemesidir. Yüzeyden bakıldığında yalnızca ses üreten bir cihaz gibi görünen kulaklıkların, Bluetooth Low Energy protokolü üzerinden nasıl kontrol edildiği; servis düzeyi iletişim, durum telemetrisi ve komut mekanizmaları üzerinden adım adım incelenmektedir.

İnceleme, cihaz–istemci iletişim katmanını merkeze alarak kulaklığın yalnızca bir donanım değil, çok katmanlı bir yazılım ve protokol sistemi olarak nasıl davrandığını ortaya koymayı amaçlamaktadır.

> [!NOTE]
> **Motivasyon:** Bu araştırmanın çıkış noktası oldukça basit bir merakla başladı. Günlük olarak kullandığım Bluetooth kulaklığın kontrol uygulamasının üretici tarafından bir süre sonra uygulama marketlerinden kaldırıldığını fark ettim. Uygulama ortadan kalkınca, kulaklık hâlâ ses veriyor, bağlanıyor ve temel işlevlerini yerine getiriyordu. Ancak kulaklığın sunduğu bazı ayarların ve davranışların kontrolünü kaybetmiştim.

Bu durum, kulaklığımın uygulama ile nasıl iletişim kurduğunu merak etmeme yol açtı. Basit bir kullanım sorusuyla başlayan bu merak, zamanla kulaklığın Bluetooth Low Energy üzerinden nasıl haberleştiğini, hangi verileri ürettiğini ve hangi mekanizmalarla komut aldığını anlamaya yönelik daha derin bir araştırmaya dönüştü.

Ortaya çıkan çalışma; planlanmış bir hedeften ziyade, bir cihazın “nasıl çalıştığını gerçekten görmek” isteğinin adım adım teknik bir incelemeye evrilmesinin sonucudur.

### Sorumluluk Reddi ve Kapsam

Bu çalışma kapsamında gerçekleştirilen tüm analizler, yazarın kendi kullanımında bulunan donanım üzerinde ve gözlemsel/analitik yöntemlerle yürütülmüştür. Araştırma sürecinde herhangi bir sisteme zarar verilmemiş, hizmet kesintisine yol açılmamış veya üçüncü taraflara ait altyapılara müdahalede bulunulmamıştır.

Metin içerisinde yer alan ürün adları, cihaz kimlikleri, servis uç noktaları, adresler ve benzeri teknik bilgiler; çalışmanın anlaşılabilirliğini korumak amacıyla kısmen değiştirilmiş, maskelenmiş veya anonimleştirilmiştir. Bu bilgiler, gerçek sistemlerin birebir temsili olarak değerlendirilmemelidir.

Bu makalenin amacı; bir Bluetooth kulaklık ekosisteminin teknik işleyişini, tersine mühendislik perspektifinden incelemek ve iletişim katmanlarının nasıl çalıştığını ortaya koymaktır. İçerik, güvenlik açığı istismarı, yetkisiz erişim veya zararlı kullanım senaryolarını teşvik etmeyi amaçlamamaktadır.

## Bluetooth Low Energy (BLE) Protokol Yığını ve Temel Kavramlar

Bluetooth, kısa menzilli kablosuz haberleşme için tasarlanmış, 2.4 GHz ISM bandı üzerinde çalışan, düşük güç tüketimli bir dijital iletişim protokol ailesidir. Yüzeyde kullanıcıya “cihaz eşleştirme” ve “veri aktarımı” gibi basit işlevler sunsa da, teknik açıdan Bluetooth; frekans atlamalı (FHSS), zaman dilimli, paket tabanlı ve olay temelli bir iletişim yapısına sahip karmaşık bir protokol yığınıdır.

Bluetooth; hava arayüzü üzerinden çalışan, çekirdeği standartlara dayanan ancak üreticiye özgü genişletmelerle şekillenen bir haberleşme sistemidir. İletişimin temel zamanlama ve kanal yönetimi Link Layer’da gerçekleştirilirken, üst katmanlarda servis keşfi, güvenli bağlantı kurulumu ve uygulama verisi taşınır.

Bluetooth ekosisteminde, özellikle BLE tarafında, aktif yayın ve bağlantı temelli iki ana iletişim modeli öne çıkar. Advertising paketleri, cihaz kimliği, servis ipuçları ve üreticiye özel veriler içererek pasif dinleme (passive sniffing) için yüksek değerli bir istihbarat kaynağı oluşturur. Bağlantı kurulduğunda ise iletişim, kanal atlamaları ve zamanlama nedeniyle doğrudan dinlemeye karşı daha dirençlidir; ancak güvenlik açıkları veya vendor hataları hedef alınarak analiz edilebilir.

Tersine mühendislik perspektifinde Bluetooth, standart dışı GATT karakteristikleri, özel komut setleri ve şifrelenmiş telemetri üzerinden cihaz davranışlarının yeniden inşa edilmesine imkân tanır. Özellikle kulaklıklar, IoT cihazları ve giyilebilir teknolojiler; resmi uygulamalar olmaksızın, yalnızca Bluetooth trafiği analiz edilerek kontrol edilebilir veya modellenebilir.

> [!TIP]
> Bu yaklaşım, otomotiv dünyasında yaygın olarak bilinen; araçlarda donanımsal olarak mevcut olan ancak yazılım veya konfigürasyon seviyesinde devre dışı bırakılmış özelliklerin, üretici dışı araçlar kullanılarak etkinleştirilmesine benzetilebilir. Bu tür çalışmalar, yeni bir işlev eklemekten ziyade, mevcut sistemin nasıl çalıştığını anlamaya ve var olan davranışları görünür hâle getirmeye odaklanır.

![Bluetooth Low Energy protokol yığınının katmanlı yapısı](/assets/img/headset-reverse/part1-img02.png)
_Şekil 1. Bluetooth Low Energy protokol yığınının katmanlı yapısı._

Diyagram, radyo seviyesinden GATT profiline kadar uzanan iletişim katmanlarını ve üreticiye özgü servislerin genellikle üst katmanlarda konumlandığını göstermektedir. Tersine mühendislik açısından en fazla etkileşim yüzeyi ATT ve GATT katmanlarında ortaya çıkmaktadır.

Yukarıda tanımlanan BLE protokol katmanları, sahada belirli iletişim ve veri modelleri üzerinden kendini gösterir. Advertising, bağlantı temelli iletişim, attribute tabanlı veri modeli ve üreticiye özgü genişletmeler; bu katmanlı yapının pratikteki yansımalarıdır.

1. **Advertising Mekanizması**
   - Cihazlar periyodik olarak advertising paketleri yayınlar.
   - Bu paketler şunları içerebilir: Cihaz adresi, Servis UUID’leri, Üreticiye özel (Manufacturer Specific Data).

2. **Connection-Oriented İletişim**
   - Bağlantı kurulduktan sonra: Kanal atlamaları senkronize edilir, Connection interval ve latency uygulanır.
   - Şifreleme bu aşamada devreye girer (LE Legacy / LE Secure Connections).
   - Zayıf eşleştirme senaryoları (Just Works) analiz için zemin oluşturur.

3. **Veri Modeli (Attribute-Centric Design)**
   - BLE’de “paket” değil anlamlı veri nesneleri ön plandadır.
   - Her cihaz, kendini bir veri ağacı (GATT database) olarak sunar.
   - Bu durum: Bilinmeyen cihazların davranışsal olarak yeniden inşa edilmesini ve Vendor uygulaması olmadan kontrol edilmesini mümkün kılar.

4. **Vendor-Specific Genişletmeler**
   - Standart UUID’ler dışında özel UUID blokları kullanılır.
   - ANC, EQ, gesture, firmware update gibi işlevler çoğu zaman belgelenmez.

Reverse engineering süreci genellikle şu adımlardan oluşur:
1. GATT keşfi
2. Karakteristik erişim denemeleri
3. Notify/Write davranışlarının korelasyonu
4. Uygulama ↔ cihaz trafiğinin karşılaştırılması

## Analiz

Bu örnek senaryoda, kontrol uygulaması üretici tarafından platformlardan kaldırılmış olan ticari bir TWS Bluetooth kulaklık üzerinde BLE tabanlı iletişim incelenmektedir. Çalışma kapsamında, cihazın BLE üzerinden nasıl kontrol edildiğini anlamak amacıyla finalde sınırlı işlevlere sahip deneysel bir istemci uygulaması geliştirilmiştir.

Kullanılan yöntemler güvenlik perspektifinden değerlendirildiğinde; bireysel kullanıcı açısından uzun süreli, kritik veya fiziksel bir zarara yol açma potansiyeline sahip değildir. Ancak pasif ve aktif gözlemler üzerinden, kullanım alışkanlıklarına dair çıkarımlar yapılabilmesi ve cihaz davranışlarının zaman içinde izlenebilmesi mümkündür.

### Analiz Ortamının Seçilmesi

Bu noktada analiz ortamı üzerinde kısaca durmak gerekir. İlk bakışta sanal bir analiz ortamı (ör. REMnux sanal makinesi) kullanmak cazip görünse de, Bluetooth tabanlı düşük seviye analizlerde bu yaklaşım pratikte sınırlıdır. VMware veya VirtualBox gibi sanallaştırma çözümlerinde çalışan bir işletim sistemi, Bluetooth donanımına doğrudan erişemez; iletişim ana işletim sistemi üzerinden soyutlanmış şekilde gerçekleşir.

Bu durum, HCI seviyesinde trafik gözlemi ve BLE davranış analizi gibi çalışmalarda sanal makine kullanımını işlevsiz hâle getirir. Bu nedenle analiz ortamı için iki temel seçenek bulunmaktadır: harici bir Bluetooth USB dongle kullanmak veya donanıma doğrudan erişim sağlayan bir canlı sistem (live USB) üzerinde çalışmak.

Bu çalışmada, analiz sürecine hızlı ve temiz bir başlangıç yapabilmek amacıyla canlı sistem yaklaşımı tercih edilmiştir. Güncel ve kararlı bir Ubuntu sürümü kullanılarak, Bluetooth yığınına doğrudan erişim sağlanan bir analiz ortamı oluşturulmuştur.

Analiz ortamı hazırlandıktan sonra, Bluetooth denetleyicisinin durumunu doğrulamak için komut satırı incelemesine geçilmiştir.

```bash
ubuntu@ubuntu:~/Desktop$ bluetoothctl show
Controller C0:A5:E8:XX:XX:XX (public)
    Manufacturer: 0x0002 (2)
    Version: 0x0b (11)
    Name: ubuntu
    Alias: ubuntu
    Class: 0x006c010c (7078156)
    Powered: yes
    Discoverable: no
    DiscoverableTimeout: 0x000000b4 (180)
    Pairable: no
    UUID: A/V Remote Control (0000110e-0000-1000-8000-00805f9b34fb)
    # ... (Diğer UUID'ler)
    Discovering: no
    Roles: central
    Roles: peripheral
```

Şimdi hedef kulaklığı eşleştirme moduna (pairing mode) alıyoruz. Bu adımın uygulaması cihazdan cihaza farklılık gösterebilir; amaç cihazın eşleştirilebilir ve keşfedilebilir durumda olmasıdır.

Ardından Bluetooth yönetim oturumunu başlatmak için `bluetoothctl` çalıştırılır:

```bash
[bluetooth]# scan on
SetDiscoveryFilter success
Discovery started
[CHG] Controller C0:A5:E8:XX:XX:XX Discovering: yes
[NEW] Device D0:65:AE:XX:XX:XX TWS-DEVICE-01 [LE]
[NEW] Device 7D:C0:CE:XX:XX:XX 7D-C0-CE-XX-XX-XX
[NEW] Device 9C:0D:AC:XX:XX:XX TWS-DEVICE-01
```

> [!NOTE]
> **Not:** Tarama çıktısında aynı fiziksel cihaza ait birden fazla girişin gözlemlenmesi, cihazın hem Bluetooth Classic hem de Bluetooth Low Energy bağlamında farklı advertising davranışları sergilediğini göstermektedir. Bu durum, özellikle TWS sınıfı kulaklıklarda yaygın olan çift modlu (dual-mode) mimarinin doğal bir sonucudur.

Bu aşamada hedef cihazın Bluetooth Low Energy reklam paketleri pasif olarak gözlemlenmiş ve cihazın BLE üzerinden aktif olarak advertise ettiği doğrulanmıştır. Şimdi hedef cihaza bağlanalım:

```bash
connect D0:65:AE:XX:XX:XX
```

Bağlantı sırasında bir eşleştirme isteği (pairing request) gelirse `yes` ile onaylayarak devam ediyoruz. Ardından cihazı güvenilir olarak işaretliyoruz:

```bash
trust D0:65:AE:XX:XX:XX
```

Cihaz durumunu ve sunulan servisleri görüntülemek için:

```bash
info D0:65:AE:XX:XX:XX
Device D0:65:AE:XX:XX:XX (random)
    Name: [TWS-DEVICE] [LE]
    Alias: [TWS-DEVICE] [LE]
    Appearance: 0x03c0 (960)
    Paired: yes
    Bonded: yes
    Trusted: yes
    Connected: yes
    UUID: Generic Access Profile    (00001800-0000-1000-8000-00805f9b34fb)
    UUID: Generic Attribute Profile (00001801-0000-1000-8000-00805f9b34fb)
    UUID: Device Information        (0000180a-0000-1000-8000-00805f9b34fb)
    UUID: Battery Service           (0000180f-0000-1000-8000-00805f9b34fb)
    UUID: Unknown                   (0000aa00-0000-1000-8000-00805f9b34fb)
    UUID: Google                    (0000fe2c-0000-1000-8000-00805f9b34fb)
    UUID: Vendor specific           (XXXXXXXXXXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX)
```

Bu noktada `Connected: yes` ve BLE servis UUID’lerinin listelenmesi, hedef cihazla BLE bağlantısının kurulduğunu ve GATT üzerinden servis keşfi yapılabildiğini göstermektedir. Dolayısıyla cihaz artık GATT servislerini tam olarak açabilir.

Bir sonraki bölümde, GATT menüsüne geçerek servis ve karakteristik envanterini çıkaracağız.
