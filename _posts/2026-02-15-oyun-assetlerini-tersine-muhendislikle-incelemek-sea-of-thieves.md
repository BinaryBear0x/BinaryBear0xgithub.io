---
layout: post
title: Oyun Assetlerini Tersine Mühendislikle İncelemek (Sea of Thieves)
date: 2026-02-15 21:00:00 +0300
categories: [Tersine Mühendislik, Oyun Geliştirme]
tags: [reverse-engineering, sea-of-thieves, unreal-engine, 3d-printing]
image:
  path: /assets/img/posts/2026-02-17-sea-of-thieves/cover.jpg
  alt: Sea of Thieves Cover Art
---

> **Merak bilginin yakıtıdır.**

## Bilgilendirme

Sea of Thieves İngiltere menşeili Rare Studio tarafından geliştirilen, dağıtımı Microsoft tarafından sağlanan Game as a service yapısına dayanan Korsan temalı FPS aksiyon macera türünden bir çok oyunculu oyundur.

Geliştirilmesinde oyun motoru olarak Unreal Engine 4 tercih edilmiştir.

### Merak aşaması

![image.png](/assets/img/posts/2026-02-17-sea-of-thieves/image.png)

Sea of Thieves yıllardır oynadığım ve arkasında dönen sistemleri, mekanikleri ve assetleri merak ettiğim bir oyun olmakla beraber Unreal Engine 4 ile olan kısmi oyun geliştirme tecrübem sayesinde daha da ilgimi çekmiştir.

Bu ilginin yanında bir 3D yazıcı sahibi olmam sebebiyle oyundaki çeşitli figürleri basmak istedim ancak internet üzerinde model paylaşılan sitelerin hiçbirinde istediğim modelleri bulamadım.

![image-1.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-1.png)

İstediğim model çeşitli başarımlar kazanmanın doğrultusunda gemilerin dekorasyonunda kullanılan **"Servant of the Flame Shipmate"** isimli modeldi.

İlgili modelin sadece 3D basılmış versiyonunu Etsy üzerinde kalitesiz şekilde bulabildim.

![image-2.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-2.png)

Tam olarak bu an merakın bilgiye dönüşmek üzere olduğu an oldu benim için.

### Bilgiyi elde etme aşaması

Oyunun kaynak klasörüne Steam üzerinden ulaştım ve incelemeye başladım.

![image-3.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-3.png)

Burada yapacağım değişiklikler ya da işlemler oyunun hile korumasını tetikleyebileceği için ilgili dosyaları tespit ettikten sonra sanal makine üzerinde güvenli bir yere taşımam gerekecekti.

Kısa bir araştırma sonrasında oyunun `Sea of Thieves\Athena\Content\Paks` dosya yolunda 4885 adet dosyanın olduğu klasöre ulaştım.

.pak dosyaları genellikle oyunlarda karşımıza çıkan bir sıkıştırılmış dosya tipidir yani en basit tanımla ziplenmiş veri diyebiliriz.

İlgili .pak dosyalarını incelemek için Umodel isimli Unreal Engine Resource Viewer programını kullandım.

![image-4.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-4.png)

Klasör isimleri daha anlamlı hale geldiğine göre artık istediğim modeli aramaya başlayabilirdim.

`Trinkets/Decorations/Standing_small` yolu istediğim modelin özelliklerini belirtiyordu.

Denemek için karşıma çıkan ilk modele tıkladım ve;

![image-5.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-5.png)

Oyunun assetlerinin çalınmaması için AES ile şifrelendiğini fark ettim.

Aes key dosya içerisinde bir yerlerde gizlenmiş olabilirdi ancak bunu bulmak için mimariyi kafamda düzgün kurmam gerekiyordu.

### İlk aşama: Aes keyi yakalamak ve Steam DRM korumasını kırmak

DRM kısaca dijital haklar yönetimi demektir ve kullanıcıların satın aldıkları dijital içeriklere erişim sağlaması için lisans oluşturmakla görevlidir. Oyunların çalışma performansını bu lisansı dinamik olarak oluşturduğu için kötü etkiler.

