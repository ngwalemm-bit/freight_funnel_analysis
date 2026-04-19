# Client Conversion Funnel Analysis in Freight Forwarding

This project analyzes the quote-to-booking funnel in a freight forwarding business to identify where clients drop off, what drives non-conversion, and which operational or commercial factors reduce conversion performance.

## Project Overview

The goal of this project is to analyse the client conversion funnel within a freight forwarding business, from quote request through to shipment completion, in order to identify:
### 	Where potential clients drop off
### 	What factors influence conversion
### 	Key operational inefficiencies affecting revenue


## Business Problem
Where are we losing potential clients in the quote-to-booking process?

Despite high quote volumes, only a small percentage convert into actual shipments. This analysis investigates the root causes behind this gap.

## Tools Used
### 	Excel → Data cleaning & initial exploration
### 	SQL (PostgreSQL) → Data transformation & analysis
### Power BI → Dashboard & visualization

## Dataset Description

The dataset contains 4,000 quote records across the following funnel stages:
### 	1.	Quote Requested
### 	2.	Quote Sent
### 	3.	Quote Viewed
### 	4.	Quote Accepted
### 	5.	Shipment Booked
### 	6.	Shipment Completed

#### It also includes:
##### Pricing data
##### Response times
##### Customer segments
##### Drop-off reasons
##### Routes and sales reps

## Data Cleaning 

Data cleaning was performed in both Excel and SQL to ensure analytical accuracy.

#### Key steps (Excel):
##### Standardized Customer Segment values (e.g. SME, sme → SME)
##### Removed invalid numeric values:
##### Quoted amounts = 0
##### Negative response times
##### Fixed drop-off logic:
##### Removed drop-off stages/reasons for Booked/Completed records
##### Handled missing values appropriately
##### Ensured correct data types for numerical analysis (especially pricing)

### Exploratory Analysis (Excel)
Initial analysis was conducted using pivot tables to understand:
Funnel Distribution
•	Sent: 1369
•	Viewed: 1269
•	Accepted: 682
•	Completed: 452
•	Booked: 228

### Drop-Off Insights
#### Highest drop-off occurs at:
##### Quote Sent
##### Quote Viewed
##### Quote Accepted

#### Key Observation
A large portion of quotes fail to progress beyond early stages, indicating inefficiencies in conversion despite strong initial customer interest.

## Data Cleaning (SQL)


``` sql
CREATE VIEW quotes_funnel_clean AS
SELECT 
		quote_id,
		customer_id,
		sales_rep_id,
		product_id,
		quote_request_date,
		quote_sent_date,
		quote_viewed_date,
		quote_accepted_date,
		shipment_booked_date,
		shipment_completed_date,
		origin_country,
		destination_country,
		TRIM(route) AS route,
		transport_mode,
		TRIM(carrier) AS carrier,
		quoted_amount_usd,
		final_booked_amount_usd,
		lead_time_days,
		CASE 
			WHEN response_time_hours < 0 THEN NULL
			ELSE response_time_hours
		END AS response_time_hours,
		quote_status,
		CASE 
			WHEN quote_status IN ('Booked', 'Completed') THEN null
			ELSE drop_off_stage
		END AS drop_off_stage,
			CASE WHEN quote_status IN ('Booked', 'Completed') THEN null
			ELSE drop_off_reason
		END AS drop_off_reason,
		priority_level,
		CASE
			WHEN LOWER(customer_segment) IN ('sme', 'small business')THEN 'SME'
			ELSE customer_segment
		END AS customer_segment	
FROM	quotes_funnel;
``` 


## Key Analysis & Insights 

SQL was used to perform deeper analysis and validate findings.

### 1. Overall Conversion Rate

``` sql
SELECT
    COUNT(*) AS total_quotes,
    SUM(CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 1 
        ELSE 0 
    END) AS converted_quotes,
    ROUND(
        SUM(CASE 
            WHEN quote_status IN ('Booked', 'Completed') THEN 1 
            ELSE 0 
        END) * 100.0 / COUNT(*),
        2
    ) AS conversion_rate_pct
FROM quotes_funnel_clean;
```

Only 17% of quote requests result in booked or completed shipments, indicating significant inefficiencies in the funnel.


### 2. Funnel Distribution (%)

``` sql
SELECT
    quote_status,
    COUNT(*) AS total_quotes,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM quotes_funnel_clean
GROUP BY quote_status
ORDER BY total_quotes DESC;
```

#### Majority of quotes are stuck in early stages:
#### Sent (34%)
#### Viewed (31%)
#### Suggests strong lead generation but weak conversion

### 3. Drop-Off Drivers

``` sql
SELECT
    drop_off_reason,
    COUNT(*) AS total_dropoffs,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER (), 2) AS pct_of_total
FROM quotes_funnel_clean
WHERE drop_off_reason IS NOT NULL
GROUP BY drop_off_reason
ORDER BY total_dropoffs DESC;
```

### Top reasons:
#### Competitor Won (17.6%)
#### Route Constraint (16.86%)
#### No Feedback (16.72%)

Drop-offs are evenly distributed across multiple causes, suggesting that conversion issues are multi-dimensional rather than driven by a single factor. Other contributing factors may include competitive pressure, weak routing strategies, and poor internal tracking processes.

### 4. Pricing Impact

``` sql
SELECT
    CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 'Converted'
        ELSE 'Not Converted'
    END AS conversion_group,
    COUNT(*) AS total_quotes,
    ROUND(AVG(quoted_amount_usd), 2) AS avg_quoted_amount
FROM quotes_funnel_clean
WHERE quoted_amount_usd IS NOT NULL
AND quoted_amount_usd > 0
GROUP BY conversion_group;
```

