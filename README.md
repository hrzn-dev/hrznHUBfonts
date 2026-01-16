# 🎮 Distorted Vigil - Item & Inventory System

## Genel Bakış

Unity 6.3 ve FishNet Networking için tam özellikli, modüler Item ve Inventory sistemi. Server-authoritative multiplayer desteği, özel item data persistence, ve mevcut interaction sisteminizle tam entegrasyon.

---

## ✨ Özellikler

### Core Systems
- ✅ **ScriptableObject Tabanlı Item Tanımları** - Kolay item oluşturma ve düzenleme
- ✅ **Runtime ItemInstance System** - Her item'ın kendi custom data'sı
- ✅ **Ağırlık Sistemi** - 1000 birim maksimum kapasite
- ✅ **Stackable Items** - `isStackable` boolean ile kontrol
- ✅ **Hotbar Backend** - 1-9 tuşları için slot assignment
- ✅ **FishNet Network Sync** - Server-authoritative, SyncList ile otomatik sync
- ✅ **Custom Data Persistence** - Item yere atılıp toplandığında data korunur

### World Integration
- ✅ **WorldItem Component** - IInteractable implementation
- ✅ **3 Standart Interaction**:
  - **Hold** - Sürükle, döndür, fırlat
  - **Take** - Envantere al
- ✅ **Physics Support** - Rigidbody ile gerçekçi fizik

### Example Items
- ✅ **MedkitItem** - 3 saniye hold-to-heal mechanic
- ✅ **FlashlightItem** - Toggle + battery system (customData ile şarj korunur)
- ✅ **KnifeItem** - Instant attack, raycast damage
- ✅ **DataDriverItem** - Sinyal storage, serialization
- ✅ **SignalDownloader** - Driver attachment, signal download

---

## 📁 Dosya Yapısı

```
Assets/Scripts/
├── Enums/
│   ├── InputState.cs
│   └── ItemEnums.cs
├── ScriptableObjects/Items/
│   └── ItemSO.cs
├── Items/
│   ├── ItemInstance.cs
│   ├── InventorySlot.cs
│   ├── WorldItem.cs
│   └── Types/
│       ├── MedkitItem.cs
│       ├── FlashlightItem.cs
│       ├── KnifeItem.cs
│       └── DataDriverItem.cs
├── Player/
│   ├── InventoryManager.cs
│   └── Health.cs
├── Managers/
│   └── ItemDatabase.cs
└── Systems/Environment/
    └── SignalDownloader.cs
```

---

## 🚀 Hızlı Başlangıç

### 1. ItemSO Oluşturma

```
Project → Sağ Tık → Create → Distorted Vigil → Items → Item Definition
```

Inspector'da ayarla:
- Item ID, Name, Description, Icon
- Weight, isStackable, itemType
- holdDuration (0 = instant, >0 = hold gerekli)
- worldPrefab, itemPrefab

### 2. Item Prefab Oluşturma

**Item Prefab** (Equipped/Inventory):
```
GameObject + ItemInstance subclass (MedkitItem, FlashlightItem, vb.)
```

**World Prefab** (Yere atılmış):
```
GameObject
├── NetworkObject
├── Rigidbody
├── Collider
├── WorldItem
└── ItemInstance subclass
```

### 3. Player Setup

```csharp
Player
├── InventoryManager (maxSlots: 20, maxWeight: 1000)
└── Health (maxHealth: 100)
```

### 4. ItemDatabase

Scene'e `ItemDatabase` GameObject ekle, Resources/Items/ klasöründen otomatik yükler.

---

## 💻 Kod Örnekleri

### Yeni Item Tipi Oluşturma

```csharp
public class MyCustomItem : ItemInstance
{
    [Header("Custom Settings")]
    [SerializeField] private float customValue = 10f;
    
    protected override void Awake()
    {
        base.Awake();
        
        // Custom data initialize
        if (!HasData("myData"))
        {
            SetData("myData", 100f);
        }
    }
    
    public override void OnUse(GameObject user, InteractionPacket packet)
    {
        var leftClick = packet.GetInput("LeftClick");
        
        if (packet.ActionTag == "OnPressed" && leftClick.GetBool())
        {
            // Item kullanımı
            float data = GetData<float>("myData");
            Debug.Log($"Item kullanıldı! Data: {data}");
        }
    }
    
    public override float GetHoldDuration() => 0f; // Instant
}
```

