## Procurement kpi Analysis

# PROJECT OVERVIEW
  This project explores procurement performance metrics using SQL (MySQL).
  The dataset contains procurement-related KPIs such as product categories, supplier information, quantities, delivery status, lead times, and costs.

# DATASET INFORMATION

    | Column               | Description                                    |
    | -------------------- | ---------------------------------------------- |
    | `PO_ID`              | Purchase Order ID                              |
    | `Supplier`           | Supplier Name                                  |
    | `Item_Category`      | Category of the item                           |
    | `Quantity`           | Number of items ordered                        |
    | `Price`              | Price per unit                                 |
    | `Order_Status`       | Status of the order (e.g., Delivered, Pending) |
    | `Delivery_Time_Days` | Days taken to deliver                          |
    | `Shipping_Cost`      | Cost of shipping                               |
    | `Order_Date`         | Date of order                                  |
    | `Delivery_Date`      | Date of delivery                               |
    | `Lead_Time_Days`     | Supplier lead time                             |

# TOOLS
 - MySQL – Data exploration and analysis
 - Tableau – For visualization of results

## SKILLS
  - SQL Joins,
  - CTEs,
  - CASE,
  - Subqueries,
  - Aggregate Functions

## QUESTIONS 

# 1. Supplier Performance
  Which supplier has delivered the highest total number of units excluding defective units? Only include Delivered orders.
        
    SELECT 
        Supplier,
        SUM(Quantity - COALESCE(Defective_Units, 0)) AS Net_Units_Delivered
    FROM procurement_kpi
    WHERE Order_Status = 'Delivered'
    GROUP BY Supplier
    ORDER BY Net_Units_Delivered DESC
    LIMIT 1;
<img width="608" height="138" alt="Screenshot 2025-08-11 193026" src="https://github.com/user-attachments/assets/80cb595d-d0f9-4703-a687-7cbd631fcbb1" />


# 2. Category Quality Analysis
  Which item category has the highest average defect rate?
  Defect Rate = (Defective_Units / Quantity) * 100
  Only include Delivered orders, and ignore orders where Defective_Units is NULL.
  
    SELECT
        Item_Category,
        ROUND(AVG((Defective_Units / Quantity) * 100), 2) AS Average_Defect_Rate
    FROM procurement_kpi
    WHERE Order_Status = 'Delivered'
    AND Defective_Units IS NOT NULL
    AND Quantity > 0
    GROUP BY Item_Category
    ORDER BY Average_Defect_Rate DESC;
    
<img width="495" height="146" alt="Screenshot 2025-08-11 193207" src="https://github.com/user-attachments/assets/a42b9cf7-6328-4920-a628-f197cd3088c8" />


# 3. Supplier Pricing Analysis
  For each supplier, list all their Purchase Orders where the negotiated price per unit is higher than that supplier’s average negotiated price across all their delivered orders.

    SELECT 
        PO_ID,
        Supplier,
        Negotiated_Price,
        (SELECT AVG(Negotiated_Price)
    FROM procurement_kpi AS sub
    WHERE sub.Supplier = pk.Supplier
    AND sub.Order_Status = 'Delivered') AS Average_Negotiated_Price,
        Order_Date
    FROM procurement_kpi AS pk
    WHERE Order_Status = 'Delivered'
    AND Negotiated_Price > (SELECT AVG(Negotiated_Price)
                          FROM procurement_kpi AS sub
                          WHERE sub.Supplier = pk.Supplier
                            AND sub.Order_Status = 'Delivered');
                            
<img width="517" height="295" alt="Screenshot 2025-08-11 193416" src="https://github.com/user-attachments/assets/f62f324c-6ef4-41c6-8de3-1b6d7def5ff1" />


# 4. Procurement Spend Classification
  Analyze total procurement spend per supplier and classify their spend tier based on total actual spend.

    SELECT
        supplier,
        SUM(quantity * negotiated_price) AS total_actual_spend,
    CASE
        WHEN SUM(quantity * negotiated_price) > 100000 THEN 'high'
        WHEN SUM(quantity * negotiated_price) BETWEEN 50000 AND 100000 THEN 'medium'
        ELSE 'low'
    END AS spend_tier
    FROM procurement_kpi
    WHERE Order_Status = 'Delivered'
    GROUP BY supplier
    ORDER BY total_actual_spend;