#### Converted avg: 10,569 USD
#### Not converted avg: 10,674 USD

Price has some influence on conversion, but the difference is small, indicating that pricing alone does not explain performance gaps.

### 5. Response Time Impact

``` sql
SELECT
    CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 'Converted'
        ELSE 'Not Converted'
    END AS conversion_group,
    COUNT(*) AS total_quotes,
    ROUND(AVG(response_time_hours), 2) AS avg_response_time
FROM quotes_funnel_clean
WHERE response_time_hours IS NOT NULL
GROUP BY conversion_group;
```

#### Converted: 35.91 hrs
#### Not Converted: 36.41 hrs

Faster response times slightly improve conversion, suggesting speed plays a role in competitiveness.

### 6. Customer Segment Performance

``` sql
SELECT
    customer_segment,
    COUNT(*) AS total_quotes,
    SUM(CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 1 
        ELSE 0 
    END) AS converted_quotes,
    ROUND(
        SUM(CASE 
            WHEN quote_status IN ('Booked', 'Completed') THEN 1 
            ELSE 0 
        END) * 100.0 / COUNT(*),
        2
    ) AS conversion_rate_pct
FROM quotes_funnel_clean
GROUP BY customer_segment
ORDER BY conversion_rate_pct DESC;
```

|Segment	Conversion Rate
|_________________________
|Enterprise	   | 19.75%    
|SME	         |  17.22%
|Retail	       | 15.87%
|Manufacturing | 14.90%

Enterprise clients convert best, likely due to established relationships, while SMEs generate more quotes but convert less efficiently. SMEs drive traffic, enterprises drive revenue.

### 7. Route Performance

``` sql
SELECT
    route,
    COUNT(*) AS total_quotes,
    SUM(CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 1 
        ELSE 0 
    END) AS converted_quotes,
    ROUND(
        SUM(CASE 
            WHEN quote_status IN ('Booked', 'Completed') THEN 1 
            ELSE 0 
        END) * 100.0 / COUNT(*),
        2
    ) AS conversion_rate_pct
FROM quotes_funnel_clean
GROUP BY route
ORDER BY conversion_rate_pct ASC;
```

Low-performing routes include:
#### Germany → UK
#### China → UK
#### South Africa → India

Conversion varies significantly by route, indicating that carrier relationships and route optimization impact success.

### 8. Sales Rep Performance

``` sql
SELECT
    sales_rep_id,
    COUNT(*) AS total_quotes,
    SUM(CASE 
        WHEN quote_status IN ('Booked', 'Completed') THEN 1 
        ELSE 0 
    END) AS converted_quotes,
    ROUND(
        SUM(CASE 
            WHEN quote_status IN ('Booked', 'Completed') THEN 1 
            ELSE 0 
        END) * 100.0 / COUNT(*),
        2
    ) AS conversion_rate_pct
FROM quotes_funnel_clean
GROUP BY sales_rep_id
ORDER BY conversion_rate_pct DESC;
```

#### Best: ~20.79%
#### Lowest: ~12.77%

Performance varies across sales reps, suggesting differences in customer targeting, follow-up, or engagement strategies.

## Key Insights

### The business generates strong quote volume but fails to convert effectively
### Conversion issues are caused by a combination of:
#### Competitive pressure
#### Operational constraints (routes)
#### Weak follow-up processes
### Pricing matters, but is not the primary driver
### Customer segment and sales rep behavior significantly influence outcomes

## Power BI Dashboard
The Power BI dashboard provides a visual representation of the client conversion funnel, highlighting where and why potential clients drop off, as well as key factors influencing conversion.

### Dashboard Overview

![App Screenshot](images/funnel.png)

### Key Dashboard Insights:

#### ⁠The majority of quotes remain in early funnel stages (Sent & Viewed), confirming low conversion efficiency
#### ⁠Drop-off reasons are distributed across multiple factors, with competitor pricing, route constraints, and lack of feedback being the most prominent
#### ⁠Faster response times are associated with higher conversion rates
#### ⁠Enterprise clients show higher conversion efficiency compared to SMEs
#### ⁠Pricing differences between converted and non-converted quotes are minimal, suggesting non-price factors play a significant role

## Business Recommendations
### 1. Improve Operational Efficiency
#### Strengthen post-quote follow-ups
#### Align pricing, operations, and sales teams


### 2. Optimize Route Strategy
#### Improve carrier partnerships
#### Focus on high-performing trade lanes


### 3. Reduce Response Time
#### Implement SLAs for quote turnaround
#### Automate parts of the quoting process


### 4. Improve Data Tracking
#### Enforce mandatory drop-off reason logging
#### Reduce “No Feedback” gaps

### 5. Refine Sales Strategy
#### Train low-performing reps
#### Focus on high-conversion client segments

## Conclusion

While there is strong client interest, operational gaps, routing limitations, and inconsistent sales execution significantly reduce revenue potential.

Overall, the business does not have a demand problem — it has a *conversion efficiency problem, which presents a clear opportunity for improvement through better alignment between sales, pricing, and operations.

## Limitations

#### High number of missing drop-off reasons reduces clarity
#### Dataset is synthetic (simulated), may not reflect real-world constraints fully
#### External factors (market competition, client budgets) not captured
#### ⁠Drop-off reason data contains inconsistencies (blank vs “No Feedback”), which may slightly affect categorization
#### ⁠Dataset is simulated and may not reflect real-world operational complexity

