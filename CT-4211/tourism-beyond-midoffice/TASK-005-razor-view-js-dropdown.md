# TASK-005: Razor View ve JavaScript Güncellemesi (IsDomestic Dropdown)

> **PRD**: CT-4211/PRD-CT-4211.md
> **Functional Requirement**: FR-1
> **Project**: tourism-beyond-midoffice
> **Layer**: Presentation
> **Depends On**: TASK-004
> **Estimated Complexity**: S

## Objective

MidOffice otel listeleme sayfasının Razor view'ına (`HotelList.cshtml`) Yurtiçi/Yurtdışı dropdown bileşeninin eklenmesi ve DataTable AJAX çağrısına (`hotel-list.js`) `IsDomestic` parametresinin dahil edilmesi.

## Current State

### HotelList.cshtml
- Dosya: `MidOffice.UI/Views/Hotel/HotelList.cshtml`
- Filtre panelinde 3 kolon halinde filtre dropdown'ları bulunmaktadır.
- 3. kolonda (sağ taraf): Ülke (`ddlCountry`), Şehir (`ddlCity`), Bölge (hiyerarşik 3 seviye), Turizm Belgesi Tipi filtreleri yer almaktadır.
- Ülke dropdown'ı satır ~128–134 civarında:
  ```html
  <div class="form-group">
      <label class="control-label col-md-4">
          <label asp-for="HotelCountryHashId">@SiteResource.Country</label>
      </label>
      <div class="col-md-8">
          <select class="form-control" asp-for="HotelCountryHashId" 
                  asp-items="Model.HotelCountryList" id="ddlCountry"></select>
      </div>
  </div>
  ```

### hotel-list.js
- Dosya: `MidOffice.UI/wwwroot/js/hotel/hotel-list.js`
- DataTable AJAX `data` fonksiyonunda 20 filtre parametresi gönderilmektedir.
- `IsDomestic` parametresi mevcut değildir.
- Son parametre: `d.CorporateAndIndividualSharedHotel = $('#CorporateAndIndividualSharedHotel').val();`

## Target State

### HotelList.cshtml — Yeni Dropdown
Ülke filtresinin hemen üstüne veya altına (tercihen üstüne, çünkü konsept olarak daha geniş bir filtre) yeni bir `<select>` dropdown eklenir:

```html
<div class="form-group">
    <label class="control-label col-md-4">
        <label>Yurtiçi/Yurtdışı</label>
    </label>
    <div class="col-md-8">
        <select class="form-control" asp-for="IsDomestic" asp-items="Model.IsDomesticList" id="IsDomestic"></select>
    </div>
</div>
```

### hotel-list.js — AJAX Data Parametresi
`data` fonksiyonuna yeni satır eklenir:
```javascript
d.IsDomestic = $('#IsDomestic').val();
```

## Implementation Guidance

### Adım 1: HotelList.cshtml güncellemesi
1. `MidOffice.UI/Views/Hotel/HotelList.cshtml` dosyasını aç.
2. Ülke dropdown'ının bulunduğu `form-group` div'ini bul (`ddlCountry` ID'li select).
3. Bu `form-group` div'inin hemen **üstüne** yeni dropdown'ı ekle:
   ```html
   <div class="form-group">
       <label class="control-label col-md-4">
           <label>Yurtiçi/Yurtdışı</label>
       </label>
       <div class="col-md-8">
           <select class="form-control" asp-for="IsDomestic" asp-items="Model.IsDomesticList" id="IsDomestic"></select>
       </div>
   </div>
   ```
4. Mevcut filtrelerin HTML yapısını (`form-group`, `col-md-4` label, `col-md-8` input) birebir takip et.
5. `asp-for="IsDomestic"` binding'i `HotelFilterViewModel.IsDomestic` property'sine bağlanır.
6. `asp-items="Model.IsDomesticList"` binding'i TASK-004'te initialize edilen dropdown listesini kullanır.

### Adım 2: hotel-list.js güncellemesi
1. `MidOffice.UI/wwwroot/js/hotel/hotel-list.js` dosyasını aç.
2. DataTable AJAX konfigürasyonundaki `"data": function (d) { ... }` bloğunu bul.
3. Mevcut son satırdan sonra (ör. `d.CorporateAndIndividualSharedHotel` satırından sonra) ekle:
   ```javascript
   d.IsDomestic = $('#IsDomestic').val();
   ```

### Stil ve UX Notları
- Dropdown mevcut filtre stilini takip etmeli (`form-control` class).
- Label metni: "Yurtiçi/Yurtdışı" (Türkçe — MidOffice UI dili Türkçe).
- Varsayılan seçili değer: "Tümü" (value: `""`, TASK-004'te `IsDomesticList`'in ilk elemanı olarak ayarlandı).

## Acceptance Criteria

- [ ] `HotelList.cshtml` sayfasında Ülke filtresinin yakınında "Yurtiçi/Yurtdışı" label'ı ile bir dropdown görünür.
- [ ] Dropdown 3 seçenek içerir: Tümü (varsayılan), Yurtiçi, Yurtdışı.
- [ ] Sayfa ilk yüklendiğinde "Tümü" seçili gelir.
- [ ] DataTable AJAX çağrısında `IsDomestic` parametresi gönderilir.
- [ ] Dropdown değeri değiştirilip filtre uygulandığında DataTable doğru sonuçları gösterir.
- [ ] HTML yapısı mevcut filtre dropdown'larıyla tutarlı (`form-group`, `col-md-4/8` grid).

## Test Requirements

- [ ] Manuel test: Sayfa yüklendiğinde "Yurtiçi/Yurtdışı" dropdown'ı görünür ve varsayılan "Tümü" seçili.
- [ ] Manuel test: "Yurtdışı" seçilip Listele butonuna basıldığında, DataTable yalnızca yurtdışı otelleri gösterir.
- [ ] Manuel test: "Yurtiçi" seçildiğinde yalnızca Türkiye otelleri listelenir.
- [ ] Manuel test: "Tümü" seçildiğinde tüm oteller listelenir (mevcut davranış).
- [ ] Manuel test: Tarayıcı Developer Tools → Network sekmesinde AJAX isteğinde `IsDomestic` parametresinin gönderildiğinin doğrulanması.
- Naming convention: N/A (Manuel UI testi)
- Test framework: Manuel doğrulama
