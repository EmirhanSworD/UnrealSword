# UnrealSword — Unreal Engine 5.8 Oyun Geliştirme ve Bulmaca Projesi

Bu depo, **Unreal Engine 5.8** ile Blueprint mimarisi kullanılarak geliştirilen haftalık laboratuvar çalışmalarını (**H03, H04, H05, H06**) ve **Vize Ödevi #1 (`L_Vize_Bulmaca`)** projesini içerir.

---

## Proje Genel Yapısı

```text
UnrealSword/
├── Config/                      # Motor, giriş ve Unreal MCP ayarları
├── Content/
│   └── 23020021056/
│       ├── H03/                 # Akış Parkuru & Ganimet Sandığı (Blueprint akış kontrolü, enum, nadirlik)
│       ├── H04/                 # Gözcü & Mıknatıslı Altın (Vektör matematiği, Dot Product)
│       ├── H05/                 # Bulmaca Oyunu (Lab #5) & Vize Ödevi #1 (L_Vize_Bulmaca)
│       │   ├── Blueprints/      # BP_Puzzle* ve BP_Vize* oyun sınıfları
│       │   ├── Maps/            # L_H05_Bulmaca ve L_Vize_Bulmaca
│       │   ├── Materials/       # MI_Gold, MI_Copper, MI_Nickel, MI_Highlight, MI_Bomb
│       │   └── UI/              # W_PuzzleHUD ve W_VizeHUD
│       └── H06/                 # İletişim Parkuru (Event Dispatcher: Plaka, Kapı, Lamba)
├── Sword.uproject               # Unreal Engine 5.8 proje dosyası
└── Vize_Odevi_1_Mekanik_Ozeti.docx
```

---

## Hafta 05 (H05) — Bulmaca Oyunu Detaylı Açıklaması

**Hafta 05**, hazır bir oyun şablonu kullanmadan mevcut proje içerisinde **sıfırdan bir 3B Bulmaca Oyunu iskeleti** kurmayı amaçlar. Oyun; fare etkileşimi, 2 boyutlu ızgaranın 1 boyutlu dizide (Array) yönetimi, periyodik zamanlayıcılar (Timer), eşleşme algoritması, kırılma animasyonu ve UMG HUD arayüzünden oluşur.

### Temel Oyun Kuralları ve Oynanış (`L_H05_Bulmaca`)
- **4×4 Izgara Yapısı:** Toplam 16 blok, `300 cm` aralıkla `(0, 0, 0)` başlangıç noktasından itibaren yatay ve dikey eksende dizilir. Üstten bakan sabit bir `CameraActor` (`450, 450, 1500`, `Pitch = -90°`) tahtayı tam karşıdan görüntüler.
- **Blok Tipleri ve Erişilebilirlik:** Her bloğun 3 farklı tipi vardır: **A (Altın — `MI_Gold`)**, **B (Bakır — `MI_Copper`)**, **C (Nikel — `MI_Nickel`)**. Renk körlüğü erişilebilirliği için her bloğun üzerinde `TextRender` ile tip harfi (`A`, `B`, `C`) yer alır.
- **Fare Etkileşimi ve Hover:** İmleç bloğun üzerine geldiğinde ölçek büyür (`2.5 → 2.75`), tıklandığında blok seçili hale geçer (`MI_Highlight` rengi ve `+30 cm` yükselme). Aynı bloğa art arda tıklanmasını önlemek için seçim kilidi kullanılır.
- **Eşleşme ve Kırılma:** Aynı tipteki **3 blok yatay veya dikey ardışık** seçildiğinde eşleşme gerçekleşir. Bloklar adım adım küçülerek kırılır, **+30 puan** eklenir ve `0.7 sn` sonra yerlerine yeni rastgele bloklar oluşturulur. Geçersiz seçimde tüm seçimler geri alınır.
- **Rastgele Tip Değişimi ve Geri Sayım:** Seçili olmayan bloklar her **2 saniyede** bir rastgele yeni tip alır. Oyun süresi **60 saniyedir**; süre sıfırlandığında tüm zamanlayıcılar durdurulur, tıklamalar kilitlenir ve ekranda kırmızı **"OYUN BİTTİ"** uyarısı gösterilir.
- **Sağ Tık İptal:** Oyuncu istediği an sağ tıklayarak mevcut blok seçimini temizleyebilir.

### H05 Mimari ve Blueprint Sınıfları
| Blueprint / Asset | Görevi | Önemli Bileşen, Fonksiyon ve Mantık |
| :--- | :--- | :--- |
| **`BP_PuzzleGameMode`** | Oyun kuralları ve varsayılan sınıflar | `PlayerController = BP_PuzzlePlayerController`, `DefaultPawnClass = None` |
| **`BP_PuzzlePlayerController`** | Fare girişi, kamera ve sağ tık kontrolü | `ShowMouseCursor`, `EnableClickEvents`, `EnableMouseOverEvents`; `BeginPlay` kamera bağlama; `RightMouseButton → ClearSelection` |
| **`BP_PuzzleBlock`** | Tekil bulmaca bloğu | `SM_ChamferCube` karo + `Label` (`NoCollision`); `AssignBlockType`, `SetSelected`, `MakeSelectable`, `BreakBlock` |
| **`BP_PuzzleGrid`** | 4×4 tahta yönetimi, eşleşme ve zamanlayıcılar | `CreateGrid`, `CreateBlock`, `BlockSelected`, `AreSameType`, `AreInLine`, `BreakMatched`, `RefillBlocks`, `ClearSelection`, `UpdateHud`, `EndGame` |
| **`W_PuzzleHUD`** | 2B ekran arayüzü (UMG) | `txtScore` (`Puan: 0`), `txtTime` (`Süre: 60`), `txtGameOver` (`OYUN BİTTİ`) |

