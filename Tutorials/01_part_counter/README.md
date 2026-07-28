# 01 · Part Counter — Çok İstasyonlu Konveyör Hücresi

Factory IO ile simüle edilen, S7-1500 üzerinde SCL ile yazılmış bir konveyör hücresi. Tek bir yeniden kullanılabilir Function Block, **çoklu instance** ile birden fazla bağımsız istasyona uygulanır. Her istasyon parça sayar, hedefe ulaşınca durur, E-stop ile güvenli durma ve reset kilidi içerir. İstasyonlar arasında **tek-CPU içi koordinasyon** vardır (biri dolmadan diğeri başlamaz). Mantık tamamen **CASE tabanlı state machine** ile kurulmuş, çıkışlar durumdan türetilmiştir.

> Bu proje, PLC kodunu bir yazılım projesi gibi ele alma pratiğinin örneğidir: parametrize FB, UDT ile veri modelleme, çoklu instance, sembolik adresleme ve versiyon kontrolü.

---

## Ne Yapıyor

Her istasyon aynı state machine'i çalıştırır:

1. **IDLE** — Start bekler. (İstasyonun `enable` izni de gereklidir.)
2. **RUNNING** — Start'a basınca konveyör self-holding ile çalışır, parça sensörü sayar.
3. **SETTLE** — Hedef sayıya ulaşınca son parçanın yerleşmesi için 1 sn bekler.
4. **FULL** — Konveyör durur, dolu göstergesi yanar, reset bekler.
5. **ESTOP** — E-stop'a basılınca, hangi durumda olursa olsun her şey durur, kırmızı yanar. Çıkış ancak E-stop bırakılıp reset'e basılınca mümkündür.

**Koordinasyon:** İstasyon 2, ancak istasyon 1 dolduktan (`FULL`) sonra başlatılabilir. İstasyon 1 her zaman serbesttir (`enable = TRUE`); istasyon 2'nin `enable`'ı istasyon 1'in durumuna bağlıdır. Bu, ağ üzerinden gerçek handshake'in (Profinet, ileride) tek-CPU içi kavramsal provasıdır.

---

## Ortam

| Bileşen | Sürüm / Model |
|---|---|
| TIA Portal | V15 |
| CPU | S7-1500, CPU 1511-1 PN |
| Simülasyon | S7-PLCSIM + NetToPLCsim (127.0.0.1) |
| Fiziksel dünya | Factory IO (özel sahne) |
| Dil | SCL |

---

## Mimari

Tek bir Function Block (`PartCounter`) tanımlanır ve **iki ayrı Instance DB** ile çağrılır — kopyalama yok, tek kod tabanı. Her istasyonun mantıksal verisi (durum, sayaç, komut, hata) bir **UDT** (`stStation`) içinde tutulur. Fiziksel IO parametre olarak, mantıksal durum UDT üzerinden (InOut) geçer. İstasyonlar arası koordinasyon FB'nin dışında, OB1 katmanında yapılır — FB kendi istasyonundan başkasını bilmez.