DRM Korumasını kaldırmak için [Steamless](https://github.com/atom0s/Steamless) adresinden indirdiğim toolu kullandım.

> [!WARNING]
> Bu işlem teknik olarak oyunun cracklenmesini sağlamaz çünkü Steamless isimli program Steamworks API entegrasyonunu kaldırmaz, araya bir emülatör koymaz ve hile koruma sistemlerini devre dışı bırakmaz.

![image-6.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-6.png)

Bu program bana DRM korumasından ayrılmış şekilde oyunun executable dosyasını veriyor.

DRM koruması kalktığı, yani ilk kalkan düştüğü için ikinci kalkana ilerledim.

[AESKeyFinder](https://github.com/GHFear/AESKeyFinder-By-GHFear) programı aracılığıyla DRM koruması kaldırılmış exeyi program klasörüne ekledim ve `RUN find 256-bit ue4 aes key.bat` isimli bat dosyasını çalıştırdım.

![image-7.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-7.png)

Program bana çıktı olarak `0x37A0BC3DC2E01D9EB4923CA266A5701F56A4802347F07927FC3FC25C93B31B50` şeklinde AES keyi verdi.

Şimdi tüm kalkanlar düştüğü için [QuickBMS](https://github.com/LittleBigBug/QuickBMS) toolunu kullanarak .pak dosyalarını dışarıya çıkarabilirdim.

QuickBMS toolunun çalışması için internetten bulabileceğiniz aşağıdaki bms scriptine sahip olmanız gerekmektedir. Bu script aracılığıyla ue temelli çoğu oyunda dışarıya çıkarma işlemi gerçekleştirebilirsiniz.

![image-8.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-8.png)

![image-9.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-9.png)

AES keyin doğru olduğuna emin olduğum için Bms scriptinde bir hata ya da ek güvenlik olduğunu düşünerek QuickBMS toolunu kullanmayı bıraktım ve UModel programına geri döndüm.

Bms scripti kullanmak yerine Override game detection kısmından ilgili oyunu, oyun motorun ve platformu seçip diğer ayarları default bıraktım.

![image-10.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-10.png)

![image-11.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-11.png)

![image-12.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-12.png)

İlgili dosyaları seçtim ve Export buttonuna tıkladım.

Gelen menüyü default olarak bırakıyorum.

Bu sefer AES keye sahip olduğumuz için keyi de girelim.

![image-13.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-13.png)

Dosyaları `D:\y\ship\Game\Models\Ships\Shp_mid_01_a` adresine export ettikten sonra hiç de şifresi kırılmış gibi gözükmeyen ilginç bir dosya türü ile karşılaştım.

![image-14.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-14.png)

![image-15.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-15.png)

Araştırmalarım sonucu öğrendim ki .Pskx dosyası Autodesk 3ds max programıyla ilişkili bir 3d model dosyasıymış.

Bu programa sahip olmadığım için - ki olsam bile düzgün çalışacak mı emin değilim- bilgisayarımda bulunan Blender isimli uygulama ile dosyayı açmaya çalıştım ancak .pskx import seçeneği yoktu.

![image-16.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-16.png)

Blender Python ile yapılmış olduğu için python dili ile yazılmış olan pluginleri destekliyor.

[Befzz/blender3d_import_psk_psa](https://github.com/Befzz/blender3d_import_psk_psa) adresinden ilgili scripti buldum ve 1,2 ufak değişiklik ile blender programına ekledim.

![image-17.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-17.png)

![image-18.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-18.png)

### Merakın bilgiye dönüştüğü kısım

4 saatlik uğraş, DRM korumasını devre dışı bırakma, AES key bulma ve dosyaları kırma sonucunda istediğime ulaştım.

![image-19.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-19.png)

![image-20.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-20.png)

3D Modeller artık elimdeydi ancak oyun için modelleme ve 3d yazıcı baskısı için modelleme arasında çok büyük farklar var. Bir oyunda bulunan modeli direkt olarak bastırmak mümkün değil.

Meshmixer programı ve creality print programı ile modeli solid hale getirip doku yırtıklarını tamir ettim.

Geriye bastırma aşaması kaldı.

Oyunda bulunan başka bir modelin baskısı ise aşağıdaki gibidir.

![image-21.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-21.png)

![image-22.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-22.png)

![image-23.png](/assets/img/posts/2026-02-17-sea-of-thieves/image-23.png)

### Final

Yukarıda anlattığım yöntemler ile Unreal Engine ile yapılmış çoğu oyunda 3d model export yapılabilir.

Okuduğunuz için teşekkürler.
