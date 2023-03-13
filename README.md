# Визуализация в Redash


Изначально посмотрим на общую сумму продаж в разрезе каждой страны по месяцам. Видим общую динамику продаж и лидирующих стран. 
Великобритания на данном графике не отображена, т.к. у нас британский интернет-магазин и основные продажи в Великобритании.

```sql
    SELECT
        Country,
        toStartOfMonth(InvoiceDate) AS Month,
        ROUND(SUM(Quantity * UnitPrice)) AS Sum_total_price_by_month
    FROM default.retail
    WHERE Quantity > 0 AND Country IN ('Netherlands','EIRE','Australia','Germany','France')
    GROUP BY Country, Month
    ORDER BY Sum_total_price_by_month DESC
```	
	
Теперь посмотрим на соотношение суммы продаж в Великобритании (внутренний рынок) и другие страны (внешний рынок) также в разрезе по месяцам. 
С августа видим рост продаж как на внутреннем, так и на внешнем рынке. Скорее всего была проведена успешная рекламная компания.

```sql
    SELECT 
        multiIf(Country='United Kingdom',Country,'Other') as All_countries,
        toStartOfMonth(InvoiceDate) AS Month,
        ROUND(SUM(Quantity * UnitPrice)) AS Sum_total_price
    FROM default.retail
    WHERE Quantity > 0
    GROUP BY All_countries, Month
    ORDER BY Month, All_countries
```	
	
ТОП-5 стран

```sql
	with stat1 as (
    SELECT 
        Country,
        ROUND(SUM(Quantity * UnitPrice)) AS Sum_by_country
    FROM default.retail
    WHERE Country != 'United Kingdom'
    GROUP BY Country
    ORDER BY Sum_by_country DESC
    ),

stat2 as (
    SELECT 
        Country,
        Sum_by_country,
        SUM(Sum_by_country) OVER() AS Total_sum
    FROM stat1),

stat3 as (
    SELECT 
        Sum_by_country,
        Total_sum,
        ROUND(Sum_by_country / Total_sum * 100,1) as Country_perc,
        multiIf(Country_perc < 5, 'Other',Country) AS Countries
    FROM stat2)

SELECT 
    Countries AS Country,
    SUM(Sum_by_country) as Country_amount
FROM stat3
GROUP BY Countries;
```

Теперь посмотрим на покупателей, нас интересует приток новых клиентов по месяцам. 
Мы смотрим на клиентов, которые совершили свою первую покупку. 
Обратим внимание, что также наблюдаем с сентября 2011 результаты успешной рекламной компании, т.к. видим приток новых клиентов.

```sql
    with uniq1 as 
    (SELECT distinct 
        CustomerID,
        toStartOfMonth(InvoiceDate) as Month,
        multiIf(Country='United Kingdom',Country,'Other') as All_countries
    FROM default.retail
    ORDER BY Month, CustomerID, All_countries
    ),

    uniq2 as 
    (SELECT 
        CustomerID,
        Month,
        All_countries,
        dense_rank() OVER (PARTITION BY CustomerID order by Month, All_countries) RNK
    FROM uniq1)

    SELECT 
        Month,
        All_countries,
        countIf(CustomerID, RNK = 1) as Uniq_count_customer
    FROM uniq2
    GROUP BY Month, All_countries
    HAVING Uniq_count_customer!=0
    ORDER BY Month, All_countries
```

В разрезе по дням видим, что 09.12.2011 у нас аномально высокая сумма продаж. Сгруппировав продажи по странам, получаем, что в этот день основное количество продаж было в Великобритании.
Также видим, что сумма продаж коррелируется с количеством проданных товаров. Чем больше проданных товаров, тем выше прибыль. 
При анализе обнаружились товары с нулевой ценой. В частности, есть заказ (InvoiceNo 578841) с большим количеством (Quantity) и нулевой ценой. 
Такие заказы стоит обсудить отдельно с отделом продаж (чтобы выявить аномалии, например ошибки в базе).

