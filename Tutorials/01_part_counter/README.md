
Readme · MD
# 01 · Part Counter — Çok İstasyonlu Konveyör Hücresi
 
Factory IO ile simüle edilen, S7-1500 üzerinde SCL ile yazılmış iki istasyonlu bir konveyör hücresi. Tek bir yeniden kullanılabilir Function Block, **çoklu instance** ile iki bağımsız istasyona uygulanır. Her istasyon parça sayar, hedefe ulaşınca durur, ortak E-stop ile güvenli durma ve reset kilidi içerir. İstasyonlar arasında **tek-CPU içi koordinasyon** (biri dolmadan diğeri başlamaz) ve bir **hücre kontrol katmanı** vardır. Mantık tamamen **CASE tabanlı state machine** ile kurulmuş, çıkışlar durumdan türetilmiştir.
 
> Bu proje, PLC kodunu bir yazılım projesi gibi ele alma pratiğinin örneğidir: parametrize FB, UDT ile veri modelleme, çoklu instance, katmanlı mimari (OB / hücre / istasyon), sembolik adresleme ve versiyon kontrolü.
 
---
 
## Ne Yapıyor
 
Her istasyon aynı state machine'i çalıştırır:
 
1. **IDLE** — Start bekler. (İstasyonun `enable` izni de gereklidir.)
2. **RUNNING** — Start'a basınca konveyör self-holding ile çalışır, parça sensörü sayar.
3. **SETTLE** — Hedef sayıya ulaşınca son parçanın yerleşmesi için 1 sn bekler; toplam üretime eklenir.
4. **FULL** — Konveyör durur, dolu göstergesi yanar, reset bekler.
5. **ESTOP** — E-stop'a basılınca, hangi durumda olursa olsun her şey durur, kırmızı yanar. Çıkış ancak E-stop bırakılıp reset'e basılınca mümkündür.
**Koordinasyon:** İstasyon 2, ancak istasyon 1 dolduktan (`FULL`) sonra başlatılabilir. İstasyon 1 her zaman serbesttir; istasyon 2'nin `enable`'ı istasyon 1'in `full` durumuna bağlıdır. Bu, ağ üzerinden gerçek handshake'in (Profinet, ileride) tek-CPU içi kavramsal provasıdır.
 
**Ortak E-stop:** Tek bir fiziksel E-stop butonu her iki istasyona da bağlıdır — basıldığında iki istasyon birden durur.
 
---
 
## Ortam
 
| Bileşen | Sürüm / Model |
|---|---|
| TIA Portal | V15 |
| CPU | S7-1500, CPU 1511-1 PN |
| Simülasyon | S7-PLCSIM + NetToPLCsim (127.0.0.1) |
| Fiziksel dünya | Factory IO (özel sahne, iki istasyon + kontrol paneli) |
| Dil | SCL |
 
---
 
## Mimari
 
Üç katmanlı yapı. Her katman bir üsttekinden daha dar bir sorumluluğa sahiptir:
 
```
Main (OB1) — orkestrasyon
 |-- factoryIO_com()          # Factory IO haberlesme
 |-- CellControl()            # HUCRE katmani (istasyonlardan ONCE)
 |-- PartCounter [DB 1]  -- st -> station1
 +-- PartCounter [DB 2]  -- st -> station2
 
CellControl (FC) — hucre karalari
 |-- handshake: station2.enable := station1.full
 |-- CellData.anyRunning / totalCount / allReady / eStopOK
 
PartCounter (FB) — istasyon mantigi (tek tanim, cok ornek)
 |-- IO normalizasyonu (NC sinyaller Temp'te)
 |-- sayac + timer (Static/Retain) -> sonuc UDT'ye
 |-- edge yakalama (Static)
 +-- CASE state machine -> cikislar durumdan turetilir
```
 
Kritik prensip: **`PartCounter` FB'si yalnızca kendi istasyonunu bilir.** Birden fazla istasyonu ilgilendiren her karar (ortak E-stop, koordinasyon, hücre durumu) bir üst katmanda — `CellControl` — yaşar. `CellControl` istasyonlardan **önce** çağrılır ki `enable` ve hücre verileri güncel olsun.
 
### Veri Modeli — `station` (UDT)
 
```
station
 |-- cmd     (komut -- disaridan istasyona)
 |    +-- enable      : Bool     # baslama izni
 |-- status  (durum -- istasyondan disariya)
 |    |-- running     : Bool
 |    |-- full        : Bool
 |    |-- ready       : Bool
 |    |-- eStop       : Bool
 |    +-- step        : Int
 +-- data    (veri)
      |-- count       : Int      # anlik sayac (o turdaki parca)
      |-- totalCount  : Int      # kumulatif uretim (tum turlar)
      +-- target      : Int      # hedef (disaridan ayarlanabilir)
```
 
**`count` vs `totalCount`:** `count` o anki turdaki parça sayısıdır (reset ile sıfırlanır). `totalCount` ise her dolum tamamlandığında (`SETTLE`) biriken kümülatif üretimdir — istasyonun ömür boyu toplam çıktısı.
 
Fiziksel IO ve geçici ara sinyaller UDT'ye **konmaz** — sayaç/timer/edge mekanizması FB'nin içinde, geçici sinyaller Temp'te yaşar. UDT yalnızca dışa açık, anlamlı veri taşır.
 
### Hücre Durumu — `CellData`
 
```
CellData
 |-- eStopOK     : Bool     # iki istasyon da E-stop'ta degil
 |-- allReady    : Bool     # her iki istasyon da IDLE/hazir
 |-- anyRunning  : Bool     # herhangi biri calisiyor
 +-- totalCount  : Int      # iki istasyonun toplam uretimi
```
 
