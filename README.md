# DOKUMENT ZLECENIOWY - INTEGRACJA SUBIEKT GT Z SYSTEMEM WMS DTW

## INFORMACJE O DOKUMENCIE

**Typ dokumentu:** Specyfikacja techniczna i funkcjonalna projektu

**Wersja:** 1.0

**Data:** 2025-11-14

**Cel:** Zapytanie ofertowe na wykonanie integracji systemów magazynowych

**Przeznaczenie:** Dla firm zewnętrznych w celu wyceny projektu


---

## STRESZCZENIE WYKONAWCZE

Poszukujemy wykonawcy do zbudowania systemu automatycznej integracji pomiędzy naszym systemem ERP **Subiekt GT** a zewnętrznym systemem magazynowym **WMS DTW**. Integracja ma działać w środowisku **Windows Server** i automatycznie synchronizować dokumenty magazynowe w czasie rzeczywistym poprzez REST API.

**Szacowany czas realizacji:** do uzgodnienia, jak najszybciej

**Środowisko:** Windows Server 2022

**Technologia:** REST API, COM Interface

**Tryb pracy:** Ciągły. Automatyczna synchronizacja uruchamiana w momencie oflagowania dokumentu w Subiekt GT.


---

## 0. KWESTIE DO UZGODNIENIA Z WYKONAWCĄ

### 0.1. Pytania do wykonawcy
1. **Czas:**
   - Jaki realny czas realizacji projektu?
   - Kiedy możecie rozpocząć pracę?

2. **Wycena:**
   - Ile wynosi całkowity koszt realizacji projektu (netto)?
   - Jakie są formy płatności?
   - Czy są dodatkowe koszty (np. wsparcie, szkolenia)?

3. **Wsparcie:**
   - Jaki zakres wsparcia powdrożeniowego oferujecie?
   - Czy oferujecie gwarancję na kod?
   - Jak wygląda proces zgłaszania błędów?

4. **Własność kodu:**
   - Czy kod źródłowy będzie przekazany w pełni?
   - Czy będziemy mogli go modyfikować?
   - Czy są jakieś ograniczenia licencyjne?

---

## 1. CEL PROJEKTU

### 1.1. Cel biznesowy

Automatyzacja przepływu dokumentów magazynowych pomiędzy systemem sprzedażowym Subiekt GT a zewnętrznym centrum logistycznym obsługiwanym przez system WMS DTW. Eliminacja ręcznego wprowadzania danych i ich przesyłania drogą mailową, redukcja błędów oraz skrócenie czasu realizacji zamówień.

### 1.2. Zakres funkcjonalny

System ma:
- Automatycznie pobierać dokumenty z Subiekt GT (WZ, FZ, RW) oznaczone odpowiednią flagą
- Przekształcać dane do formatu wymaganego przez DTW API
- Wysyłać dokumenty do systemu DTW przez REST API
- Automatycznie generować i przesyłać pliki PDF dokumentów WZ
- Automatycznie dodawać brakujące produkty do systemu DTW
- Flagować przetworzone dokumenty w Subiekt GT
- Logować wszystkie operacje i błędy
- Działać w trybie automatycznym 

---

## 2. SYSTEMY DO INTEGRACJI

### 2.1. System źródłowy: Subiekt GT

**Producent:** InsERT S.A.
**Wersja:** Subiekt GT (z licencją Sfera)
**Baza danych:** Microsoft SQL Server
**Interfejs dostępu:** COM (Component Object Model) - biblioteka Sfera

#### Charakterystyka systemu:
- System ERP dla małych i średnich firm
- Zarządzanie sprzedażą, magazynem, finansami
- Instalacja lokalna na serwerze Windows
- Dostęp programistyczny przez interfejs COM (ActiveX)
- Wymaga autoryzacji operatora Subiekt GT

