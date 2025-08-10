# procurement-kpi

## Supplier Performance
  Which supplier has delivered the highest total number of units excluding defective units? Only include Delivered orders.
        
    SELECT 
        Supplier,
        SUM(Quantity - COALESCE(Defective_Units, 0)) AS Net_Units_Delivered
    FROM procurement_kpi
    WHERE Order_Status = 'Delivered'
    GROUP BY Supplier
    ORDER BY Net_Units_Delivered DESC
    LIMIT 1;
    

## Category Quality Analysis
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

## Supplier Pricing Analysis
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


## Procurement Spend Classification
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

## Delivery Time Performance

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

## Cost Control Alert: Price Overruns
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

## A list of delivered purchase orders 
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

## Which purchase orders had a quantity higher than the average for their own item category (among delivered orders)?

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



