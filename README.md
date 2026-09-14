# ubuntu-tech-sales-portfolio

 Retail (fictional)

Currency: South African Rand (ZAR/R)

Period: 2024-2025

Main fact table: sales_orders.csv (~12,035 rows including intentional duplicates)

TABLES
1. customers
   customer_id PK
   customer_name
   email
   phone
   city
   province
   segment
   signup_date

2. products
   product_id PK
   product_name
   category
   cost_price
   list_price
   active

3. stores
   store_id PK
   store_name
   city
   province

4. sales_orders
   order_id PK (but contains intentional duplicate rows)
   order_date
   customer_id FK -> customers.customer_id
   product_id FK -> products.product_id
   store_id FK -> stores.store_id
   quantity
   unit_price_zar
   discount
   subtotal_zar
   vat_zar
   total_zar
   payment_method
   sales_channel
   order_status

INTENTIONAL DATA-QUALITY ISSUES
- Duplicate sales rows
- Missing customer/payment/channel/price values
- Mixed date formats
- Numeric values stored as text in some rows
- Currency symbols embedded in some prices
- Leading/trailing spaces
- Inconsistent capitalization
- Inconsistent payment/channel/status labels
- Inconsistent province capitalization
- A few malformed phone numbers
- Missing product cost prices
- Percentage-formatted discounts mixed with decimal discounts
- Cancelled/returned/pending orders mixed with completed sales