<img width="491" height="137" alt="Screenshot 2025-08-11 193511" src="https://github.com/user-attachments/assets/1917bd3a-feda-4f74-9ef5-6c334e585909" />


# 5. Delivery Time Performance

For each supplier, calculate:

- Their average delivery time (in days)
- Count of total delivered orders

Classify their delivery speed using:

- 'Fast' if avg days ≤ 7
- 'Moderate' if between 8 and 14
- 'Slow' if > 14

      SELECT
          supplier,
          AVG(DATEDIFF(delivery_date, order_date)) AS average_delivery_time,
          COUNT(*) AS delivered_order_count,
        CASE
          WHEN AVG(DATEDIFF(delivery_date, order_date)) <= 7 THEN 'fast'
          WHEN AVG(DATEDIFF(delivery_date, order_date)) BETWEEN 8 AND 14 THEN 'moderate'
          WHEN AVG(DATEDIFF(delivery_date, order_date)) > 14 THEN 'slow'
          ELSE 'unknown'
          END AS delivery_speed
      FROM procurement_kpi
      WHERE Order_Status = 'Delivered'
      GROUP BY supplier
      ORDER BY average_delivery_time;

<img width="497" height="138" alt="Screenshot 2025-08-11 193622" src="https://github.com/user-attachments/assets/94b93cf4-b7ec-45ce-b191-d4dd45e0408b" />


# 6. Cost Control Alert: Price Overruns
  Identify all Purchase Orders where the supplier charged more per unit than the average unit price for that item category.

    SELECT
        po_id,
        supplier,
        item_category,
        unit_price,
        (SELECT AVG(Unit_Price)
            FROM procurement_kpi AS sub
            WHERE sub.Item_Category = main.Item_Category
            AND sub.Order_Status = 'delivered') AS avg_category_unit_price
    FROM procurement_kpi AS main
    WHERE main.Unit_Price > (
                              SELECT AVG(Unit_Price)
                              FROM procurement_kpi AS sub
                              WHERE sub.Item_Category = main.Item_Category
                              AND sub.Order_Status = 'delivered'
                            )
    AND main.Order_Status = 'delivered'
    ORDER BY item_category;

<img width="502" height="298" alt="Screenshot 2025-08-11 193819" src="https://github.com/user-attachments/assets/77b10edb-5ce3-4c63-ab0f-a237e8e70151" />


# 7. A list of delivered purchase orders 
  where the quantity ordered is above the average quantity of all delivered orders.

    WITH avg_delivered_qty AS (
        SELECT AVG(quantity) AS avg_qty
        FROM procurement_kpi
        WHERE Order_Status = 'Delivered'
        )
    SELECT 
        PO_ID, 
        Supplier, 
        Quantity, 
        Order_Status
    FROM procurement_kpi, avg_delivered_qty
    WHERE quantity > avg_qty
    AND Order_Status = 'Delivered';

<img width="460" height="296" alt="Screenshot 2025-08-11 194009" src="https://github.com/user-attachments/assets/b77dcaee-7f52-48fa-b71b-6c83c799b05a" />


# 8. Which purchase orders had a quantity higher than the average for their own item category (among delivered orders)?

    WITH category_avg_quantity AS (
        SELECT
            Item_Category,
            AVG(quantity) AS avg_qty
        FROM procurement_kpi
        WHERE Order_Status = 'Delivered'
        GROUP BY Item_Category
        )
    SELECT
        pk.PO_ID,
        pk.Supplier,
        pk.Item_Category,
        pk.quantity,
        cte.avg_qty AS category_avg_quantity
    FROM procurement_kpi AS pk
    JOIN category_avg_quantity AS cte
        ON pk.Item_Category = cte.Item_Category
    WHERE
        pk.Order_Status = 'Delivered'
        AND pk.quantity > cte.avg_qty;

<img width="482" height="294" alt="Screenshot 2025-08-11 194046" src="https://github.com/user-attachments/assets/e3aeb75a-4f31-4787-b7f8-f1741764409c" />

