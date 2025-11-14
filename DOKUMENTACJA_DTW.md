# Dokumentacja API DTW - Custom Integrator

## Wprowadzenie

System API oparty na architekturze REST działa jako pośrednik pomiędzy zewnętrznymi klientami a wewnętrznym systemem WMS. Nasłuchuje żądań HTTP, umożliwiając integrację zewnętrznych systemów z magazynem.

API przyjmuje komunikaty dotyczące:
- Zamówień
- Awizacji dostaw
- Dokumentów WZ
- Danych masterdata produktów

## Autoryzacja

Wszystkie requesty wymagają autoryzacji poprzez statyczny token w headerze:

```
auth-key: <YOUR_TOKEN>
```

**Uwagi:**
- Klucz generowany przez pracowników DTW
- Token jest statyczny i nie wygasa
- Musi być dodany do każdego requestu

### Przykład nagłówków

| Key | Value |
|-----|-------|
| Content-Type | application/json |
| auth-key | test_key |

---

## Endpointy

System ma otwarte cztery endpointy. Wszystkie używają metody **POST**.

### 1. AddOrder
**Endpoint:** `{Address}/AddOrder`  
**Metoda:** POST  
**Format:** JSON  
**Opis:** Dodawanie zamówień do systemu WMS

#### Struktura JSON

```json
{
  "OrderHeader": {
    "CustomerOrderCode": "string",
    "AgencyCode": "string",
    "PickRemarks": "string",
    "PackRemarks": "string",
    "ExpShipDate": "YYYY-MM-DD",
    "IsB2B": boolean,
    "Customer": {
      "FullName": "string",
      "Address": "string",
      "City": "string",
      "ZipCode": "string",
      "CountryCode": "string",
      "Phone": "string",
      "Email": "string"
    }
  },
  "Lines": [
    {
      "ItemCode": "string",
      "Quantity": int,
      "ExpDate": "string",
      "Lot": "string"
    }
  ]
}
```

#### Opis pól

**OrderHeader:**
- `CustomerOrderCode` (**wymagane**) - Kod zamówienia klienta
- `AgencyCode` (opcjonalne) - Kod dostawcy (np. DPD, DHL, UPS). Wartość domyślna: UPS
- `PickRemarks` (opcjonalne) - Notatki dotyczące zbiórki
- `PackRemarks` (opcjonalne) - Notatki dotyczące pakowania
- `ExpShipDate` (**wymagane**) - Data przewidywanego odbioru w formacie `YYYY-MM-DD`
- `IsB2B` (**wymagane**) - Boolean (true/false). Wartość domyślna: false

**Customer:**
- `FullName` (**wymagane**) - Nazwa/imię i nazwisko zamawiającego
- `Address` (**wymagane**) - Adres wysyłki
- `City` (**wymagane**) - Miasto wysyłki
- `ZipCode` (**wymagane**) - Kod pocztowy
- `CountryCode` (opcjonalne) - Kod kraju (np. "PL")
- `Phone` (**wymagane**) - Numer telefonu
- `Email` (**wymagane**) - Adres e-mail

**Lines:**
- `ItemCode` (**wymagane**) - Kod produktu zarejestrowanego w masterdata DTW
- `Quantity` (**wymagane**) - Ilość (INT)
- `ExpDate` (opcjonalne) - Data ważności
- `Lot` (opcjonalne) - Partia/stock

#### Przykład requestu

```json
{
  "OrderHeader": {
    "CustomerOrderCode": "CustomerOrderNumber20-20-11",
    "AgencyCode": "DHL",
    "ExpShipDate": "2025-12-28",
    "Customer": {
      "FullName": "First Name Last Name",
      "Address": "Street 12A",
      "City": "CityName",
      "ZipCode": "11-222",
      "CountryCode": "PL",
      "Phone": "123-321-312",
      "Email": "test-mail@test.com",
      "IsB2B": false
    }
  },
  "Lines": [
    {
      "ItemCode": "registered_item_code",
      "Quantity": 20
    },
    {
      "ItemCode": "registered_item_code",
      "Quantity": 20
    },
    {
      "ItemCode": "registered_item_code",
      "Quantity": 20
    }
  ]
}
```

#### Odpowiedź sukcesu (HTTP 200)

```json
{
  "WMSOrder": "OR11626472",
  "CustomerOrderNumber": "CustomerOrderNumber28-20-11",
  "Message": "Order OR11626472 has been registered in WMS"
}
```

#### Walidacja i błędy

**Warunki walidacji:**

1. **CustomerOrderCode musi być unikatowy** dla danego klienta

**HTTP 400:**
```
"The CustomerOrderCode CustomerOrderCode already exists for current depositor and is associated with the OrderCode WMS order Code"
```

2. **ItemCode musi być wcześniej zarejestrowany** w systemie WMS

