# TASK-006: Unit Test — IsDomestic Filtresi için Repository ve Service Testleri

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-3, FR-4, FR-5
> **Project**: tourism-beyond-midoffice
> **Layer**: Test
> **Depends On**: TASK-002, TASK-003
> **Estimated Complexity**: M

## Objective

`HotelRepository.HotelListForDataTableAsync()` ve `HotelService.HotelListForDataTableAsync()` metotlarında IsDomestic filtresinin doğru çalıştığını doğrulayan unit testlerin yazılması.

## Current State

- Test projesi: `MidOffice.Api.Tests/MidOffice.Api.Tests.csproj` — mevcut ancak minimal düzeyde (xUnit framework'ü yapılandırılmış).
- `HotelRepository` ve `HotelService` için mevcut test dosyaları sınırlı olabilir.
- Repository EF Core `DbContext` kullanmaktadır ve testlerde In-Memory veya mock provider ile test edilebilir.
- Service katmanı constructor injection kullanmaktadır — repository bağımlılığı Moq ile mock'lanabilir.

## Target State

Aşağıdaki test senaryolarını kapsayan unit testler yazılır:

### Repository Testleri (`HotelRepositoryTests`)
1. `HotelListForDataTable_IsDomesticOne_ReturnsOnlyTurkeyHotels` — `IsDomestic = 1` → yalnızca `CountryId == 218` oteller döner.
2. `HotelListForDataTable_IsDomesticTwo_ReturnsOnlyInternationalHotels` — `IsDomestic = 2` → yalnızca `CountryId != 218` oteller döner.
3. `HotelListForDataTable_IsDomesticNull_ReturnsAllHotels` — `IsDomestic = null` → tüm oteller döner.
4. `HotelListForDataTable_IsDomesticWithCountryFilter_AppliesBothFilters` — `IsDomestic = 2` + `CountryId = 5` → yalnızca `CountryId == 5` oteller döner.

### Service Testleri (`HotelServiceTests`)
5. `HotelListForDataTableAsync_WhenIsDomesticProvided_MapsToFilterDto` — `IsDomestic` değerinin `HotelFilterRequestModel` → `HotelFilterDto` mapping'inde korunduğunun doğrulanması.

## Implementation Guidance

### Adım 1: Test projesi hazırlığı
1. `MidOffice.Api.Tests` projesine aşağıdaki NuGet paketlerinin mevcut olduğunu doğrula:
   - `xunit`
   - `xunit.runner.visualstudio`
   - `Moq`
   - `Microsoft.EntityFrameworkCore.InMemory` (Repository testleri için)
2. Proje referansları: `MidOffice.Repository`, `MidOffice.Service`, `MidOffice.Domain`, `MidOffice.Api.RequestModel` projelerine referans ekle.

### Adım 2: Repository test dosyası oluşturma
1. `MidOffice.Api.Tests/Repository/HotelRepositoryTests.cs` dosyası oluştur.
2. EF Core In-Memory Database ile test DbContext'i yapılandır.
3. Seed data ekle:
   ```csharp
   // Turkey hotels (CountryId = 218)
   new Hotel { Id = 1, Name = "Antalya Palace", CountryId = 218, IsSoftDeleted = false, ... },
   new Hotel { Id = 2, Name = "İstanbul Grand", CountryId = 218, IsSoftDeleted = false, ... },
   // International hotels
   new Hotel { Id = 3, Name = "Athens Hilton", CountryId = 85, IsSoftDeleted = false, ... },
   new Hotel { Id = 4, Name = "Rome Marriott", CountryId = 107, IsSoftDeleted = false, ... }
   ```
4. Her test senaryosu için `HotelFilterDto` oluştur, `HotelListForDataTableAsync()` çağır, sonuçları assert et.

### Adım 3: Service test dosyası oluşturma
1. `MidOffice.Api.Tests/Service/HotelServiceTests.cs` dosyası oluştur.
2. `IHotelRepository` interface'ini Moq ile mock'la.
3. `HotelListForDataTableAsync()` çağrısında `IsDomestic` değerinin repository'ye doğru aktarıldığını `Callback` ile doğrula.

### Test Convention
- Format: `{MethodName}_{Condition}_{ExpectedResult}`
- Framework: xUnit (`[Fact]`, `[Theory]`)
- Mocking: Moq
- DB Mock: EF Core InMemory Provider

## Acceptance Criteria

- [ ] En az 5 unit test yazılmış ve tümü başarıyla geçiyor.
- [ ] `IsDomestic = 1` senaryosu yalnızca Türkiye otellerini döndürüyor.
- [ ] `IsDomestic = 2` senaryosu yalnızca uluslararası otelleri döndürüyor.
- [ ] `IsDomestic = null` senaryosu tüm otelleri döndürüyor.
- [ ] Service DTO mapping testi `IsDomestic` aktarımını doğruluyor.
- [ ] Testler `dotnet test` komutuyla CI pipeline'da çalışabilir durumda.

## Test Requirements

- [ ] Unit test: `HotelListForDataTable_IsDomesticOne_ReturnsOnlyTurkeyHotels`
- [ ] Unit test: `HotelListForDataTable_IsDomesticTwo_ReturnsOnlyInternationalHotels`
- [ ] Unit test: `HotelListForDataTable_IsDomesticNull_ReturnsAllHotels`
- [ ] Unit test: `HotelListForDataTable_IsDomesticWithCountryFilter_AppliesBothFilters`
- [ ] Unit test: `HotelListForDataTableAsync_WhenIsDomesticProvided_MapsToFilterDto`
- Naming convention: `{MethodName}_{Condition}_{ExpectedResult}`
- Test framework: xUnit + Moq + EF Core InMemory
