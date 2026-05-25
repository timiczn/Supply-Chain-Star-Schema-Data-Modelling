# Supply-Chain-Star-Schema-Data-Modelling
A dimensional data model built on top of a raw global e-commerce supply chain dataset, covering 180,519 order transactions across 5 international markets. The project transforms a single wide flat file into a structured Star Schema using Power Query inside Power BI, with the goal of making the data reliably queryable for supply chain analytics.
# Supply Chain Star Schema: Data Modelling

### Project Overview

The supply chain analytics team was working with a single flat file containing 53 columns and 180,519 rows. Customer information, product details, order transactions, delivery performance, and financial metrics were all sitting in one table with no structure. Running any meaningful analysis on it meant wading through redundant data, ambiguous columns, and no clear relationships between entities.

This project breaks that flat file apart into a properly normalised Star Schema: one fact table and six dimension tables: and validates that the model can answer the business questions the team actually needs answered.

<img width="1056" height="500" alt="Screenshot 2026-05-06 072932" src="https://github.com/user-attachments/assets/507ae3b8-7cc6-461f-a6da-44d3195ac0c1" />


### Business Questions the Model Answers

The model was designed and validated against six core business questions:

1. Which markets and regions generate the most profit?
2. Which shipping modes have the highest late delivery rates?
3. Which customer segments are most valuable by sales and profit?
4. Which product categories drive the most revenue?
5. How does delivery performance vary by market and shipping mode?
6. What is the trend of orders and profit over time?

All six were validated with live visuals after the model was built. Results are in the Validation section.

### Data Structure

The dataset is organised as a Star Schema with one central fact table and six supporting dimension tables. All tables were built in Power Query as references from a staging table, keeping the raw data untouched throughout the transformation process.

### Fact_OrderItems
The core transaction table with 180,519 rows. The grain is one order line item. Each row carries foreign keys linking to all six dimensions and the following measures: Sales, Order Item Total, Order Profit Per Order, Benefit per Order, Sales per Customer, Order Item Discount, Order Item Discount Rate, Order Item Profit Ratio, Order Item Quantity, Order Item Product Price, Days for Shipping (Real), Days for Shipment (Scheduled), and Late Delivery Risk. Order Date and Shipping Date are also held here as date fields supporting the role-playing date dimension.

### Dim_Customer
Contains demographic and segmentation information for each customer: Customer Name (merged from first and last name), Customer Segment, City, State, Country, Street, and Zipcode. 20,652 rows. Natural key: Customer Id.

### Dim_Product
Holds product, category, and department information in a single flat table. Since a Star Schema was chosen over Snowflake, the three-level hierarchy (Product, Category, Department) was collapsed here rather than split into separate tables. 118 rows. Natural key: Product Card Id.

### Dim_Geography
Stores the full location context for each order: Market, Order Region, Order Country, Order City, and Order State. No natural key existed, so a surrogate key (Geography_Id) was created using an index column after removing duplicates on the combination of all five columns. 3,772 rows.

### Dim_Shipping
Maps the unique combinations of Shipping Mode and Delivery Status. No natural key existed, so a surrogate key (Shipping_Id) was created the same way as Geography. 12 rows.

### Dim_Order
Holds order-level status information. One row per unique Order Id. The grain of the fact table is the line item, so this dimension sits one level above it, describing the order as a whole. 65,752 rows. Natural key: Order Id.

### Dim_Date
A calendar table built from scratch in Power Query using M code, covering 2015 to 2020. Columns: Date, Year, Month Number, Month Name, Month Abbrev, Quarter, Week Number. Month Abbrev is sorted by Month Number to ensure correct chronological ordering in all visuals. 2,192 rows. Natural key: Date.

The Date dimension connects to the fact table twice. The active relationship is on Order Date. The inactive relationship is on Shipping Date and is activated in DAX using USERELATIONSHIP() when analysis by shipping date is required. This is a role-playing dimension pattern.


### Key Design Decisions

### Grain of the Fact Table
The first question answered before any column was moved: what does one row represent? Order Id was not unique across rows. A single order could have multiple line items, each with its own Order Item Id. The grain is therefore the order line item, not the order. Everything else in the model flows from this.

### Star Schema over Snowflake
The product data has a three-level hierarchy across 118 products, 51 categories, and 11 departments. A Snowflake Schema would have split these into separate tables. Star Schema was chosen because the hierarchy is shallow, the dataset size does not justify the added complexity, and Power BI is optimised for flat dimensions. The tradeoff is some redundancy in Dim_Product, which is acceptable at this scale.