**HTTP 400:**
```
"Product code was found which does not exist in WMS! Product Code list not found in WMS: missingProducts"
```

**Inne błędy (HTTP 400):**
- `"An error occured while binding the model"` - Błąd mapowania modelu JSON
- `"Something went wrong while processing the file to the internal WMS system"` - Błąd przekazywania do WMS

---

### 2. AddReceipt
**Endpoint:** `{Address}/AddReceipt`  
**Metoda:** POST  
**Format:** JSON  
**Opis:** Dodawanie awizacji dostaw

#### Struktura JSON

```json
{
  "ReceiptHeader": {
    "ReceiptNumber": "string",
    "Supplier": "string",
    "ExpectedDate": "YYYY-MM-DD",
    "RefDocNo": "string",
    "Remarks": "string",
    "CustomerOrderCode": "string"
  },
  "ReceiptLines": [
    {
      "ItemCode": "string",
      "itemDescr": "string",
      "Quantity": int,
      "DetailRemarks": "string",
      "SSCC": "string"
    }
  ]
}
```

#### Opis pól

**ReceiptHeader:**
- `ReceiptNumber` (**wymagane**) - Numer zamówienia
- `Supplier` (opcjonalne) - Nazwa dostawcy (np. "DPD"). Wartość domyślna: Ravenol
- `ExpectedDate` (**wymagane**) - Data przewidywanej dostawy w formacie `YYYY-MM-DD`
- `RefDocNo` (opcjonalne) - Numer dokumentu referencyjnego
- `Remarks` (opcjonalne) - Notatka awizacji
- `CustomerOrderCode` (opcjonalne) - Numer zamówienia klienta

**ReceiptLines:**
- `ItemCode` (**wymagane**) - Kod produktu zarejestrowanego w masterdata DTW
- `itemDescr` (**wymagane**) - Opis produktu
- `Quantity` (**wymagane**) - Ilość (INT)
- `DetailRemarks` (opcjonalne) - Notatka dla linii
- `SSCC` (opcjonalne) - Numer SSCC

#### Przykład requestu

```json
{
  "ReceiptHeader": {
    "ReceiptNumber": "New Receipt - 12",
    "Supplier": "DPD",
    "ExpectedDate": "2025-12-25",
    "Remarks": "Just for test",
    "CustomerOrderCode": "test-number-code"
  },
  "ReceiptLines": [
    {
      "ItemCode": "New Item",
      "Quantity": 20,
      "DetailRemarks": "New Receipt"
    },
    {
      "ItemCode": "New Item",
      "Quantity": 20,
      "DetailRemarks": "New Receipt"
    }
  ]
}
```

#### Odpowiedź sukcesu (HTTP 200)

```json
{
  "WmsReceipt": "test-receipt",
  "Message": "Receipt has been registered in WMS"
}
```

#### Walidacja i błędy

**Warunki walidacji:**

1. **ReceiptNumber musi być unikatowy** dla danego klienta

**HTTP 400:**
```
"The ReceiptNumber already exists for current depositor and is associated with the WMSReceiptNumber"
```

2. **ItemCode musi być wcześniej zarejestrowany** w systemie WMS

**HTTP 400:**
```
"Product code was found which does not exist in WMS! Product Code list not found in WMS: missingProducts"
```

**Inne błędy (HTTP 400):**
- `"An error occured while binding the model"` - Błąd mapowania modelu JSON
- `"Something went wrong while processing the file to the internal WMS system"` - Błąd przekazywania do WMS

---

### 3. UploadGoodsIssueNote
**Endpoint:** `{Address}/UploadGoodsIssueNote`  
**Metoda:** POST  
**Format:** FORM Data  
**Opis:** Przekazywanie dokumentów WZ (wydanie zewnętrzne)

#### Struktura FORM Data

| Pole | Typ | Opis |
|------|-----|------|
| OrderNumber | string (opcjonalne) | Numer zamówienia WMS |
| CustomerOrderCode | string (opcjonalne) | Numer zamówienia klienta |
| GoodIssueNote | File | Plik PDF (max 3MB) |

#### Wymagania pliku

- **Format:** PDF
- **Maksymalny rozmiar:** 3 MB
- Inne formaty zostaną odrzucone

#### Struktura formy (przykład z Postman)

| Key | Value | Type |
|-----|-------|------|
| OrderNumber | TestOrderCodeWms | Text |
| GoodIssueNote | test-2.pdf | File |
| CustomerOrderCode | TestOrderCodeCustomer | Text |

#### Walidacja

**Warunki:**
- Co najmniej jedno pole (`OrderNumber` lub `CustomerOrderCode`) musi być wypełnione
- Jeśli wypełnione są oba pola, następuje weryfikacja zgodności
- Jeśli wypełnione jest tylko jedno pole, weryfikacja odbywa się tylko dla tego pola