#### Dane dostępne z systemu:
- Dokumenty magazynowe: WZ (Wydanie Zewnętrzne), FZ (Faktura Zakupu), RW (Rozchód Wewnętrzny)
- Dane kontrahentów: nazwa, adres, telefon, email
- Dane adresów dostawy przypisane do WZ
- Kartoteka towarowa: symbol, nazwa, EAN, waga, wymiary
- Flagi własne do oznaczania dokumentów WZ, RW, FZ

### 2.2. System docelowy: WMS DTW

**Typ:** System zarządzania magazynem (WMS)
**Dostęp:** REST API przez HTTPS
**Autoryzacja:** Statyczny klucz API (header `auth-key`)
**Format danych:** JSON (głównie), multipart/form-data (dla plików PDF)
**Dokumentacja API:** Dostarczona przez DTW

#### Endpointy API do implementacji:

| Endpoint | Metoda | Format | Przeznaczenie |
|----------|--------|--------|---------------|
| `/AddOrder` | POST | JSON | Dodawanie zamówień do WMS |
| `/AddReceipt` | POST | JSON | Dodawanie awizacji dostaw |
| `/UploadGoodsIssueNote` | POST | multipart/form-data | Przesyłanie PDF dokumentów WZ |
| `/UploadMasterData` | POST | JSON | Przesyłanie danych produktowych |

#### Wymagania DTW:
- Unikalne numery dokumentów (CustomerOrderCode, ReceiptNumber)
- Produkty muszą być najpierw dodane przez `/UploadMasterData`
- Pliki PDF max 3MB
- Format daty: YYYY-MM-DD
- Pełna walidacja danych po stronie API

---

## 3. ARCHITEKTURA ROZWIĄZANIA

### 3.1. Diagram przepływu danych

