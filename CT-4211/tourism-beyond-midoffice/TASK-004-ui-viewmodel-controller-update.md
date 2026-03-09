# TASK-004: UI ViewModel ve Controller Güncellemesi (IsDomestic Filtre Desteği)

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-2, FR-6
> **Project**: tourism-beyond-midoffice
> **Layer**: Presentation
> **Depends On**: TASK-001
> **Estimated Complexity**: M

## Objective

`HotelFilterViewModel` sınıfına `IsDomestic` filtre alanı ve dropdown listesinin eklenmesi. UI Controller'daki üç action'da (sayfa yükleme, DataTable POST, Excel export) `IsDomestic` değerinin doğru şekilde mapping'inin yapılması.

## Current State

### HotelFilterViewModel
- Dosya: `MidOffice.UI` projesinde (ViewModel klasöründe veya doğrudan Controller ile aynı alanda).
- Mevcut dropdown listeleri: `HotelCountryList`, `HotelCategoryList`, `StatusList`, `IsActivePricePlanList`, `CorporateAndIndividualSharedHotelList`, `TourismCertificateTypeList`, vb.
- `IsDomestic` veya `IsDomesticList` property'si mevcut değil.

### HotelController (UI)
- **HotelList() GET** (`MidOffice.UI/Controllers/HotelController.cs`, satır ~311–367): Sayfa yükleme action'ı. Dropdown listelerini (`IsActivePricePlanList`, `CorporateAndIndividualSharedHotelList`, vb.) initialize eder. `IsDomesticList` mevcut değil.
- **HotelListForDataTable() POST** (satır ~397–437): DataTable AJAX çağrısını karşılar. `HotelFilterViewModel` → `HotelFilterRequestModel` mapping yapılır. `IsDomestic` mapping'i mevcut değil.
- **SaveHotelListToExcelFile()** (satır ~370–395): Excel export action'ı. Aynı `HotelFilterViewModel` → `HotelFilterRequestModel` mapping yapılır. `IsDomestic` mapping'i mevcut değil.

## Target State

### HotelFilterViewModel'e eklenenler:
```csharp
public int? IsDomestic { get; set; }
public List<SelectListItem> IsDomesticList { get; set; }
```

### HotelController.HotelList() GET — IsDomesticList initialize:
```csharp
model.IsDomesticList = new List<SelectListItem>
{
    new() { Text = SiteResource.All, Value = "" },
    new() { Text = "Yurtiçi", Value = "1" },
    new() { Text = "Yurtdışı", Value = "2" }
};
```

### HotelController.HotelListForDataTable() POST — Mapping:
`HotelFilterRequestModel` object initializer'a ekleme:
```csharp
IsDomestic = filterModel.IsDomestic
```

### HotelController.SaveHotelListToExcelFile() — Mapping:
Aynı şekilde `FilterModel` içindeki `HotelFilterRequestModel` object initializer'a:
```csharp
IsDomestic = filterModel.IsDomestic
```

## Implementation Guidance

### Adım 1: HotelFilterViewModel güncellemesi
1. `HotelFilterViewModel` sınıfının dosyasını aç.
2. Mevcut `CorporateAndIndividualSharedHotel` property'sinden sonra ekle:
   ```csharp
   public int? IsDomestic { get; set; }
   public List<SelectListItem> IsDomesticList { get; set; }
   ```

### Adım 2: HotelList() GET action güncellemesi
1. `MidOffice.UI/Controllers/HotelController.cs` dosyasını aç.
2. `HotelList()` GET action'ında `return View(model)` satırından hemen önce, mevcut dropdown listelerinin initialize edildiği bölüme (ör. `TourismCertificateTypeList` satırından sonra) şu bloğu ekle:
   ```csharp
   model.IsDomesticList = new List<SelectListItem>
   {
       new() { Text = SiteResource.All, Value = "" },
       new() { Text = "Yurtiçi", Value = "1" },
       new() { Text = "Yurtdışı", Value = "2" }
   };
   ```
3. **Not**: Mevcut dropdown listelerinin aynı pattern'ini takip et (`SiteResource.All` kullanımı, `""` ile varsayılan "Tümü" seçeneği).

### Adım 3: HotelListForDataTable() POST action güncellemesi
1. Aynı dosyada `HotelListForDataTable()` POST action'ını bul.
2. `var fiterReq = new HotelFilterRequestModel() { ... }` bloğunda `CorporateAndIndividualSharedHotel` atamasından sonra ekle:
   ```csharp
   IsDomestic = filterModel.IsDomestic
   ```

### Adım 4: SaveHotelListToExcelFile() action güncellemesi
1. Aynı dosyada `SaveHotelListToExcelFile()` action'ını bul.
2. `FilterModel = new HotelFilterRequestModel() { ... }` bloğunda `CorporateAndIndividualSharedHotel` atamasından sonra ekle:
   ```csharp
   IsDomestic = filterModel.IsDomestic
   ```

## Acceptance Criteria

- [ ] `HotelFilterViewModel` sınıfında `int? IsDomestic` ve `List<SelectListItem> IsDomesticList` property'leri mevcut.
- [ ] `HotelList()` GET action'ında `IsDomesticList` başarıyla initialize ediliyor (Tümü/Yurtiçi/Yurtdışı).
- [ ] `HotelListForDataTable()` POST action'ında `IsDomestic` değeri `HotelFilterRequestModel`'e aktarılıyor.
- [ ] `SaveHotelListToExcelFile()` action'ında `IsDomestic` değeri `HotelFilterRequestModel`'e aktarılıyor.
- [ ] Varsayılan değer "Tümü" (`""` / `null`).
- [ ] Excel export'ta yurtiçi/yurtdışı filtresi uygulanmış veriler export ediliyor.

## Test Requirements

- [ ] Manuel test: Sayfa yüklendiğinde dropdown'da 3 seçenek görünür (Tümü, Yurtiçi, Yurtdışı).
- [ ] Manuel test: "Yurtdışı" seçilip filtre uygulandığında DataTable yalnızca yurtdışı otelleri gösterir.
- [ ] Manuel test: "Yurtiçi" seçilip Excel export yapıldığında dosyada yalnızca yurtiçi oteller bulunur.
- Naming convention: N/A (UI integration test, manuel doğrulama)
- Test framework: Manuel + xUnit (controller action unit test opsiyonel)
