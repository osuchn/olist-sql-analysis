# Olist E-Commerce: analiza w SQL (SQL Server)

Projekt SQL na brazylijskim zbiorze danych e-commerce Olist, wykonany w SQL Server Management Studio (SSMS).

## Dane

[Brazilian E-Commerce Public Dataset by Olist](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) (Kaggle). Pliki z danymi nie są dołączone do repozytorium.

## Krok 1: Import

Pliki CSV zaimportowano do bazy `OlistEcommerce` za pomocą kreatora Import Flat File w SSMS.

Sprawdzenie liczby wierszy po imporcie:

```sql
SELECT t.name AS tabela, SUM(p.rows) AS liczba_wierszy
FROM sys.tables t
JOIN sys.partitions p ON t.object_id = p.object_id
WHERE p.index_id IN (0, 1)
GROUP BY t.name
ORDER BY t.name;
```

Wynik: liczby wierszy zgadzają się z opisem zbioru (np. orders 99 441, order_items 112 650).
<img width="448" height="232" alt="image" src="https://github.com/user-attachments/assets/62c60b1b-3ea2-4f4d-9e11-183fc9742fbd" />

## Krok 2: Sprawdzenie typów kolumn

Po imporcie sprawdzono, jakie typy danych nadał kreator poszczególnym kolumnom:

```sql
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```
Poniżej przedstawiono fragment wyniku:

<img width="504" height="466" alt="image" src="https://github.com/user-attachments/assets/fcd096f3-a600-4934-9d75-262e47f818da" />

Pełny wynik obejmuje wszystkie kolumny w bazie (46 wierszy), dlatego do dokumentacji użyto zawężonego zapytania, które pokazuje tylko te kolumny, które po analizie pełnej listy powinny być liczbami, a zostały zaimportowane jako tekst:

```sql
SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE DATA_TYPE = 'nvarchar'
  AND COLUMN_NAME IN ('order_item_id', 'price', 'freight_value', 'payment_sequential',
                      'payment_installments', 'payment_value', 'review_score', 'product_photos_qty')
ORDER BY TABLE_NAME, ORDINAL_POSITION;
```

<img width="463" height="238" alt="image" src="https://github.com/user-attachments/assets/a146616d-245e-4303-a284-f8651d87e10e" />


### Wnioski

- Daty zostały rozpoznane poprawnie (`datetime2`), tak samo kody pocztowe i wymiary produktów (`int`).
- Część kolumn liczbowych została zaimportowana jako tekst (`nvarchar`) i wymaga zamiany na typy liczbowe:
  - `order_items`: `order_item_id`, `price`, `freight_value`
  - `order_payments`: `payment_sequential`, `payment_installments`, `payment_value`
  - `order_reviews`: `review_score`
  - `products`: `product_photos_qty`
- W tabeli `product_category_name_translation` kolumny nazywają się `column1` i `column2`, co oznacza, że kreator nie rozpoznał nagłówka pliku.

### Dlaczego część kolumn jest tekstem

Kolumny `price`, `freight_value` i `payment_value` zawierają liczby z kropką jako separatorem dziesiętnym (np. `58.90`). Import tych kolumn jako typ liczbowy kończył się błędem `Input string was not in a correct format`, prawdopodobnie przez polskie ustawienia regionalne Windowsa, w których separatorem jest przecinek. Dlatego kolumny z liczbami zaimportowano jako tekst (`nvarchar`). Zamiana na typy liczbowe zostanie wykonana w SQL w następnym kroku.

## Krok 3: Naprawa tabeli `product_category_name_translation`

W wyniku kroku 2 okazało się, że kolumny tej tabeli nazywają się `column1` i `column2`. Kreator importu nie rozpoznał nagłówka pliku, więc trafił on do tabeli jako pierwszy wiersz danych. Potwierdzenie:

```sql
SELECT TOP 5 * FROM product_category_name_translation;
```

<img width="621" height="159" alt="image" src="https://github.com/user-attachments/assets/151afd45-067b-4236-95fd-c4760887cf88" />


Usunięto wiersz z nagłówkiem:

```sql
DELETE FROM product_category_name_translation
WHERE column1 = 'product_category_name';
```

Sprawdzenie liczby wierszy (oczekiwane 71 kategorii):

```sql
SELECT COUNT(*) FROM product_category_name_translation;
```

Wynik: 71 wierszy.

Następnie przemianowano kolumny na właściwe nazwy:

```sql
EXEC sp_rename 'product_category_name_translation.column1', 'product_category_name', 'COLUMN';
EXEC sp_rename 'product_category_name_translation.column2', 'product_category_name_english', 'COLUMN';
```

Po zmianie tabela zawiera kolumny `product_category_name` (nazwa portugalska) i `product_category_name_english` (nazwa angielska).


## Krok 4: Konwersja typów w tabeli `order_items`

Kolumny `order_item_id`, `price` i `freight_value` zostały zaimportowane jako tekst (`nvarchar`). Przed zmianą typu sprawdzono, czy wszystkie wartości da się przekonwertować. Funkcja `TRY_CAST` zwraca `NULL`, jeśli konwersja się nie powiedzie, a `COUNT` pomija wartości `NULL`, więc różnica między liczbą wszystkich wierszy a wynikiem `COUNT(TRY_CAST(...))` pokazuje liczbę wartości, które się nie konwertują:

```sql
SELECT
  COUNT(*) AS wszystkie,
  COUNT(TRY_CAST(order_item_id AS INT)) AS poprawne_item_id,
  COUNT(TRY_CAST(price AS DECIMAL(10,2))) AS poprawne_price,
  COUNT(TRY_CAST(freight_value AS DECIMAL(10,2))) AS poprawne_freight
FROM order_items;
```

<img width="574" height="64" alt="image" src="https://github.com/user-attachments/assets/af1b3a32-62d0-4208-b47c-6f6ff89b94ef" />


Wszystkie wartości dały się przekonwertować, więc zmieniono typy kolumn:

```sql
ALTER TABLE order_items ALTER COLUMN order_item_id INT;
ALTER TABLE order_items ALTER COLUMN price DECIMAL(10,2);
ALTER TABLE order_items ALTER COLUMN freight_value DECIMAL(10,2);
```

Sprawdzenie typów po zmianie:

```sql
SELECT COLUMN_NAME, DATA_TYPE
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'order_items';
```
<img width="330" height="213" alt="image" src="https://github.com/user-attachments/assets/bfbaffdf-03be-4633-bb39-4d780126fe8d" />
