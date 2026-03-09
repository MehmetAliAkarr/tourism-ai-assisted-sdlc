# TASK-003: Service Katmanında IsDomestic DTO Mapping Güncellemesi

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-4
> **Project**: tourism-beyond-midoffice
> **Layer**: Application
> **Depends On**: TASK-001
> **Estimated Complexity**: S

## Objective

`HotelService.HotelListForDataTableAsync()` metodundaki `HotelFilterRequestModel` → `HotelFilterDto` mapping'ine `IsDomestic` alanının eklenmesi. Bu sayede API katmanından gelen filtre değeri Repository katmanına doğru şekilde aktarılacaktır.

## Current State

- `MidOffice.Service/HotelService.cs` dosyasındaki `HotelListForDataTableAsync()` metodu, `HotelFilterRequestModel`'den `HotelFilterDto`'ya manuel mapping yapmaktadır (AutoMapper kullanılmıyor).
- Mapping bloğunda 17 property atanmaktadır; `IsDomestic` mevcut değildir.
- Mevcut mapping örneği:
  ```csharp
  var filterDto = new HotelFilterDto
  {
      Name = datatableRequestMode.FilterModel.Name,
      CountryId = datatableRequestMode.FilterModel.CountryId,
      CorporateAndIndividualSharedHotel = datatableRequestMode.FilterModel.CorporateAndIndividualSharedHotel
      // ... diğer alanlar
  };
  ```

## Target State

- Mapping bloğuna `IsDomestic = datatableRequestMode.FilterModel.IsDomestic` satırı eklenir.

## Implementation Guidance

1. **`MidOffice.Service/HotelService.cs`** dosyasını aç.
2. `HotelListForDataTableAsync()` metodundaki `new HotelFilterDto { ... }` bloğunu bul.
3. `CorporateAndIndividualSharedHotel` atamasından sonra şu satırı ekle:
   ```csharp
   IsDomestic = datatableRequestMode.FilterModel.IsDomestic
   ```
4. Projeyi derle ve hata olmadığını doğrula.

## Acceptance Criteria

- [ ] `HotelService.HotelListForDataTableAsync()` metodunda `IsDomestic` değeri `HotelFilterRequestModel`'den `HotelFilterDto`'ya aktarılır.
- [ ] `IsDomestic = null` gönderildiğinde `filterDto.IsDomestic` da `null` olur.
- [ ] `IsDomestic = 1` gönderildiğinde `filterDto.IsDomestic` da `1` olur.
- [ ] Proje başarıyla derleniyor.

## Test Requirements

- [ ] Unit test: `HotelListForDataTableAsync` çağrıldığında `IsDomestic` değerinin `HotelFilterDto`'ya doğru aktarıldığının doğrulanması (Moq ile repository mock'lanarak).
- Naming convention: `HotelListForDataTableAsync_WhenIsDomesticProvided_MapsToFilterDto`
- Test framework: xUnit + Moq (proje konvansiyonu)