### Inventory'ye Item Ekleme

```csharp
InventoryManager inventory = player.GetComponent<InventoryManager>();
ItemInstance item = ...; // ItemDatabase'den al veya instantiate et

if (inventory.CanAddItem(item))
{
    inventory.TryAddItem(item);
}
```

### Custom Data Kullanımı

```csharp
// Veri kaydetme
item.SetData("batteryCharge", 75f);
item.SetData("signalList", mySignalList);

// Veri okuma
float charge = item.GetData<float>("batteryCharge", 100f);
List<SignalData> signals = item.GetData<List<SignalData>>("signalList");

// Veri kontrolü
if (item.HasData("batteryCharge"))
{
    // ...
}
```

---

## 🔌 Mevcut Sistem Entegrasyonu

### PlayerInteraction ile Uyumluluk

WorldItem, mevcut `IInteractable` interface'ini implement eder:

```csharp
public class WorldItem : NetworkBehaviour, IInteractable
{
    public string GetInteractableName() => itemInstance.itemSO.itemName;
    public bool IsExclusive => true;
    public List<InteractionOption> GetOptions(GameObject player) { ... }
    public void OnInteract(GameObject player, string optionId, InteractionPacket packet) { ... }
}
```

Yerdeki itemlar otomatik olarak PlayerInteraction sisteminizde görünür ve etkileşime açıktır.

---

## 🌐 Multiplayer (FishNet)

### Server-Authoritative Design

Tüm inventory değişiklikleri server'da validate edilir:

```csharp
[ServerRpc]
private void ServerAddItem(string instanceID, string itemID, int stackCount)
{
    // Ağırlık kontrolü
    if (currentWeight + itemWeight > maxWeight) return;
    
    // Item ekle
    slots[index] = InventorySlot.Create(item);
    
    // Client'lara bildir
    RpcInventoryChanged();
}
```

### Network Sync

- `SyncList<InventorySlot>` - Otomatik inventory sync
- `[SyncVar] float currentWeight` - Ağırlık sync
- `ObserversRpc` - UI güncellemeleri için

---

## 🎯 Kullanım Senaryoları

### Senaryo 1: Medkit Kullanımı

1. Player medkit'i inventory'den equip eder
2. Sol tık'a 3 saniye basılı tutar
3. Her frame sağlık restore edilir
4. 3 saniye sonra medkit tüketilir

### Senaryo 2: Fener Bataryası

1. Fener equip edilir
2. Sol tık ile açılır/kapanır
3. Açıkken her saniye batarya azalır
4. Batarya bitince otomatik kapanır
5. Fener yere atılıp toplandığında batarya korunur

### Senaryo 3: Data Driver + Signal Downloader

1. Player Data Driver'ı inventory'den yere atar
2. Driver'ı tutar (E basılı)
3. SignalDownloader'ın attachment zone'una yaklaştırır
4. Sağ tık ile döndürür → Driver takılır
5. "Sinyal İndir" seçeneğini 5 saniye tutar
6. Sinyal driver'ın customData'sına eklenir
7. Driver çıkarılır ve başka oyuncuya verilebilir (sinyal korunur!)

---

## 🛠️ Eksik Özellikler (Gelecek Güncellemeler)

- [ ] **ItemUsageHandler** - Equipped item kullanımı için
- [ ] **Input Action Map** - "ItemActions" action map integration
- [ ] **Inventory UI** - Slot display, drag-and-drop
- [ ] **Equip System** - Visual item equipping

---

## 📚 Daha Fazla Bilgi

- **Kurulum Rehberi**: `walkthrough.md`
- **Implementation Plan**: `implementation_plan.md`
- **Task Breakdown**: `task.md`

---

## 🤝 Katkıda Bulunma

Sorular, öneriler veya bug raporları için GitHub Issues kullanabilirsiniz.

---

## 📄 Lisans

Bu sistem Distorted Vigil projesi için özel olarak geliştirilmiştir.

---

**Developed with ❤️ by Claude (Antigravity AI Assistant)**
