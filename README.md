
--1 Average loan amount in this year

SELECT 
    year,
    AVG(loan_amount) AS avg_loan_amount
FROM (
    SELECT 
        ld.year,
        al.loan_amount
    FROM loan_default ld
    JOIN amount_of_loan al ON ld.id = al.id
) AS derived_table
GROUP BY year
ORDER BY avg_loan_amount DESC;



--2 This query identifies patterns where loans with higher interest rate spreads are more likely to default. 
--This query helps spot risky loans that are more likely to default based on high-interest rate spreads. This can be used to re-evaluate loan pricing models.
 
 WITH default_risk AS (
    SELECT al.id, al.loan_amount, al.Interest_rate_spread, ld.loan_type, ld.Credit_Worthiness
    FROM amount_of_loan al
    JOIN loan_default ld ON al.id = ld.id
    WHERE  ld.loan_limit IS NOT NULL -- Focus on loans with limits
      AND al.Interest_rate_spread > 1.5 -- High interest rate spread
)
SELECT id, loan_amount, Interest_rate_spread, loan_type, Credit_Worthiness
FROM default_risk
ORDER BY Interest_rate_spread DESC;


--3 This query explores patterns of pre-approved loans based on loan purpose and gender. It can help detect any gender-based biases in pre-approval decisions and how they relate to different loan purposes.
--This query highlights how different loan purposes and genders are treated in pre-approvals. The results can guide decisions on whether pre-approval processes need to be adjusted to ensure fairness across different demographics and purposes.

SELECT ld.loan_purpose, ld.Gender, COUNT(*) AS total_loans, 
       SUM(CASE WHEN ld.approv_in_adv = 'Yes' THEN 1 ELSE 0 END) AS pre_approved_loans,
       AVG(al.loan_amount) AS avg_loan_amount, 
       (SUM(CASE WHEN ld.approv_in_adv = 'Yes' THEN 1 ELSE 0 END) * 1.0 / COUNT(*)) AS pre_approval_rate
FROM loan_default ld
JOIN amount_of_loan al ON ld.id = al.id
WHERE ld.loan_purpose IS NOT NULL AND ld.Gender IS NOT NULL
GROUP BY ld.loan_purpose, ld.Gender
HAVING COUNT(*) > 5 -- Only include groups with more than 5 loans
ORDER BY pre_approval_rate DESC;



--4 Analyze whether lower credit scores lead to longer loan terms and higher default rates.
--This will show if lower credit scores tend to have longer loan terms, which in turn might lead to higher default risks.

WITH low_credit_loans AS (
    SELECT 
        ld.id,
        ld.Credit_Score,
        al.term
    FROM 
        loan_details ld
    JOIN 
        amount_of_loan al ON ld.id = al.id
    WHERE 
        ld.Credit_Score < 600
)
SELECT 
    lcl.id,
    lcl.Credit_Score,
    lcl.term,
    CASE 
        WHEN ldft.loan_limit IS NULL THEN 'Defaulted'
        ELSE 'Not Defaulted'
    END AS default_status
FROM 
    low_credit_loans lcl
LEFT JOIN 
    loan_default ldft ON lcl.id = ldft.id
ORDER BY 
    lcl.Credit_Score;




--5 Explore whether longer loan terms have resulted in significantly higher or lower interest rates over the years, showing the evolution of risk perception.
-- This query groups loans into different tenure buckets and tracks the average interest rate for each group over the years, allowing to see how loan tenure correlates with interest rates and if lenders are offering different rates based on loan durations over time.

WITH loan_trends AS (
    SELECT 
        al.term,
        al.rate_of_interest,
        ld.year,
        ROW_NUMBER() OVER (PARTITION BY ld.year ORDER BY al.term DESC) AS rank_within_year,
        NTILE(3) OVER (ORDER BY al.term) AS loan_term_bucket
    FROM amount_of_loan al
    JOIN loan_default ld ON al.id = ld.id
    WHERE al.term IS NOT NULL AND al.rate_of_interest IS NOT NULL
)
SELECT 
    lt.year,
    lt.loan_term_bucket,
    AVG(lt.rate_of_interest) AS avg_interest_rate,
    COUNT(*) AS total_loans
FROM loan_trends lt
GROUP BY lt.year, lt.loan_term_bucket
ORDER BY lt.year, lt.loan_term_bucket;



--6 Determine whether certain loan types  lead to significantly higher LTV ratios compared to others and how LTV changes across property values.
--This query gives an understanding of whether higher property values lead to lower or higher LTV ratios across different loan types, which may indicate the relative risk tolerance of lenders for particular loan products.

WITH ltv_analysis AS (
    SELECT 
        ld.id,
        ld.loan_amount,
        ld.property_value,
        ld.LTV,
        ldf.loan_type,   -- Retrieve loan_type from loan_default
        CASE 
            WHEN ld.property_value < 100000 THEN 'Low Value Property'
            WHEN ld.property_value BETWEEN 100000 AND 500000 THEN 'Mid Value Property'
            ELSE 'High Value Property'
        END AS property_value_category
    FROM loan_details ld
    JOIN loan_default ldf ON ld.id = ldf.id  -- Join to get loan_type
    WHERE ld.LTV IS NOT NULL AND ld.property_value IS NOT NULL
)
SELECT 
    loan_type,
    property_value_category,
    AVG(LTV) AS avg_ltv,
    COUNT(*) AS total_loans
FROM ltv_analysis
GROUP BY loan_type, property_value_category
ORDER BY loan_type, property_value_category;


--7 Compare the interest rates and terms between business/commercial loans and personal loans.
--This query helps you understand if business or commercial loans have significantly different terms compared to personal loans, providing useful information for product development or risk analysis.

SELECT 
    ld.business_or_commercial,
    AVG(al.loan_amount) AS avg_loan_amount,
    AVG(al.rate_of_interest) AS avg_interest_rate,
    COUNT(ld.id) AS total_loans
FROM loan_default ld
JOIN amount_of_loan al ON ld.id = al.id
GROUP BY ld.business_or_commercial
ORDER BY ld.business_or_commercial;


--8 Top 4 Ages with the Highest Average Loan Amounts

SELECT 
    age,
    AVG(loan_amount) AS avg_loan_amount
FROM (
    SELECT 
        ld.age,
        al.loan_amount
    FROM loan_details ld
    JOIN amount_of_loan al ON ld.id = al.id
) AS derived_table
GROUP BY age
ORDER BY avg_loan_amount DESC
LIMIT 4;

