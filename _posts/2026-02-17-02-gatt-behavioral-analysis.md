---
layout: post
title: "Bir Kulaklığın Anatomisi - Bölüm 2: GATT Analizleri ve Davranışsal Mapping"
categories: [reverse-engineering]
date: 2026-02-17 12:00:00 +0300
tags: [bluetooth, ble, gatt, protocol-analysis]
toc: true
image:
  path: /assets/img/headset-reverse/part1-img01.png
  alt: Headset Reverse Engineering Cover
---

# GATT Analizleri ve Davranışsal Mapping

Bu bölümde, cihazın GATT (Generic Attribute Profile) servislerini ve karakteristiklerini inceleyerek, hangi servislerin hangi işlevleri yerine getirdiğini ortaya çıkaracağız.

## GATT Envanteri: Servis ve Karakteristiklerin Çıkarılması

Bluetooth LE bağlantısı kurulduktan sonra cihazın GATT veritabanını envanterlemek için `bluetoothctl` üzerinde GATT menüsüne geçilir ve tüm attribute’lar listelenir:

```bash
menu gatt
list-attributes
```

### 1) Battery Service (Standart BLE Servisi)
**UUID:** Battery Service (0000180f-0000-1000-8000-00805f9b34fb)
**Battery Percentage:** 0x5a (90)

Bu servis, cihazın Bluetooth SIG tarafından tanımlanmış standart Battery Service’i kullandığını göstermektedir. Batarya yüzdesinin doğrudan okunabilir olması, ilgili telemetri bilgisinin şifrelenmemiş ve pasif biçimde erişilebilir olduğunu ortaya koymaktadır.

**Not:** Çıktıda Battery Service’in iki farklı instance olarak göründüğü dikkat çekmektedir:
- `service001a/char001b` → Battery Level (0x2A19)
- `service0015/char0016` → Battery Level (0x2A19)

Bu durum genellikle cihazın birden fazla pil kaynağını (ör. sol/sağ kulaklık veya kulaklık + şarj kutusu) ayrı ayrı raporladığına işaret eder.

### 2) Vendor-Specific Servisler (Asıl Analiz Yüzeyi)

Listelenen servisler arasında standart dışı olanlar şunlardır:
- **UUID: Unknown** (0000aa00-0000-1000-8000-00805f9b34fb)
- **UUID: Google** (0000fe2c-0000-1000-8000-00805f9b34fb)
- **UUID: Vendor specific** (5052494d-2dab-0341-6972-6f6861424c45)

Bu servisler, cihazın standart BLE profillerinin ötesinde, üreticiye özgü kontrol ve telemetri mekanizmaları barındırdığını göstermektedir.

#### 0000aa00 (Unknown)
Bluetooth ekosisteminde sıkça, üreticiye özgü kontrol, durum ve telemetri verilerinin toplandığı merkezi bir servis olarak kullanılmaktadır. ANC durumu, mod geçişleri ve sensör çıktıları gibi veriler çoğu zaman bu servis altında taşınır.

#### 0000fe2c (Google)
Google tarafından tanımlanmış bu UUID, cihazın Fast Pair ve Android ekosistemi ile entegrasyon yeteneklerine sahip olabileceğine işaret etmektedir. Bu servis, eşleştirme kolaylığı ve kullanıcı deneyimi odaklı yardımcı veriler taşımak amacıyla kullanılmaktadır.

#### 5052494d-…
Standart BLE UUID bloklarının dışında kalan bu servis, açık biçimde vendor-proprietary bir yapıyı temsil etmektedir. UUID’nin ASCII karşılığı incelendiğinde üreticiye özgü bir imza barındırdığı görülmekte olup, cihazın özgün kontrol protokolünün bu servis üzerinden çalıştığı güçlü biçimde değerlendirilmektedir.

### 3) Manufacturer Specific Data
**ManufacturerData.Key:** 0x065a
**ManufacturerData.Value:** 06 00 9c 0d ac 09 a1 29 01

Manufacturer Specific Data alanı, üreticiye özgü verilerin BLE advertising ve bağlantı aşamalarında taşınmasına imkân tanır. Bu veri bloğu, cihazın BLE kimliği ile Classic Bluetooth tarafındaki adresleme ve durum bilgilerinin ilişkilendirilebildiğini göstermektedir.

---

## GATT Haritası ve Analiz Yüzeyi

Bu harita, cihazın standart BLE servislerinin yanında üç ayrı vendor spesific servis sunduğunu gösterir: `0000aa00`, `0000fe2c` ve `5052494d-…`. Devam eden adımlarda analiz odağı bu servisler altında yer alan karakteristiklerin davranışsal sınıflandırılmasıdır (event/state/command).