### 2B Izgara ↔ 1B Array Matematiksel Dönüşümü
4×4 ızgaradaki 16 blok tek boyutlu `Blocks` dizisinde saklanır:
- **İleri dönüşüm:** `Index = Satır × Size + Sütun`
- **Ters dönüşüm:** `Satır = Index / Size` (tam sayı bölme), `Sütun = Index % Size` (mod alma)
- **Hat Doğrulama (`AreInLine`):** Yalnızca ardışık index bakmak `2, 3, 4` gibi farklı satırlara taşan üçlülerde hatalı sonuç verir. Bu nedenle yatayda **aynı satır + `Max - Min == 2`**, dikeyde **aynı sütun + `Max - Min == 2 × Size`** şartları birlikte doğrulanır.

---

## Vize Ödevi #1 — Genişletilmiş Bulmaca Oyunu (`L_Vize_Bulmaca`)

Lab #5 haritası (`L_H05_Bulmaca`) korunarak oluşturulan **`L_Vize_Bulmaca`** haritasında oyunun çekirdek oynanışını dönüştüren **1 Ana Mekanik** ve onu destekleyen **1 Yan Mekanik** geliştirilmiştir:

### Ana Mekanik: Yerçekimi ve Zincirleme Düşüş (Gravity & Cascade)
- Eşleşen bloklar kırıldığında `BP_VizePuzzleGrid.ApplyGravityAndRefill` her sütunu alttan üste tarar; üstteki sağlam bloklar `BP_VizePuzzleBlock.StartFallTo` ve `EventTick` içindeki `VInterpTo` ile boşluklara akıcı biçimde kayar, üstten yeni bloklar düşer.
- Düşüş sonrası `CheckCascadeMatches` tüm 4×4 tahtayı yatay ve dikey 3'lü eşleşmeler için otomatik tarar. Yeni eşleşme yakalandığında `CascadeCount` artarak otomatik zincirleme kırılma tetiklenir ve katlanan bonus puan verilir.

### Yan Mekanik: Bomba Blok (Bomb Block)
- Yeni doğan bloklar `%22` olasılıkla (`BombChancePercent = 22`) **Bomba Blok** olur; ayrıca oyun başında merkez slotta garantili bir Bomba Blok bulunur.
- Bomba bloklar kırmızı `MI_Bomb` materyali ve erişilebilir `*A` / `*B` / `*C` etiketiyle ayırt edilir.
- Eşleşmeye (veya zincir kırılmaya) dahil olan Bomba Blok, `ExpandBombs` ile çevresindeki **3×3** komşu alanı patlatır (alandaki diğer bombalar da `BombDetonated` kontrolüyle zincirleme patlar), patlama animasyonu oynatır ve bomba başına **+50 ekstra puan** kazandırır.

### Vize Puanlama ve Arayüz (`W_VizeHUD`)
- **Puan Formülü:** `(Kırılan Blok Sayısı × 10 × CascadeCount) + (Patlayan Bomba Sayısı × 50)`
- **`txtStatus` Bildirim Bandı:** Eşleşme, `"BOMBA PATLADI! +..."` ve `"ZINCIR x2! +..."` durumlarını canlı olarak ekranda gösterir.

---

## Diğer Haftaların Kısa Özeti

- **Hafta 03 (`H03` — Akış Parkuru & Ganimet Sandığı):** `L_H03_AkisParkuru` (`BP_DifficultyPanel`, `BP_HealthStation`, `BP_MaterialDisplay`, `BP_Mine`) ve `L_H03_GanimetSandigi` (`BP_LootChest`, `E_Rarity`) ile akış kontrolü, fonksiyon bölme ve rastgele nadirlik hesaplaması.
- **Hafta 04 (`H04` — Gözcü & Mıknatıs):** `L_H04_GozcuMiknatis` ve `L_H04_VektorDemo` üzerinde `BP_Watcher` (Dot Product görüş açısı algılama) ve `BP_MagnetCoin` (vektör mesafesiyle çekim hareketi).
- **Hafta 06 (`H06` — Blueprint İletişimi):** `L_H06_Iletisim` üzerinde `BP_PressurePlate`, `BP_Door` ve `BP_Lamp` sınıflarının `Event Dispatcher` (`OnPlatePressed`, `OnPlateReleased`, `OnOpened`) mimarisiyle gevşek bağlı (decoupled) haberleşmesi.

---

## Çalıştırma Talimatları
1. **Unreal Engine 5.8** ile `Sword.uproject` dosyasını açın.
2. Temel Lab #5 bulmaca oyunu için `Content/23020021056/H05/Maps/L_H05_Bulmaca`, Vize Ödevi #1 için `Content/23020021056/H05/Maps/L_Vize_Bulmaca` haritasını açıp **Play** butonuna basın.
3. **Kontroller:** Sol tık ile 3 eşleşen blok seçin; sağ tık ile mevcut seçimi iptal edin.