Bu özet, ileride HMI (B4) "hücre durumu" ekranının veri kaynağıdır.
 
### State Machine
 
```
   [IDLE] --start+enable--> [RUNNING] --hedef--> [SETTLE]
     ^                                              |
     |                                          +1sn|
     |               reset                      [FULL]
     +------------------------------------------- <-+
 
   E-STOP: her durumdan --> [ESTOP] --(E-stop birak + reset)--> IDLE
```
 
| Step | Durum | Aksiyon |
|---|---|---|
| 0 | IDLE | Start + enable bekle |
| 20 | RUNNING | Konveyör çalışır (self-holding), parça sayılır |
| 30 | SETTLE | 1 sn bekleme, toplam üretime ekle |
| 40 | FULL | Dur, dolu göstergesi, reset bekle |
| 990 | ESTOP | Güvenli durum; çıkış için E-stop bırakılmalı + reset |
 
---
 
## IO Listesi (istasyon başına)
 
| Sinyal | Tip | Kontak | Açıklama |
|---|---|---|---|
| `start` | DI | NO | Başlat butonu |
| `stop` | DI | NC | Durdur butonu |
| `reset` | DI | NO | Reset / onay butonu |
| `eStopOk` | DI | NC | Acil durdurma (fail-safe, ortak) |
| `sensor` | DI | NC | Parça sensörü |
| `convOut` | DQ | — | Konveyör motoru |
| `greenLamp` | DQ | — | Çalışıyor göstergesi |
| `yellowLamp` | DQ | — | Bekleme göstergesi |
| `redLamp` | DQ | — | E-stop göstergesi |
| `levelLamp` | DQ | — | Dolu göstergesi |
 
İki istasyon `_1` / `_2` sonekli ayrı IO setleri kullanır; E-stop butonu ortaktır.
 
> **Güvenlik notu:** E-stop, stop ve parça sensörü **NC** (normally closed) bağlıdır. Kablo kopması veya besleme kaybı sinyali düşürür ve sistemi güvenli tarafa iter (fail-safe). Bu bilinçli bir tasarım tercihidir.
 
---
 
## Tasarım Kararları
 
- **Üç katmanlı mimari.** OB1 orkestrasyon, `CellControl` hücre kararları, `PartCounter` istasyon mantığı. Her katmanın sorumluluğu nettir.
- **Tek FB, çoklu Instance DB.** İkinci istasyon için kod kopyalanmaz; aynı FB ikinci bir Instance DB ile çağrılır. Bir class'tan iki obje türetmek gibi.
- **UDT ile veri modelleme.** İstasyon verisi `station` UDT'sinde toplanır; fiziksel IO ve geçici sinyaller dışarıda tutulur.
- **Koordinasyon FB'nin dışında.** İstasyonlar arası bağ `CellControl`'de kurulur; FB kendi istasyonundan başkasını bilmez — bu yeniden kullanılabilirliği korur.
- **Ortak E-stop.** Tek fiziksel buton iki istasyona bağlı; basıldığında hücre bütünüyle durur.
- **SCL, ladder değil.** Okunabilir, diff'lenebilir, versiyonlanabilir.
- **IO normalizasyonu tek yerde.** NC sinyallerin `NOT` çevrimi bloğun başında; mantık kodunda hiç `NOT` yoktur.
- **Self-holding çalışma.** Konveyör start'a bir kez basınca mühürlenir; stop veya E-stop bunu ezer.
- **E-stop CASE dışında, koşulsuz.** Her çevrim en başta kontrol edilir.
- **E-stop'tan çıkış otomatik değil.** İnsan onayı (reset) zorunludur.
- **Çıkışlar durumdan türetilir.** `convOut := running`, lambalar `step` değerinden.
---
 
## Bilinen Davranış
 
İstasyon 2'nin başlatılabilmesi için, start'a **istasyon 1 dolduktan sonra** basılması gerekir. İstasyon 1 dolmadan önce yapılan start basışı (edge olduğu için) kaybolur. Bu bilinçli bir tasarım: operatör, istasyon 1 dolduğunda istasyon 2'yi elle başlatır.
 
---
 
## Nasıl Çalıştırılır
 
1. `tia_project/PartCounter.zap15` dosyasını TIA Portal V15'te **Project -> Retrieve** ile aç.
2. Projeyi derle, S7-PLCSIM'e yükle, CPU'yu RUN'a al.
3. NetToPLCsim'i başlat, PLC'yi ekle (127.0.0.1), server'ı başlat.
4. `factoryio/part_counter.factoryio` sahnesini Factory IO'da aç, S7-PLCSIM driver'ıyla bağla.
5. İstasyon 1'de Start'a bas — konveyör çalışır, parça sayar. Dolunca istasyon 2 başlatılabilir. Tek E-stop iki istasyonu birden durdurur.
---
 
## Dosya Yapısı
 
```
01_part_counter/
 |-- README.md
 |-- src/
 |    |-- Main.scl               # OB1 - orkestrasyon
 |    |-- CellControl.scl        # hucre katmani
 |    +-- PartCounter.scl        # istasyon FB'si
 |-- tia_project/
 |    +-- PartCounter.zap15      # TIA Portal arsivi (Retrieve ile ac)
 |-- factoryio/
 |    +-- part_counter.factoryio # Factory IO sahnesi
 +-- media/
      +-- image.PNG        # cell görüntüsü
```
 
---
 
*Atılay Can Peker — [github.com/CanPeker](https://github.com/CanPeker)*