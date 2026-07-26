# 01 · Part Counter — Sensör Tetiklemeli Konveyör Hücresi

Factory IO ile simüle edilen, S7-1500 üzerinde SCL ile yazılmış bir konveyör hücresi. Parça sayar, hedefe ulaşınca durur, E-stop ile güvenli durma ve reset kilidi içerir. Mantık tamamen **CASE tabanlı state machine** ile kurulmuş, çıkışlar durumdan türetilmiştir.


## Ne Yapıyor

1. **IDLE** — Start bekler.
2. **RUNNING** — Start'a basınca konveyör self-holding ile çalışır, parça sensörü sayar.
3. **SETTLE** — Hedef sayıya (5) ulaşınca son parçanın yerleşmesi için 1 sn bekler.
4. **FULL** — Konveyör durur, dolu göstergesi yanar, reset bekler.
5. **ESTOP** — E-stop'a basılınca, hangi durumda olursa olsun her şey durur, kırmızı yanar. Çıkış ancak E-stop bırakılıp reset'e basılınca mümkündür.

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

Mantık tek bir Function Block (`PartCounter`) içinde toplanmıştır. OB1 yalnızca bu FB'yi çağırır ve fiziksel IO'yu parametrelere bağlar — mantık OB1'de değildir.

```
OB1 (orkestrasyon)
 └── PartCounter (FB) — state machine + sayaç + E-stop
      ├── IO normalizasyonu (NC sinyaller tek yerde çevrilir)
      ├── CTU sayaç
      ├── R_TRIG edge yakalama (start / reset)
      └── CASE state machine → çıkışlar durumdan türetilir
```

### State Machine

```
        ┌──────┐  start   ┌─────────┐  5 parça  ┌────────┐
        │ IDLE │────────▶ │ RUNNING │─────────▶ │ SETTLE │
        └──────┘          └─────────┘  +1sn     └────┬───┘
           ▲                                          │
           │            reset                    ┌────▼───┐
           └──────────────────────────────────── │  FULL  │
                                                  └────────┘

        E-STOP: her durumdan ──▶ [ ESTOP ] ──(E-stop bırak + reset)──▶ IDLE
```

| Step | Durum | Aksiyon |
|---|---|---|
| 0 | IDLE | Start bekle |
| 20 | RUNNING | Konveyör çalışır (self-holding), parça sayılır |
| 30 | SETTLE | 1 sn bekleme |
| 40 | FULL | Dur, dolu göstergesi, reset bekle |
| 990 | ESTOP | Güvenli durum; çıkış için E-stop bırakılmalı + reset |

---

## IO Listesi

| Sinyal | Tip | Kontak | Açıklama |
|---|---|---|---|
| `start` | DI | NO | Başlat butonu |
| `stop` | DI | NC | Durdur butonu |
| `reset` | DI | NO | Reset / onay butonu |
| `eStop` | DI | NC | Acil durdurma (fail-safe) |
| `sensor` | DI | NC | Parça sensörü |
| `convOut` | DQ | — | Konveyör motoru |
| `greenLamp` | DQ | — | Çalışıyor göstergesi |
| `yellowLamp` | DQ | — | Dolu / bekleme göstergesi |
| `redLamp` | DQ | — | E-stop göstergesi |

> **Güvenlik notu:** E-stop, stop ve parça sensörü **NC** (normally closed) bağlıdır. Kablo kopması veya besleme kaybı sinyali düşürür ve sistemi güvenli tarafa iter (fail-safe). Bu bilinçli bir tasarım tercihidir.

---

## Tasarım Kararları

- **SCL, ladder değil.** Mantık yazılım mühendisliği disipliniyle yazıldı — okunabilir, diff'lenebilir, versiyonlanabilir.
- **IO normalizasyonu tek yerde.** NC sinyallerin `NOT` çevrimi bloğun başında yapılır; mantık kodunda hiç `NOT` yoktur. Sensör tipi değişirse tek satır düzeltilir.
- **Self-holding çalışma.** Konveyör, start'a bir kez basınca mühürlenir; stop veya E-stop bunu ezer.
- **E-stop CASE dışında, koşulsuz.** Her çevrim en başta kontrol edilir — hangi state'te olursa olsun yakalanır. Bir state dalına gömülseydi o dala girene kadar işlemezdi.
- **E-stop'tan çıkış otomatik değil.** İnsan onayı (reset) zorunludur — tehlike hâlâ mevcutken sistemin kendiliğinden hazır hale gelmesi engellenir.
- **Çıkışlar durumdan türetilir.** `convOut := running`, lambalar `step` değerinden. "Hangi geçişte kim neyi yaktı" izini sürmek yerine, her state'te ne olacağı tek bakışta görülür.

---

## Nasıl Çalıştırılır

1. `tia_project/PartCounter.zap15` dosyasını TIA Portal V15'te **Project → Retrieve** ile aç.
2. Projeyi derle, S7-PLCSIM'e yükle, CPU'yu RUN'a al.
3. NetToPLCsim'i başlat, PLC'yi ekle (127.0.0.1), server'ı başlat.
4. `factoryio/part_counter.factoryio` sahnesini Factory IO'da aç, S7-PLCSIM driver'ıyla bağla.
5. Sahnede Start'a bas — konveyör çalışır, parça sayar.

---

## Dosya Yapısı

```
01_part_counter/
├── README.md
├── src/
│   └── PartCounter.scl          # okunabilir kaynak kod
├── tia_project/
│   └── PartCounter.zap15        # TIA Portal arşivi (Retrieve ile aç)
├── factoryio/
│   └── part_counter.factoryio   # Factory IO sahnesi
└── media/
    └── PartCounter.mp4          # demo videosu
```

---

## Kaynak Kod

Tam SCL için: [`src/PartCounter.scl`](src/PartCounter.scl)

---

*Atılay Can Peker — [github.com/[CanPeker](https://github.com/[CanPeker])*
