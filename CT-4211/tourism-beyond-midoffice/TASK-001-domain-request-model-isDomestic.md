# TASK-001: Domain ve API Request Model'e IsDomestic Filtre Alanı Eklenmesi

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-3, FR-4
> **Project**: tourism-beyond-midoffice
> **Layer**: Domain
> **Depends On**: None
> **Estimated Complexity**: S

## Objective

`HotelFilterRequestModel` (API katmanı) ve `HotelFilterDto` (Domain katmanı) sınıflarına `IsDomestic` property'sinin eklenmesi. Bu task, filtreleme zincirinin temel veri modelini oluşturur ve diğer tüm task'ların bağımlılığıdır.

## Current State

- `HotelFilterRequestModel` sınıfı (`MidOffice.Api.RequestModel/Hotel/HotelFilterRequestModel.cs`): 19 filtre property içerir (`HotelId`, `Name`, `CountryId`, `HotelCityId`, vb.). `IsDomestic` alanı mevcut değil.
- `HotelFilterDto` sınıfı (`MidOffice.Domain/FilterDto/HotelFilterDto.cs`): Aynı filtre alanlarının Domain katmanı karşılığı. `IsDomestic` alanı mevcut değil.

## Target State

- Her iki sınıfa da `public int? IsDomestic { get; set; }` property eklenir.
- Property nullable (`int?`) olacak — `null` = filtre yok (Tümü), `1` = Yurtiçi, `2` = Yurtdışı.
- Mevcut API kontratı geriye dönük uyumlu kalır (nullable property gönderilmezse varsayılan `null` olacak).

## Implementation Guidance

1. **`MidOffice.Api.RequestModel/Hotel/HotelFilterRequestModel.cs`** dosyasını aç.
2. `CorporateAndIndividualSharedHotel` property'sinden sonra şu satırı ekle:
   ```csharp
   public int? IsDomestic { get; set; }
   ```
3. **`MidOffice.Domain/FilterDto/HotelFilterDto.cs`** dosyasını aç.
4. `CorporateAndIndividualSharedHotel` property'sinden sonra aynı satırı ekle:
   ```csharp
   public int? IsDomestic { get; set; }
   ```
5. Projeyi derle ve hata olmadığını doğrula.

## Acceptance Criteria

- [ ] `HotelFilterRequestModel` sınıfında `public int? IsDomestic { get; set; }` property mevcut.
- [ ] `HotelFilterDto` sınıfında `public int? IsDomestic { get; set; }` property mevcut.
- [ ] Her iki property de `int?` (nullable) tipinde.
- [ ] Proje başarıyla derleniyor, mevcut testler kırılmıyor.

## Test Requirements

- [ ] Unit test: `HotelFilterRequestModel` ve `HotelFilterDto` sınıflarında `IsDomestic` property'sinin `null`, `1`, `2` değerlerini kabul ettiğinin doğrulanması.
- Naming convention: `IsDomestic_WhenNull_DefaultsToNull`, `IsDomestic_WhenSetToOne_ReturnsDomestic`
- Test framework: xUnit + Moq (proje konvansiyonu)
