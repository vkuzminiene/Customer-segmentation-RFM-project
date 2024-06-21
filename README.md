# Customer-segmentation-RFM-project
Project contains task description, SQL code filtering and extracting data, visualizing data and insights from the data.

Here you can find my full project for Customer segmentation and Recency, Frequency, Monetary (RFM): 
https://docs.google.com/spreadsheets/d/1yb5GCMiNCAcODzuNQ1lqC2XSNPj7EdvFcBeOGHYrLD0/edit?usp=sharing 
If you want to check original visualisations, you can find them here: 
https://public.tableau.com/views/M3S3RFMvisualisationsbyVK/RFMsegmentation?:language=en-US&publish=yes&:sid=&:display_count=n&:origin=viz_share_link

___
SQL query:
WITH invoices AS -- CTE counting total spend sum per invoice
(
SELECT
DISTINCT InvoiceNo AS unique_invoices,
ROUND(SUM (UnitPrice*Quantity) OVER (PARTITION BY InvoiceNo),2) AS sum_per_invoice
FROM `tc-da-1.turing_data_analytics.rfm`
WHERE Quantity>0 --excluding not relate to customer data
AND UnitPrice>0 --excluding negative values with DESCRIPTION "adjust bad debt"
),
last_purchase AS ( -- CTE finding last purchase to count recency
SELECT
CustomerID,
MAX(InvoiceDate) AS last_purchase_date
FROM `tc-da-1.turing_data_analytics.rfm`
WHERE CAST(InvoiceDate AS DATE) BETWEEN '2010-12-01' AND '2011-12-01'
GROUP BY CustomerID
),
r_score AS -- CTE counting recency score
(SELECT
CustomerID,
NTILE(5) OVER (ORDER BY DATE_DIFF('2011-12-01', DATE(last_purchase.last_purchase_date), DAY)) AS recency_score
FROM last_purchase
)
,
f_score AS -- CTE counting frequency score
(
SELECT
CustomerID,
NTILE(5) OVER (ORDER BY COUNT(DISTINCT rfm.InvoiceNo)) AS frequency_score
FROM
`tc-da-1.turing_data_analytics.rfm` AS rfm
GROUP BY CustomerID
),
m_score AS ( -- CTE counting monetary score
SELECT
CustomerID,
NTILE(5) OVER (ORDER BY SUM(sum_per_invoice)) AS monetary_score,
FROM (
SELECT
rfm.CustomerID,
ROUND(SUM(DISTINCT invoices.sum_per_invoice) OVER (PARTITION BY rfm.CustomerID), 2) AS sum_per_invoice
FROM `tc-da-1.turing_data_analytics.rfm` AS rfm
JOIN invoices ON rfm.InvoiceNo = invoices.unique_invoices
WHERE rfm.CustomerID IS NOT NULL
AND CAST(InvoiceDate AS DATE) BETWEEN '2010-12-01' AND '2011-12-01'
GROUP BY rfm.CustomerID, invoices.sum_per_invoice, rfm.InvoiceNo
)
GROUP BY CustomerID
),
rfm_segmentation_score AS ( --CTE for 3 digits segmentation score
SELECT
DISTINCT rfm.CustomerId,
recency_score,
frequency_score,
monetary_score,
CONCAT(recency_score, frequency_score, monetary_score) AS combined_score
FROM `tc-da-1.turing_data_analytics.rfm` AS rfm
JOIN invoices
ON rfm.InvoiceNo=invoices.unique_invoices
JOIN last_purchase
ON rfm.CustomerID=last_purchase.CustomerID
LEFT JOIN m_score
ON rfm.CustomerID = m_score.CustomerID
LEFT JOIN f_score
ON rfm.CustomerID=f_score.CustomerID
LEFT JOIN r_score
ON rfm.CustomerID=r_score.CustomerID
WHERE rfm.CustomerID IS NOT NULL
AND CAST(InvoiceDate AS DATE) BETWEEN '2010-12-01' AND '2011-12-01'
),
rfm_segmentation AS ( -- CTE describing segments
SELECT
*,
CASE
WHEN rfm_segmentation_score.combined_score IN ('555', '554', '544', '545', '454', '455', '445') THEN 'Champion'
WHEN rfm_segmentation_score.combined_score IN ('543', '444', '435', '355', '354', '345', '344', '335') THEN 'Loyal'
WHEN rfm_segmentation_score.combined_score IN ('553', '551', '552', '541', '542', '533', '532', '531', '452', '451', '442', '441', '431', '453', '433', '432', '423', '353', '352', '351', '342', '341', '333', '323') THEN 'Potential Loyalist'
WHEN rfm_segmentation_score.combined_score IN ('512', '511', '422', '421', '412', '411', '311') THEN 'New Customer'
WHEN rfm_segmentation_score.combined_score IN ('525', '524', '523', '522', '521', '515', '514', '513', '425','424', '413','414','415', '315', '314', '313') THEN 'Promising'
WHEN rfm_segmentation_score.combined_score IN ('535', '534', '443', '434', '343', '334', '325', '324') THEN 'Need Atention'
WHEN rfm_segmentation_score.combined_score IN ('331', '321', '312', '221', '213', '231','241', '251') THEN 'About To Sleep'
WHEN rfm_segmentation_score.combined_score IN ('155', '154', '144', '214','215','115', '114', '113') THEN 'Loosing them'
WHEN rfm_segmentation_score.combined_score IN ('255', '254', '245', '244', '253', '252', '243', '242', '235', '234', '225', '224', '153', '152', '145', '143', '142', '135', '134', '133', '125', '124') THEN 'At Risk'
WHEN rfm_segmentation_score.combined_score IN ('332', '322', '233', '232', '223', '222', '132', '123', '122','212', '211') THEN 'Hibernating'
WHEN rfm_segmentation_score.combined_score IN ('111', '112', '121', '131', '141', '151') THEN 'Lost'
ELSE '0'
END AS segment
FROM rfm_segmentation_score
)
SELECT -- main query counting segment size
DISTINCT segment,
COUNT(*) OVER (PARTITION BY segment) AS segment_size
FROM rfm_segmentation
ORDER BY segment_size;