```
┌─────────────────────────────────────────────────────────────────┐
│                   SUBIEKT GT (SQL Server)                       │
│            Dokumenty: WZ, FZ, RW z flagami                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ COM Interface (Sfera)
                            │ - Pobieranie dokumentów z flagą
                            │ - Wczytywanie danych dokumentów
                            │ - Generowanie PDF
                            │ - Flagowanie przetworzonych
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                  WARSTWA INTEGRACYJNA                        │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Moduł pobrania  │  │ Moduł mapowania  │  │ Moduł wysyłki │ │
│  │ z Subiekt GT    │→ │ danych           │→ │ do DTW API    │ │
│  └─────────────────┘  └──────────────────┘  └───────────────┘ │
│                                                                 │
│  ┌─────────────────┐  ┌──────────────────┐  ┌───────────────┐ │
│  │ Logger          │  │ Obsługa błędów   │  │ Retry logic   │ │
│  └─────────────────┘  └──────────────────┘  └───────────────┘ │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ HTTPS REST API
                            │ - Authorization: auth-key
                            │ - Content-Type: application/json
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DTW REST API                               │
│  /AddOrder  |  /AddReceipt  |  /UploadGoodsIssueNote  |  ...   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SYSTEM WMS DTW                                │
│                      (Magazyn)             │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2. Komponenty systemu

System integracyjny powinien składać się z następujących modułów:

#### A) Moduł połączenia z Subiekt GT
- Nawiązywanie połączenia COM z biblioteką Sfera
- Autoryzacja operatorem Subiekt GT
- Ustawianie kontekstu magazynu
- Obsługa błędów połączenia

#### B) Moduł pobierania dokumentów
- Wyszukiwanie dokumentów WZ/FZ/RW z określoną flagą na wszystkich magazynach
- Wczytywanie pełnych danych dokumentu
- Pobieranie danych kontrahenta/odbiorcy (adres dostawy)
- Pobieranie pozycji dokumentu (towary)
- Generowanie plików PDF dla WZ

#### C) Moduł mapowania danych
- Transformacja struktury danych Subiekt GT → DTW
- Mapowanie WZ → AddOrder (JSON)
- Mapowanie FZ → AddReceipt (JSON)
- Mapowanie RW → AddOrder (JSON)
- Mapowanie Produkty → UploadMasterData (JSON)
- Parsowanie adresów z pola uwag (dla RW)
- Walidacja wymaganych pól

#### D) Klient REST API
- Implementacja 4 endpointów DTW
- Obsługa autoryzacji (auth-key)
- Wysyłka JSON przez POST
- Wysyłka plików PDF (multipart/form-data)
- Parsowanie odpowiedzi DTW
- Obsługa timeout (konfigurowalny)

#### E) Moduł obsługi błędów
- Retry logic dla błędów sieciowych (HTTP 5xx)
- Walidacja danych przed wysyłką
- Detekcja duplikatów (CustomerOrderCode już istnieje)
- Detekcja brakujących produktów w DTW
- Automatyczne dodawanie brakujących produktów

#### F) System logowania
- 4 poziomy logów: DEBUG, INFO, WARNING, ERROR
- Logowanie requestów HTTP (URL, headers, body)
- Logowanie responsów HTTP (status, body, czas)
- Logowanie wyjątków z pełnym stack trace
- Ukrywanie wrażliwych danych (auth-key)
- Automatyczna rotacja plików logów

#### G) Moduł flagowania dokumentów
- Ustawianie odpowiednich flag w Subiekt GT po pomyślnym imporcie
- Obsługa błędów ustawiania flag
- Logowanie operacji flagowania

---

## 4. SZCZEGÓŁOWY OPIS FUNKCJONALNOŚCI

### 4.1. Proces synchronizacji dokumentów WZ (Wydanie Zewnętrzne)

**Użytkownik Subiekt GT:**
1. Tworzy dokument WZ w Subiekt GT
2. Uzupełnia dane odbiorcy (wybiera z kartoteki kontrahenta i/lub dodaje nowy adres dostawy)
3. Dodaje pozycje towarowe do dokumentu
4. Zapisuje dokument
5. **Zaznacza flagę "WZ gotowa do importu do DTW"** dla utworzonego dokumentu WZ


**System integracyjny (automatycznie):**
1. Pobiera listę dokumentów WZ z flagą "do importu" dla wszystkich magazynów
2. Dla każdego dokumentu WZ:
   - Wczytuje pełne dane dokumentu przez COM
   - Pobiera dane odbiorcy i adres dostawy
   - Pobiera listę towarów z dokumentu
   - **Generuje plik PDF** dokumentu WZ
   - Sprawdza czy wszystkie produkty istnieją w bazie danych DTW
   - Jeśli produkt nie istnieje: dodaje go do DTW przez `/UploadMasterData`
   - Mapuje dane na format DTW AddOrder (JSON)
   - Wysyła zamówienie do DTW przez `/AddOrder`
   - Wysyła plik PDF do DTW przez `/UploadGoodsIssueNote`
   - Ustawia flagę "zaimportowane do DTW" dla dokumentu WZ
3. Loguje wynik operacji (sukces/błąd)
4. Przechodzi do kolejnego dokumentu

**Wynik:**
- Dokument WZ został przesłany do systemu DTW
- DTW rozpoczyna proces kompletacji i wysyłki
- Dokument w Subiekt GT jest oflagowany jako "zaimportowany"

### 4.2. Proces synchronizacji dokumentów FZ (Faktura Zakupu - Awizacja)

**Użytkownik Subiekt GT:**
1. Tworzy dokument FZ (awizację dostawy)
2. Wybiera dostawcę
3. Dodaje pozycje towarowe z ilościami
4. Zapisuje dokument
5. **Zaznacza flagę "Awizacja DTW"**


**System integracyjny (automatycznie):**
1. Pobiera listę dokumentów FZ z flagą "Awizacja DTW"
2. Dla każdego dokumentu FZ:
   - Wczytuje dane dokumentu (numer, data dostawy, dostawca, numer oryginału)
   - Pobiera pozycje towarowe
   - Sprawdza czy produkty istnieją w DTW (dodaje jeśli nie)
   - Mapuje dane na format DTW AddReceipt (JSON)
   - Wysyła awizację do DTW przez `/AddReceipt`
   - Ustawia flagę "zaimportowane do DTW" dla dokumentu FZ
3. Loguje wynik operacji

**Wynik:**
- Awizacja została przesłana do DTW
- DTW przygotowuje się na przyjęcie dostawy
- Dokument w Subiekt GT jest oflagowany

### 4.3. Proces synchronizacji dokumentów RW (Rozchód Wewnętrzny)

**Użytkownik Subiekt GT:**
1. Tworzy dokument RW
2. Dodaje pozycje towarowe
3. **W polu "Uwagi" wpisuje adres dostawy w formacie:**
   ```
   Jan Kowalski
   Główna 10/5
   00-001 Warszawa
   tel. 123456789
   jan@kowalski.com
   ```
4. Zapisuje dokument
5. **Zaznacza flagę "RW gotowa do importu"**

**System integracyjny (automatycznie):**
1. Pobiera listę dokumentów RW z flagą "do importu"
2. Dla każdego dokumentu RW:
   - Wczytuje dane dokumentu
   - **Parsuje adres dostawy z pola "Uwagi"** (regex/split na nazwe odbiorcy, ulicę z numerem domu, kod pocztowy, miasto)
   - Pobiera pozycje towarowe
   - Sprawdza produkty w DTW (dodaje jeśli nie istnieje jakaś pozycja)
   - Mapuje dane na format DTW AddOrder (JSON)
   - Wysyła jako zamówienie do DTW przez `/AddOrder`
   - Ustawia flagę "zaimportowane do DTW"
3. Loguje wynik operacji

**Wynik:**
- Rozchód wewnętrzny został przesłany jako zamówienie do DTW
- Dokument w Subiekt GT jest oflagowany

### 4.4. Automatyczne dodawanie produktów do DTW

**Scenariusz:**
System wykrywa że dokument zawiera produkt, którego nie ma w DTW.

**Proces:**
1. DTW zwraca błąd HTTP 400: "Product code was found which does not exist in WMS!"
2. System parsuje listę brakujących produktów z odpowiedzi
3. Dla każdego brakującego produktu:
   - Pobiera dane z kartoteki towarowej Subiekt GT (symbol, nazwa, EAN, waga, wymiary)
   - Mapuje na format DTW UploadMasterData
   - Wysyła do DTW przez `/UploadMasterData`
4. Po dodaniu wszystkich produktów - ponawia wysyłkę dokumentu

**Format danych produktu do DTW:**
```json
{
  "MasterDataItemsList": [
    {
      "ProductCode": "PROD-001",
      "ItemDescription": "Nazwa produktu",
      "BarCode": "5901234567890",
      "Weight": 0.5,
      "Height": 0.2,
      "Length": 0.3,
      "Liters": 1,
      "ProdGroup": "Kategoria A",
      "ShelfLife": 365,
      "MinimumShelfLife": 30,
      "CapCountAfterPicking": 100,
      "QuantityPerCarton": 12
    }
  ]
}
```

---

## 5. MAPOWANIE DANYCH

### 5.1. Mapowanie WZ → AddOrder (DTW)

**Dane źródłowe (Subiekt GT WZ):**
```php
[
    'numer' => 'WZ 1/2025',
    'data_utworzenia' => '2025-01-15',
    'uwagi' => 'Uwagi do zamówienia',
    'adres_dostawy' => [
        'nazwa' => 'Jan Kowalski',
        'ulica' => 'Główna',
        'nr_domu' => '10',
        'nr_lokalu' => '5',
        'kod_pocztowy' => '00-001',
        'miejscowosc' => 'Warszawa',
        'telefon' => '123456789'
    ],
    'email' => 'jan@kowalski.com',
    'towary' => [
        ['symbol' => 'PROD-001', 'ilosc' => 10.0],
        // ...
    ]
]
```

**Dane docelowe (DTW AddOrder JSON):**
```json
{
  "OrderHeader": {
    "CustomerOrderCode": "WZ 1/2025",
    "AgencyCode": "UPS",
    "PickRemarks": "",
    "PackRemarks": "",
    "ExpShipDate": "2025-01-15",
    "IsB2B": false,
    "Customer": {
      "FullName": "Jan Kowalski",
      "Address": "Główna 10/5",
      "City": "Warszawa",
      "ZipCode": "00-001",
      "CountryCode": "PL",
      "Phone": "123456789",
      "Email": "jan@kowalski.com"
    }
  },
  "Lines": [
    {
      "ItemCode": "PROD-001",
      "Quantity": 10
    }
  ]
}
```

**Reguły mapowania:**
- `CustomerOrderCode` ← numer dokumentu WZ (musi być unikatowy!)
- `ExpShipDate` ← data_utworzenia dokumentu
- `FullName` ← nazwa z adresu dostawy
- `Address` ← konkatenacja: ulica + " " + nr_domu + "/" + nr_lokalu
- `ItemCode` ← symbol towaru
- `Quantity` ← ilość (zaokrąglona do INT)

### 5.2. Mapowanie FZ → AddReceipt (DTW)

**Dane źródłowe (Subiekt GT FZ):**
```php
[
    'numer' => 'FZ 1/2025',
    'numer_oryginalu' => 'FV/2025/001',
    'data_utworzenia' => '2025-01-15',
    'uwagi' => 'Dostawa z Polski',
    'kontrahent_nazwa' => 'Firma Dostawcza Sp. z o.o.',
    'towary' => [
        ['symbol' => 'PROD-001', 'nazwa' => 'Produkt testowy', 'ilosc' => 100.0],
        // ...
    ]
]
```

**Dane docelowe (DTW AddReceipt JSON):**
```json
{
  "ReceiptHeader": {
    "ReceiptNumber": "FZ 1/2025",
    "Supplier": "Firma Dostawcza Sp. z o.o.",
    "ExpectedDate": "2025-01-15",
    "RefDocNo": "FV/2025/001",
    "Remarks": "Dostawa z Polski",
    "CustomerOrderCode": ""
  },
  "ReceiptLines": [
    {
      "ItemCode": "PROD-001",
      "itemDescr": "Produkt testowy",
      "Quantity": 100,
      "DetailRemarks": "",
      "SSCC": ""
    }
  ]
}
```

### 5.3. Mapowanie RW → AddOrder (DTW)

Podobne do WZ, ale:
- Adres dostawy jest parsowany z pola "Uwagi"
- Format uwag (wymagany):
  ```
  [Imię Nazwisko]
  [Ulica] [Nr domu]/[Nr lokalu]
  [Kod pocztowy] [Miejscowość]
  tel. [Telefon]
  [Email]
  ```

**Przykład parsowania:**
```
Wejście (pole uwagi):
Jan Kowalski
Główna 10/5
00-001 Warszawa
tel. 123456789
jan@example.com