- **0000aa00 (Unknown)** → Üreticiye özgü kontrol/telemetri yüzeyi adayı.
- **0000fe2c (Google)** → Fast Pair/Android entegrasyonu.
- **5052494d-…** → Vendor-proprietary protokol yüzeyi.

Bir sonraki adımda, bu servislerin altındaki karakteristikler notify/read/write davranışlarına göre sınıflandırılacak ve hangi endpoint’in “event”, hangisinin “state”, hangisinin “command” rolü oynadığı deneysel olarak çıkarılacaktır.

---

## Batarya Telemetrisi — Notify Mekanizmasının Doğrulanması

Bu aşamada, cihazın batarya telemetrisini BLE üzerinden olay temelli (event-driven) olarak iletip iletmediğini test ediyoruz. Bunun için standart Battery Service altında yer alan Battery Level karakteristiği dinlenmektedir.

```bash
select-attribute /org/bluez/hci0/dev_D0_65_xx_xx_xx_xx/service0015/char0016
[DEVICE-LE]# notify on
[CHG] Attribute ... Notifying: yes
Notify started
```

Notify aktif hale geldikten sonra karakteristikten aşağıdaki değerler alınmıştır:
```
[CHG] Attribute ... Value: 64
```

**Gelen Verinin Yorumlanması:**
Hex `0x64` → Decimal `100`. Yani %100 şarj.

**Neden Bu Önemli?**
Batarya servisi, en güvenli ve standart test yüzeyi olduğu için seçilmiştir. Burada notify mekanizmasının çalıştığının doğrulanması, bir sonraki adımda ANC durumu, mod değişimleri ve sensör olayları gibi vendor-specific telemetri kanallarının da benzer şekilde dinlenebileceğine işaret eder.

---

## Vendor-Specific Telemetri — ANC Olay Akışının Gözlemlenmesi

Bu aşamada, daha önce envanteri çıkarılan vendor-specific servis altındaki bir karakteristik dinlenmiştir:

```bash
select-attribute /org/bluez/hci0/dev_D0_65_AE_XX_XX_XX/service001f/char0037
notify on
```

Notify mekanizması etkinleştirildikten sonra, kulaklığın Active Noise Cancelling (ANC) modu değiştirilirken aşağıdaki değerler gözlemlenmiştir:

```
[CHG] Attribute ... Value: 02
[CHG] Attribute ... Value: 00
[CHG] Attribute ... Value: 01
```

**İlgili karakteristik:** `char0037`, **UUID:** `0000aa20` (Vendor-specific).

Bu gözlem, söz konusu karakteristiğin olay temelli (event-driven) bir telemetri hattı olarak kullanıldığını göstermektedir. ANC ile etkileşim sırasında, cihazın BLE üzerinden tek baytlık durum veya olay kodları yayınladığı açıkça görülmektedir.

**Gözlemlenen Veri Modeli:**
Her değer, ANC’nin mevcut durumunu temsil eder:
- `00` → Kapalı (Off)
- `01` → Açık (On)
- `02` → Alternatif mod (ör. ANC / Ambient / Transparency)

Bu yaklaşımda karakteristik, cihazın anlık durumunu periyodik veya olay temelli olarak bildirir.

> [!IMPORTANT]
> **Not:** Bu aşamada elimizde, ilk bir davranış eşlemesi (initial mapping) yapmak için yeterli veri bulunmaktadır. Ancak bu eşlemenin kesinleştirilmesi için, kontrollü ve tekrarlanabilir bir test senaryosu gereklidir.

---

## Trafik Analizi

Classic Bluetooth (BR/EDR) trafiği, Wireshark ve `btmon` araçları kullanılarak hem bağlantı kurulumu hem de aktif kullanım senaryoları sırasında gözlemlenmiştir.

Elde edilen veriler incelendiğinde, ses iletimi ve profil yönetimi (A2DP/HFP/AVRCP) dışında, ANC kontrolü veya cihaz telemetrisiyle ilişkili herhangi bir uygulama-seviyesi veri taşınmadığı doğrulanmıştır. Bu durum, kontrol ve durum bilgisinin Classic kanal yerine Bluetooth Low Energy üzerinden yürütüldüğünü teyit etmektedir.

Analiz sürecinde ayrıca, mobil istemci tarafında Bluetooth logları alınmış; Android ekosistemine özgü Fast Pair mekanizması ve ilişkili servisler incelenmiştir. Ancak bu incelemeler sonucunda, çalışmanın odağını değiştirecek ölçekte kritik veya exploit edilebilir bir bulguya rastlanmamıştır.

Bu nedenle çalışma, Classic Bluetooth trafiği veya Fast Pair mekanizmasının derinlemesine araştırılması yerine, cihaz kontrol mantığının fiilen konumlandığı BLE katmanına ve kontrol uygulamasının tersine mühendisliğine odaklanacak şekilde ilerletilmiştir.