```sql
    SELECT
        toStartOfDay(InvoiceDate) AS Day,
        ROUND(SUM(Quantity * UnitPrice)) AS Sum_revenue_by_day,
        SUM(Quantity) AS Sum_quantity
    FROM default.retail
    WHERE Quantity > 0
    GROUP BY Day
    ORDER BY Sum_revenue_by_day 
```	
	
ТОП-10 продуктов по сумме продаж

```sql
    SELECT 
        StockCode,
        Description,
        ROUND(SUM(Quantity * UnitPrice)) AS Total_sum
    FROM default.retail
    GROUP BY StockCode, Description
    ORDER BY Total_sum DESC
    LIMIT 10
```

ТОП-10 возвратов (по сумме) смотрим по клиенту и по стране клиента.
Обратим внимание на клиента из Великобритании с CustomerID = 16446. У него наибольшее количество возвратов. 
Но он также попадает и в ТОП-10 клиентов, сделавших самые дорогие покупки. Этот клиент купил большое количество товара, но потом сделал его возврат. 
Аналогичная ситуация и с другим клиентом CustomerID = 12346. 
Пожалуй, это не самые интересные нам клиенты и стоит повременить с персональной скидкой для них.

```sql
    SELECT 
        Country || ' ' || toString(CustomerID) AS Client,
        ROUND(SUM(Quantity*UnitPrice)) AS Sum_return
    FROM default.retail
    WHERE Quantity < 0
    GROUP BY Client
    ORDER BY Sum_return
    LIMIT 10
```

Посмотрим на ТОП-10 клиентов по сумме покупок за год. А также посмотрим, как эти клиенты совершали покупки в течение года, был ли рост суммы покупок. 
Можем повысить их лояльность, предложив дополнительные скидки.
Также посмотрев, с каких стран эти клиенты, видим, что максимальные суммы покупок не только среди клиентов из Великобритании, 
но и других стран (в частности Нидерланды, Ирландия и Австралия).

```sql
    SELECT
        Country || ' ' || toString(CustomerID) AS Client,
        ROUND(SUM(Quantity * UnitPrice)) AS Sum_revenue
    FROM default.retail
    WHERE Quantity > 0
    GROUP BY Client
    ORDER BY Sum_revenue DESC
    LIMIT 10
```	
	
Считаем AOU - среднюю сумму заказа (средний чек) по месяцам ТОП-5 стран, а также Великобритании. Видим, что средний чек выше в Австралии и Нидерландах, тогда как в Великобритании он невысокий.

```sql
    SELECT
        Country,
        toStartOfMonth(InvoiceDate) AS Month,
        ROUND(AVG(Quantity * UnitPrice)) AS AOU
    FROM default.retail
    WHERE Quantity > 0 AND Country IN ('United Kingdom','Netherlands','EIRE','Australia','Germany','France')
    GROUP BY Month, Country
    ORDER BY AOU DESC
```

Посмотрев на количество заказов по месяцам для ТОП-5 стран и Великобритании мы наблюдаем, что количество заказов в Великобритании значительно выше, чем в других странах, хотя средний чек ниже.

```sql
    SELECT
        Country,
        toStartOfMonth(InvoiceDate) AS Month,
        ROUND(COUNT(InvoiceNo)) AS Count_orders
    FROM default.retail
    WHERE Quantity > 0 AND Country IN ('United Kingdom','Netherlands','EIRE','Australia','Germany','France')
    GROUP BY Month, Country
    ORDER BY Count_orders DESC 
 ```
    
<p align="center">

  <img width="860" height="650" src="https://github.com/Juldid/Redash/blob/main/Redash%201.JPG">

</p>

<p align="center">

  <img width="860" height="650" src="https://github.com/Juldid/Redash/blob/main/Redash%202.JPG">

</p>

<p align="center">

  <img width="660" height="650" src="https://github.com/Juldid/Redash/blob/main/Redash%203.JPG">

</p>
