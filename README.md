# JCars Logistics Sales & Performance Analysis
JCars Logistics operates across 8 branches in Kenya, selling vehicles to individual customers, car dealers, corporates, government agencies and NGOs. The goal of this project is to transform a dataset provided by JCars into a reliable data model and to create reports that provides insights into revenue, profitability, branch, payment, delivery and logistics and customer behaviour while also providing clear recommendations for management.
## Dataset
- Source: [`Jcars_data.csv`](data/Jcars_data.csv), 32 columns, one row per order.
- Fields cover customer details, location, vehicle specs, pricing/cost, payment and delivery.

## Data Quality Issues Identified
- Misspelt/abbreviated categories (e.g. `toyta` instead of `toyota`, `KSM` instead of `Kisumu`, `walkin` instead of `Walk in`)
- Double spaces, partial names, inconsistent capitalization (e.g. sales rep as just `Grace` instead of `Grace Njeri`, )
- Numbers written as words (`thirty`, `twenty twenty`, `ten percent`)
- Millions shorthand (`6.6M`)
- Mixed currencies in cost fields (KES, USD, ZAR, and a corrupted `?` symbol)
- Error/placeholder text (`#VALUE!`, `error`, `NULL`, `missing`, `-`)
- Out-of-range values (negative/zero cost, ages outside 18-100)
- Inconsistent date formats and Excel serial numbers in `Order Date` and `Delivery Date`

## Data Cleaning 
- **Category columns** (Car Make, Region, Branch, Sales Rep, Payment Status, Delivery Status, etc.) used **Replace Values** to fix known misspellings and abbreviations (e.g. `totoya` to `Toyota`)<br>**Trim/Clean** to remove extra spaces and non-printable characters. <br>Where a raw value only had a first name (e.g. Sales Rep recorded as just `Grace`), Replace Values with "Match entire cell contents" was used to expand it to the full name without corrupting already-correct rows.
<br>

- **Numeric columns** (Unit Cost, Unit Selling Price, Delivery Fee, Logistics Cost) - used **Replace Values** to strip currency symbols and thousands separators, <br>**Change Type to Decimal Number** to convert text to numbers (which turns unparseable text like `#VALUE!` or `error` into an Error value), then **Replace Errors  null** to safely convert those into true nulls. <br>A **Conditional Column** handled the "M" (millions) suffix by flagging rows ending in "M" and multiplying by 1,000,000.
- **Word-based numbers** (e.g. Customer Age as `thirty`, Vehicle Year as `Twenty Twenty`) - handled with **Replace Values**, mapping specific words to their equivalent digit before converting the column to a numeric type.
- **Expected Revenue** - added as custom column, calculated as `Units Sold × Unit Selling Price × (1 − Discount) + Delivery Fee`, kept separate from the raw `Revenue Recorded` column for reconciliation.

## Currency Standardisation
 
Columns **Unit Cost**, **Unit Selling Price**, **Delivery Fee**, **Logistics Cost** and **Revenue Recorded** were mixed KES, Ksh, USD, EUR, ZAR, R and an unrecognised `?` symbol. Exchange rates used in cleaning were:
    **USD - KES:** 129.45,<br> **EUR - KES:** 147.28, <br> **ZAR - KES:** 7.85
Here is how they were standardised to KES:
 
1. **Detect currency** from the prefix before stripping symbols, using a **Conditional Column** (`Rate Multiplier`): `USD`/`$`/? - 129.45, `EUR` - 147.28, `ZAR`/`R` - 7.85, KES/KSh/no label - 1.
2. The `?` symbol did not have a clearly identifiable currency. To determine the most appropriate currency, the affected values were converted using the USD, EUR, and ZAR exchange rates. The USD conversion produced results that were reasonable and fell within the expected range for each row.
3. Another **Conditional Column** was used to flag rows with a `M` suffix. For these rows the new column stored `1,000,000` as a multiplier, while all other rows were set to `1`.
4. **Replace Values** was used to strip the currency labels (`USD`, `EUR`, `ZAR`, `R`, `?`, `,`) from the original column, leaving just the raw digits, which was then converted to a numeric type.
5. **Combine the columns**. The three columns (the cleaned numeric value, the Rate Multiplier, and the millions multiplier) were then combined by multiplying them together (product) producing the final standardised value in KES.

## Data Model
 
Star schema: **`Fact_Sales`** (one row per order, numeric measures + foreign keys) surrounded by these dimension tables:
 
`dim_customer`, `dim_car`, `dim_location`, `dim_region`, `dim_date`, `dim_salesperson`, `dim_branch`

![star-schema screenshot](/images/star-schema.png)

## Executive Dashboard

![dashboard screenshot](/images/dashboard.png)

## Key Insights
 
- Toyota drives the business (`144 units`) but Isuzu generates the largest gross loss (`KSh 23.06M`), roughly offsetting Toyota's profit (`Ksh 23.26M`).
- Only 4 of the branches (`Athi River`, `Kakamega`, `Mombasa`, `Nakuru`) have a positive gross profit margin.
- Kakamega leads on units sold, revenue and fully paid orders but also carries the highest logistics cost alongside Thika.
- Delivery fees are insufficient to cover logistics costs, resulting in a KSh 2.99M shortfall.
- Over 50% of units for Mitsubishi, Mercedes, Mazda, Isuzu and BMW are not fully paid.
- The 55+ age band is the top-revenue customer segment.
- Individual customers have the highest share of fully paid units.