\`\`\`
OB1 (orkestrasyon + koordinasyon)
 ├── PartCounter  [Instance DB 1] ── st -> station1   (UDT)
 ├── PartCounter  [Instance DB 2] ── st -> station2   (UDT)
 └── koordinasyon: station2.enable := station1.full

PartCounter (FB) — tek tanim, cok ornek
 ├── IO normalizasyonu (NC sinyaller Temp'te tek yerde cevrilir)
 ├── sayac (Static) -> sonuc UDT'ye (st.data.count)
 ├── edge yakalama (Static: start / stop / reset)
 └── CASE state machine -> cikislar durumdan turetilir
\`\`\`

### Veri Modeli — \`stStation\` (UDT)

\`\`\`
stStation
├── cmd     (komut — disaridan istasyona)
│   └── enable        : Bool     # baslama izni
├── status  (durum — istasyondan disariya)
│   ├── running       : Bool
│   ├── full          : Bool
│   └── step          : Int
└── data    (veri)
    ├── count         : Int      # sayilan parca
    └── target        : Int      # hedef (disaridan ayarlanabilir)
\`\`\`

Fiziksel IO ve geçici ara sinyaller UDT'ye **konmaz** — sayaç/timer/edge mekanizması FB'nin Static'inde, geçici sinyaller Temp'te yaşar. UDT yalnızca dışa açık, anlamlı veri taşır.

### State Machine

\`\`\`
        [ IDLE ] --start+enable--> [ RUNNING ] --hedef--> [ SETTLE ]
           ^                                                  |
           |                                              +1sn|
           |                  reset                     [ FULL ]
           +---------------------------------------------- ...

        E-STOP: her durumdan --> [ ESTOP ] --(E-stop birak + reset)--> IDLE
\`\`\`

| Step | Durum | Aksiyon |
|---|---|---|
| 0 | IDLE | Start + enable bekle |
| 20 | RUNNING | Konveyör çalışır (self-holding), parça sayılır |
| 30 | SETTLE | 1 sn bekleme |
| 40 | FULL | Dur, dolu göstergesi, reset bekle |
| 990 | ESTOP | Güvenli durum; çıkış için E-stop bırakılmalı + reset |

---

## IO Listesi (istasyon başına)

| Sinyal | Tip | Kontak | Açıklama |
|---|---|---|---|
| \`start\` | DI | NO | Başlat butonu |
| \`stop\` | DI | NC | Durdur butonu |
| \`reset\` | DI | NO | Reset / onay butonu |
| \`eStopOk\` | DI | NC | Acil durdurma (fail-safe) |
| \`sensor\` | DI | NC | Parça sensörü |
| \`convOut\` | DQ | — | Konveyör motoru |
| \`greenLamp\` | DQ | — | Çalışıyor göstergesi |
| \`yellowLamp\` | DQ | — | Bekleme göstergesi |
| \`redLamp\` | DQ | — | E-stop göstergesi |
| \`levelLamp\` | DQ | — | Dolu göstergesi |

Her istasyon kendi IO setini kullanır (\`_1\` / \`_2\` sonekli tag'ler).

> **Güvenlik notu:** E-stop, stop ve parça sensörü **NC** (normally closed) bağlıdır. Kablo kopması veya besleme kaybı sinyali düşürür ve sistemi güvenli tarafa iter (fail-safe). Bu bilinçli bir tasarım tercihidir.

---

## Tasarım Kararları

- **Tek FB, çoklu Instance DB.** İkinci istasyon için kod kopyalanmaz; aynı FB ikinci bir Instance DB ile çağrılır. Bir class'tan iki obje türetmek gibi — ortak kod, ayrı state.
- **UDT ile veri modelleme.** İstasyon verisi \`stStation\` UDT'sinde toplanır; fiziksel IO ve geçici sinyaller dışarıda tutulur. UDT yalnızca dışa açık, anlamlı durum taşır.
- **Koordinasyon FB'nin dışında.** İstasyonlar arası bağ (biri dolmadan diğeri başlamaz) OB1 katmanında kurulur. FB kendi istasyonundan başkasını bilmez — bu yeniden kullanılabilirliği korur.
- **SCL, ladder değil.** Mantık yazılım mühendisliği disipliniyle yazıldı — okunabilir, diff'lenebilir, versiyonlanabilir.
- **IO normalizasyonu tek yerde.** NC sinyallerin \`NOT\` çevrimi bloğun başında yapılır; mantık kodunda hiç \`NOT\` yoktur.
- **Self-holding çalışma.** Konveyör, start'a bir kez basınca mühürlenir; stop veya E-stop bunu ezer.
- **E-stop CASE dışında, koşulsuz.** Her çevrim en başta kontrol edilir — hangi state'te olursa olsun yakalanır.
- **E-stop'tan çıkış otomatik değil.** İnsan onayı (reset) zorunludur.
- **Çıkışlar durumdan türetilir.** \`convOut := running\`, lambalar \`step\` değerinden.

---

## Bilinen Davranış

İstasyon 2'nin başlatılabilmesi için, start'a **istasyon 1 dolduktan sonra** basılması gerekir. İstasyon 1 dolmadan önce yapılan start basışı (edge olduğu için) kaybolur. Bu bilinçli bir tasarım: operatör, istasyon 1 dolduğunda istasyon 2'yi elle başlatır.

---

## Nasıl Çalıştırılır

1. \`tia_project/PartCounter.zap15\` dosyasını TIA Portal V15'te **Project → Retrieve** ile aç.
2. Projeyi derle, S7-PLCSIM'e yükle, CPU'yu RUN'a al.
3. NetToPLCsim'i başlat, PLC'yi ekle (127.0.0.1), server'ı başlat.
4. \`factoryio/part_counter.factoryio\` sahnesini Factory IO'da aç, S7-PLCSIM driver'ıyla bağla.
5. İstasyon 1'de Start'a bas — konveyör çalışır, parça sayar. Dolunca istasyon 2 başlatılabilir.

---

## Dosya Yapısı

\`\`\`
01_part_counter/
├── README.md
├── src/
│   └── PartCounter.scl          # okunabilir kaynak kod
├── tia_project/
│   └── PartCounter.zap15        # TIA Portal arsivi (Retrieve ile ac)
├── factoryio/
│   └── part_counter.factoryio   # Factory IO sahnesi
└── media/
    └── PartCounter.mp4          # demo videosu
\`\`\`

---

## Kaynak Kod

Tam SCL için: [\`src/PartCounter.scl\`](src/PartCounter.scl)

---

*Atılay Can Peker — [github.com/CanPeker](https://github.com/CanPeker)*
