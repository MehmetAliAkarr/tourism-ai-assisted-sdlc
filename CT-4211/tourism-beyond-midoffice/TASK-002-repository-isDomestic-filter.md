# TASK-002: Repository Katmanına IsDomestic Filtre Koşulunun Eklenmesi

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-5
> **Project**: tourism-beyond-midoffice
> **Layer**: Infrastructure
> **Depends On**: TASK-001
> **Estimated Complexity**: S

## Objective

`HotelRepository.HotelListForDataTableAsync()` metodundaki EF Core LINQ sorgusuna `IsDomestic` filtre koşulunun eklenmesi. Bu sayede "Yurtiçi" seçildiğinde yalnızca Türkiye (`CountryId == 218`) otelleri, "Yurtdışı" seçildiğinde Türkiye dışı oteller listelenecektir.

## Current State

- `MidOffice.Repository/HotelRepository.cs` dosyasındaki `HotelListForDataTableAsync()` metodu EF Core LINQ sorgusuyla çalışmaktadır.
- Mevcut filtreler sırasıyla: Region (hiyerarşik), HotelId, HotelChainId, SalesTypeId, StatusId, Name, HotelCategoryId, **CountryId**, HotelCityId, HotelType, ChannelManagerTypeId, TourismCertificateTypeId, CorporateAndIndividualSharedHotel, IsActivePricePlan, SelectedProviderIdList.
- `CountryId > 0` filtresi ile tek bir ülkeye göre filtreleme yapılmaktadır ancak "Türkiye dışı tümü" gibi ters bir gruplama mevcut değildir.
- Repository'de zaten `DBCountryEnum.Turkey` referansı kullanılmaktadır (bkz. `GetHotelListWithCountriesAsync` metodu: `x.CountryId == (int)DBCountryEnum.Turkey`).

## Target State

- Mevcut filtre koşullarının arasına (tercihen `CountryId` filtresinden sonra) `IsDomestic` koşulu eklenir.
- Filtre mantığı:
  - `IsDomestic == 1` → `query.Where(t => t.CountryId == (int)DBCountryEnum.Turkey)`
  - `IsDomestic == 2` → `query.Where(t => t.CountryId != (int)DBCountryEnum.Turkey)`
  - `IsDomestic == null` veya diğer değerler → filtre uygulanmaz

## Implementation Guidance

1. **`MidOffice.Repository/HotelRepository.cs`** dosyasını aç.
2. `HotelListForDataTableAsync()` metodunda mevcut `CountryId` filtresini bul:
   ```csharp
   if (filterDto.CountryId > 0)
       query = query.Where(t => t.CountryId == filterDto.CountryId);
   ```
3. Bu satırın hemen altına şu bloğu ekle:
   ```csharp
   if (filterDto.IsDomestic.HasValue)
   {
       if (filterDto.IsDomestic.Value == 1)
           query = query.Where(t => t.CountryId == (int)DBCountryEnum.Turkey);
       else if (filterDto.IsDomestic.Value == 2)
           query = query.Where(t => t.CountryId != (int)DBCountryEnum.Turkey);
   }
   ```
4. `DBCountryEnum` namespace'inin dosyada zaten import edilmiş olduğunu doğrula. Eğer import yoksa `using SeturTourismCommon;` veya ilgili namespace'i ekle.
5. **Magic number (218) kullanma** — proje genelindeki konvansiyona uygun olarak `(int)DBCountryEnum.Turkey` kullan.

## Acceptance Criteria

- [ ] `IsDomestic == 1` gönderildiğinde yalnızca `CountryId == 218` olan oteller listelenir.
- [ ] `IsDomestic == 2` gönderildiğinde yalnızca `CountryId != 218` olan oteller listelenir.
- [ ] `IsDomestic == null` gönderildiğinde tüm oteller listelenir (mevcut davranış korunur).
- [ ] Geçersiz değer (ör. `3`) gönderildiğinde filtre uygulanmaz.
- [ ] `DBCountryEnum.Turkey` enum referansı kullanılır, magic number (218) hardcode edilmez.
- [ ] `CountryId` filtresi ve `IsDomestic` filtresi birlikte çalışır (AND koşulu).

## Test Requirements

- [ ] Unit test: `IsDomestic = 1` → yalnızca `CountryId == 218` sonuçları
- [ ] Unit test: `IsDomestic = 2` → yalnızca `CountryId != 218` sonuçları
- [ ] Unit test: `IsDomestic = null` → tüm sonuçlar (filtre uygulanmaz)
- [ ] Unit test: `IsDomestic = 1` + `CountryId = 5` → boş sonuç (AND mantığı)
- Naming convention: `HotelListForDataTable_IsDomesticOne_ReturnsOnlyTurkeyHotels`
- Test framework: xUnit + Moq (proje konvansiyonu)
