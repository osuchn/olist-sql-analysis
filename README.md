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

