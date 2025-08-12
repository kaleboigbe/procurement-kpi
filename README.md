## Procurement-kpi-Analysis

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


# Author
 - Caleb Eboigbe - Data Analyst (Supply Chain & finance Focus)
 - Contact: [caleb.villa13@gmail.com]