# Visual Insights (Tableau) 
  Here are some visual insights created in Tableau:
  
  - Supplier Performance Dashboard
    <img width="1531" height="778" alt="Screenshot 2025-08-13 192010" src="https://github.com/user-attachments/assets/9c610eb5-69d9-48ad-84a2-c7878373a820" />

  - Category Quality Analysis
    <img width="1533" height="781" alt="Screenshot 2025-08-13 192138" src="https://github.com/user-attachments/assets/4bf09505-7ea4-4c45-b28b-050db6af590c" />

  - Supplier Pricing Analysis
    <img width="1537" height="781" alt="Screenshot 2025-08-13 192238" src="https://github.com/user-attachments/assets/8d8b14c4-6353-4d32-a900-063ff9f90ad6" />

  - Procurement Spend Classification
    <img width="1538" height="781" alt="Screenshot 2025-08-13 192347" src="https://github.com/user-attachments/assets/d66ef151-49f7-451f-a1f1-4c5f0bbf7bb6" />

  - Delivery Time Performance
    <img width="1531" height="782" alt="Screenshot 2025-08-13 192442" src="https://github.com/user-attachments/assets/333b175c-d874-4894-ae00-710899cb3e57" />

  - Cost Control Alert - Price Overruns
    <img width="1536" height="782" alt="Screenshot 2025-08-13 192525" src="https://github.com/user-attachments/assets/67bd2a46-823b-443a-86fa-b2bac0eb6b44" />

  - A list of delivered purchase orders
    <img width="445" height="284" alt="Screenshot 2025-08-13 192602" src="https://github.com/user-attachments/assets/4a09cfd1-a107-4915-8048-f4a4673e022f" />

  - Quantity higher than the average
    <img width="1537" height="780" alt="Screenshot 2025-08-13 192639" src="https://github.com/user-attachments/assets/3e68ae78-b64b-4c0b-acb4-47ca59aada7d" />

# Key Findings
  - Defective Rates – Raw materials recorded the highest defect rates, while MRO items had the lowest, highlighting potential quality control gaps in raw material procurement.
  - Supplier Price Trends – Gamma Co’s negotiated price fell from 2,033.1 in 2022 to 1,886.6 in 2023, while Beta Supply’s price surged from 1,530.5 to 2,462.5, indicating contrasting supplier cost dynamics.
  - Top Performer in Delivery Volume – Epsilon Group delivered 123,693 more units than any other supplier, while Alpha Inc, Gamma Co, and Delta Logistics each recorded deliveries below 114,000 units.
  - Procurement Spend vs. Supplier Performance – Epsilon Group recorded the highest procurement spend, while Alpha Group had the lowest. A potential positive correlation exists between procurement spend and supplier performance.
  - Delivery Time Performance – Gamma Co achieved the shortest average delivery time (9 days), while Beta Supplies recorded the longest (11 days).
  - Cost vs. Delivery Time – Gamma Co incurred higher costs than Beta Supplies, but no clear relationship was found between cost incurred and delivery time.
  - Purchase Order Fulfillment Leaders – Delta Logistics and Beta Supplies led in the number of delivered purchase orders, while Gamma Co and Alpha Inc lagged behind.
  - Delivery Time vs. Fulfillment – Beta Supplies’ long delivery time may be linked to their high purchase order volume. However, faster delivery does not necessarily lead to higher fulfillment rates, as seen with Gamma Co.

# Possible Cause
  - Lack of standardized inspections before shipment.
  - Variations in logistics partnerships, geographic proximity, or production capacity.
  - Limited supplier pool or strategic sourcing not applied.
  - Focus on volume fulfillment over quality assurance.

# Recommendations
  - Conduct quarterly supplier quality audits for raw material vendors.
  - Maintain good relationships with Gamma Co and explore opportunities to lock in lower rates.
  - Identify and reward top-performing suppliers with preferred vendor status.
  - Introduce performance-based contracts linking volume bonuses to defect thresholds.

# Author
 - Caleb Eboigbe - Data Analyst (Supply Chain & finance Focus)
 - Contact: [caleb.villa13@gmail.com]