### Surrogate Keys for Geography and Shipping
Neither table had a natural key. Uniqueness in both cases is defined by the full combination of columns, not any single one. Surrogate keys were created using index columns in Power Query after removing duplicates on all relevant columns together.

### Role-Playing Date Dimension
Rather than creating two separate date tables or adding a DateKey integer column to the fact table, a single Dim_Date table was connected to the fact table twice: one active relationship on Order Date, and one inactive on Shipping Date. This is cleaner dimensional modelling and avoids unnecessary duplication.

### Order Item Product Price vs Product Price
These are two different fields. Order Item Product Price captures the price at the time of the transaction, which can differ from the listed price due to discounts or price changes over time. It belongs in the fact table. Product Price is the current listed product price and belongs in Dim_Product.

### Late Delivery Risk Measure
Late_delivery_risk is a binary column (1 = late, 0 = not late). Rather than summing it directly, a DAX measure was written to count only the rows where the value equals 1. This is more intentional and produces a cleaner, more meaningful metric.

```dax
Late Deliveries = COUNTROWS(FILTER(Fact_order_items, Fact_order_items[Late_delivery_risk] = 1))
```


### Data Quality Issues

| Issue | Detail | How It Was Handled |
|---|---|---|
| Product Description fully null | All 180,519 rows were null | Dropped from the model entirely |
| Customer Email and Password masked | Both columns contained only XXXXXXXXX | Dropped from the model entirely |
| Product Image is a URL | No analytical value | Dropped from the model |
| Order Zipcode ~86% null | 155,679 of 180,519 rows were null | Excluded and flagged |
| Date columns stored as text | Order Date and Shipping Date threw errors on standard type conversion | Fixed using Transform > Change Type > Using Locale (English US) |
| Category Id vs Category Name mismatch | 51 unique Category Ids but only 50 unique Category Names | Flagged as a data quality issue. One Category Id likely maps to two slightly different name variations in the source data |
| No natural keys for Geography and Shipping | No single column uniquely identified a geography or shipping record | Surrogate keys created using index columns after removing duplicates on full column combinations |


### Validation Tests

All six validation tests were run and passed after the model was built.

| Test | Business Question | Result |
|---|---|---|
| 01: Market Profitability | Total Profit by Market | Passed |
| 02: Delivery Performance | Late Deliveries by Shipping Mode | Passed |
| 03: Customer Segment Analysis | Total Sales by Customer Segment | Passed |
| 04: Product Category Revenue | Total Sales by Category (Top 5) | Passed |
| 05: Time Trend | Total Orders by Month across Years | Passed |
| 06: Cross Dimension Stress Test | Profit by Market and Shipping Mode simultaneously | Passed |

### Test 01: Market Profitability
<img width="225" height="207" alt="Screenshot 2026-05-06 070331" src="https://github.com/user-attachments/assets/f193217e-db7b-4bac-9722-42db2b0fefc7" />


### Test 02: Delivery Performance
<img width="641" height="311" alt="Screenshot 2026-05-06 061959" src="https://github.com/user-attachments/assets/7ec38b84-c46f-4fa0-b806-689e07cfc0fc" />


### Test 03: Customer Segment Analysis
<img width="642" height="306" alt="Screenshot 2026-05-06 062031" src="https://github.com/user-attachments/assets/c88452d1-8e78-4820-bd3f-13c814756def" />


### Test 04: Product Category Revenue
<img width="594" height="305" alt="Screenshot 2026-05-06 062012" src="https://github.com/user-attachments/assets/ecd048c9-d347-43dd-b74f-186f9601c78f" />


### Test 05: Time Trend
<img width="479" height="157" alt="Screenshot 2026-05-06 062331" src="https://github.com/user-attachments/assets/db020504-491e-4106-af95-a9293b952576" />


### Test 06: Cross Dimension Stress Test
<img width="506" height="175" alt="Screenshot 2026-05-06 070211" src="https://github.com/user-attachments/assets/3c31efbb-05b7-4c29-a1f8-bcbf3596c547" />


### Tools Used

- Power BI Desktop
- Power Query (M)
- DAX


### Dataset

DataCo Global Supply Chain Dataset: 180,519 rows, 53 columns. Covers order transactions across 5 markets: Europe, LATAM, Pacific Asia, USCA, and Africa.