#### Błędy (HTTP 400)

```
"Both fields CustomerOrderCode and OrderNumber must not be empty"
```
- Występuje gdy oba pola są puste

```
"An error occured while binding the model"
```
- Błąd mapowania formy (np. brak załączonego pliku)

```
"File of this extension is not allowed"
```
- System wykrył plik z niedozwolonym rozszerzeniem

---

### 4. UploadMasterData
**Endpoint:** `{Address}/UploadMasterData`  
**Metoda:** POST  
**Format:** JSON  
**Opis:** Przekazywanie danych produktowych (masterdata)

#### Struktura JSON

```json
{
  "MasterDataItemsList": [
    {
      "ProductCode": "string",
      "ItemDescription": "string",
      "BarCode": "string",
      "Weight": numeric,
      "Height": numeric,
      "Length": numeric,
      "Liters": int,
      "ProdGroup": "string",
      "ShelfLife": int,
      "MinimumShelfLife": int,
      "CapCountAfterPicking": int,
      "QuantityPerCarton": int
    }
  ]
}
```

#### Opis pól

- `ProductCode` (**wymagane**) - Kod produktu, musi być unikatowy
- `ItemDescription` (**wymagane**) - Opis produktu
- `BarCode` (**wymagane**) - Kod kreskowy
- `Weight` (**wymagane**) - Waga w kg (numeric, np. 0.2)
- `Height` (opcjonalne) - Wysokość w metrach (numeric, np. 0.2)
- `Length` (opcjonalne) - Długość w metrach (numeric, np. 0.2)
- `Liters` (opcjonalne) - Objętość w litrach (int)
- `ProdGroup` (opcjonalne) - Grupa produktowa
- `ShelfLife` (opcjonalne) - Okres trwałości (int)
- `MinimumShelfLife` (opcjonalne) - Minimalny okres trwałości (int)
- `CapCountAfterPicking` (opcjonalne) - Licznik po pobraniu (int)
- `QuantityPerCarton` (opcjonalne) - Ilość w kartonie (int)

#### Przykład requestu

```json
{
  "MasterDataItemsList": [
    {
      "ProductCode": "test-product-12",
      "ItemDescription": "test-product",
      "BarCode": "20003242142352321",
      "Weight": 0.002,
      "Height": 0.12,
      "Length": 2.2,
      "Liters": 2,
      "ProdGroup": "test",
      "ShelfLife": 222,
      "MinimumShelfLife": 120,
      "CapCountAfterPicking": 200,
      "QuantityPerCarton": 2
    }
  ]
}
```

#### Odpowiedź sukcesu (HTTP 200)

```
"Master data succesfully uploaded to the WMS system"
```

#### Walidacja i błędy

**ProductCode musi być unikatowy**

**HTTP 400:**
```
"Product code has been already found in the WMS system. Please remove it from the list: test-product"
```

**Uwaga:** Lista produktów nie zostanie zapisana w bazie danych, jeśli którykolwiek ProductCode już istnieje.

---

## Błędy ogólne

### HTTP 500 - Internal Server Error

Każdy endpoint może zwrócić błąd HTTP 500:

```
"Undefined error"
```

Występuje gdy podczas przetwarzania żądania wystąpił nieoczekiwany błąd po stronie serwera.

---

## Podsumowanie endpointów

| Endpoint | Metoda | Format | Opis |
|----------|--------|--------|------|
| /AddOrder | POST | JSON | Dodawanie zamówień |
| /AddReceipt | POST | JSON | Dodawanie awizacji |
| /UploadGoodsIssueNote | POST | FORM Data | Przekazywanie dokumentów WZ |
| /UploadMasterData | POST | JSON | Przekazywanie danych produktowych |

---

## Uwagi implementacyjne

1. **Autoryzacja:** Wszystkie requesty wymagają headera `auth-key`
2. **Format dat:** Zawsze używaj formatu `YYYY-MM-DD`
3. **Unikalne klucze:**
   - `CustomerOrderCode` musi być unikatowy w `/AddOrder`
   - `ReceiptNumber` musi być unikatowy w `/AddReceipt`
   - `ProductCode` musi być unikatowy w `/UploadMasterData`
4. **Walidacja produktów:** Wszystkie `ItemCode` muszą istnieć w systemie (dodane przez `/UploadMasterData`)
5. **Limity plików:** Dokumenty WZ max 3MB, tylko format PDF
6. **Wartości domyślne:**
   - `AgencyCode`: UPS
   - `Supplier`: Ravenol
   - `IsB2B`: false

---

## Kody odpowiedzi HTTP

| Kod | Znaczenie |
|-----|-----------|
| 200 | Sukces - dane przetworzone poprawnie |
| 400 | Bad Request - błąd walidacji lub nieprawidłowe dane |
| 500 | Internal Server Error - nieoczekiwany błąd serwera |