Wyjście (struktura adresu):
FullName: "Jan Kowalski"
Address: "Główna 10/5"
City: "Warszawa"
ZipCode: "00-001"
Phone: "123456789"
Email: "jan@example.com"
```

---

## 6. SPECYFIKACJA TECHNICZNA

### 6.1. Wymagania systemowe

**Serwer:**
- System operacyjny: Windows Server 2022 lub nowszy (Windows 10/11 dla testów)
- Procesor: 4+ rdzenie
- RAM: 4 GB minimum (8 GB zalecane)
- HDD: 10 GB wolnego miejsca
- Połączenie internetowe: stabilne, min. 300 Mbps

**Oprogramowanie:**
- Dowolny stack technologiczny
- Microsoft SQL Server (dla Subiekt GT)
- Subiekt GT z licencją Sfera (interfejs COM)


---

## 7. WYMAGANIA FUNKCJONALNE - PERSPEKTYWA UŻYTKOWNIKA

### 7.1. Dla użytkownika Subiekt GT (pracownik magazynu/biura)

**Scenariusz 1: Wysyłka zamówienia (WZ)**

1. Pracownik tworzy dokument WZ w Subiekt GT
2. Uzupełnia dane odbiorcy:
   - Wybiera kontrahenta (dane zaciągają się automatycznie)
   - Lub wybiera adres dostawy z listy
   - Lub wpisuje nowy adres ręcznie
3. Dodaje towary do dokumentu (wybierając z kartoteki)
4. W polu "Email" wpisuje email klienta (jeśli nie jest w bazie)
5. **Kluczowy krok:** Zaznacza checkbox/flagę **"WZ gotowa do importu do DTW"**
6. Zapisuje dokument
7. **System automatycznie (w ciągu 1 minuty):**
   - Pobiera dokument
   - Wysyła do DTW
   - Ustawia flagę "zaimportowane do W2S"
8. Pracownik widzi że flaga zmieniła się → wie że dokument został przesłany

**Scenariusz 2: Awizacja dostawy (FZ)**

1. Pracownik tworzy dokument FZ
2. Wybiera dostawcę
3. Dodaje towary które mają przyjść
4. **Zaznacza flagę "Awizacja DTW"**
5. Zapisuje
6. System automatycznie wysyła awizację do DTW
7. Flaga zmienia się na "zaimportowane do W2S"

**Scenariusz 3: Rozchód wewnętrzny (RW)**

1. Pracownik tworzy dokument RW
2. Dodaje towary
3. **W polu "Uwagi" wpisuje adres dostawy w formacie:**
   ```
   Jan Kowalski
   Główna 10/5
   00-001 Warszawa
   tel. 123456789
   jan@example.com
   ```
4. **Zaznacza flagę "RW gotowa do importu"**
5. Zapisuje
6. System automatycznie wysyła jako zamówienie do DTW
7. Flaga zmienia się na "zaimportowane do W2S"

### 7.2. Dla administratora systemu

**Panel konfiguracyjny:**
- Ustawienie URL i klucza API DTW
- Konfiguracja flag w Subiekt GT (wskazanie spośród istniejących po ID lub wybór z listy)
- Konfiguracja wzorca wydruku WZ
- Ustawienie magazynów do synchronizacji
- Ustawienie częstotliwości sprawdzania dokumentów (co minutę, co 5 minut, etc.)
- Włączenie/wyłączenie trybu testowego

**Monitorowanie:**
- Sprawdzanie statystyk w logach
- Analiza błędów
- Monitoring działania integracji


---

## 8. PODSUMOWANIE

Projekt polega na stworzeniu **kompletnego systemu automatycznej integracji** pomiędzy **Subiekt GT** (system ERP) a **DTW WMS** (system magazynowy) działającego w środowisku **Windows Server**.

**Kluczowe założenia:**
- Automatyczna synchronizacja dokumentów WZ, FZ, RW
- Komunikacja przez REST API (4 endpointy DTW)
- Interfejs COM do Subiekt GT (biblioteka Sfera)
- Automatyczne generowanie i wysyłka PDF
- Automatyczne dodawanie produktów do DTW
- Flagowanie dokumentów w Subiekt GT
- Pełne logowanie i obsługa błędów
- Tryb automatyczny - ciągły

**Dla użytkownika:**
- Prosty proces: zaznacz flagę w dokumencie → system automatycznie wyśle do DTW
- Brak ręcznego wprowadzania danych
- Szybka realizacja zamówień
- Niezawodność

**Dla administratora:**
- Łatwa konfiguracja (GUI konfiguracyjne lub pliki konfiguracyjne)
- Pełne logi do monitorowania
- Dokumentacja techniczna


