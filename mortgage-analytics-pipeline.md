# Fannie Mae Mortgage Default Risk Analytics Pipeline
## Architecture Overview and Data Flow

### Phase 1: Data Ingestion & Validation
#### Data Sources Integration
- Historical mortgage performance data
- Current loan portfolio data 
- Credit bureau data
- Property valuation data
- Economic indicators
- Payment history records

#### Implementation Steps
1. Configure BigQuery Data Transfer API for automated data ingestion:
```sql
-- Create scheduled transfer for mortgage data
CREATE TRANSFER CONFIG
project_id = 'fannie-mae-analytics'
dataset_id = 'mortgage_data'
display_name = 'Mortgage Performance Data Transfer'
params = {
    'source_format': 'CSV',
    'write_disposition': 'WRITE_APPEND',
    'schema_update_options': ['ALLOW_FIELD_ADDITION']
}
schedule = '0 */6 * * *'  -- Run every 6 hours
```

2. Implement data validation checks using BigQuery:
```sql
-- Sample validation query for loan data completeness
CREATE OR REPLACE PROCEDURE validate_loan_data()
BEGIN
  WITH validation_results AS (
    SELECT 
      loan_id,
      CASE 
        WHEN principal_balance IS NULL THEN 'Missing principal balance'
        WHEN loan_term IS NULL THEN 'Missing loan term'
        WHEN interest_rate IS NULL THEN 'Missing interest rate'
        ELSE 'VALID'
      END as validation_status
    FROM `mortgage_data.loan_details`
  )
  INSERT INTO `validation_logs.loan_validation_results`
  SELECT * FROM validation_results
  WHERE validation_status != 'VALID';
END;
```

3. Configure data quality thresholds and alerts:
- Set up BigQuery scheduled queries for data quality monitoring
- Implement notification system for validation failures
- Track data lineage and maintain audit logs

### Phase 2: Data Transformation & Enrichment
#### Feature Engineering Pipeline
1. Create standardized features using BigQuery:
```sql
CREATE OR REPLACE TABLE derived_features.loan_risk_factors AS
SELECT
  loan_id,
  -- Payment history features
  COUNT(DISTINCT CASE WHEN days_delinquent > 30 THEN payment_date END) as num_30_day_delinq,
  MAX(days_delinquent) as max_delinquency,
  
  -- Loan characteristics
  original_loan_amount / property_value as original_ltv,
  current_balance / current_property_value as current_ltv,
  
  -- Borrower features
  credit_score,
  debt_to_income_ratio,
  
  -- Property features
  property_type,
  occupancy_status,
  
  -- Economic indicators
  unemployment_rate,
  home_price_index
FROM mortgage_data.loan_details
LEFT JOIN economic_data.indicators
  ON loan_details.geography = indicators.geography
  AND loan_details.date = indicators.date
GROUP BY loan_id;
```

2. Implement feature versioning:
- Use BigQuery table snapshots for point-in-time features
- Maintain feature documentation and metadata
- Track feature importance and usage metrics

### Phase 3: Model Development & Training
#### Model Pipeline Setup
1. Configure BigQuery ML environment:
```sql
-- Create logistic regression model for default prediction
CREATE OR REPLACE MODEL risk_models.default_prediction
OPTIONS(
  model_type='LOGISTIC_REG',
  input_label_cols=['default_flag'],
  DATA_SPLIT_METHOD='AUTO',
  DATA_SPLIT_EVAL_FRACTION=0.2
) AS
SELECT
  default_flag,
  num_30_day_delinq,
  max_delinquency,
  original_ltv,
  current_ltv,
  credit_score,
  debt_to_income_ratio,
  unemployment_rate,
  home_price_index
FROM derived_features.loan_risk_factors;
```

2. Implement model evaluation pipeline:
```sql
-- Evaluate model performance
SELECT
  *
FROM
  ML.EVALUATE(MODEL risk_models.default_prediction,
    (
    SELECT
      default_flag,
      num_30_day_delinq,
      max_delinquency,
      original_ltv,
      current_ltv,
      credit_score,
      debt_to_income_ratio,
      unemployment_rate,
      home_price_index
    FROM derived_features.loan_risk_factors
    WHERE evaluation_date = CURRENT_DATE()
    ));
```

### Phase 4: Risk Scoring & Reporting
#### Production Scoring Pipeline
1. Implement real-time scoring using BigQuery API:
```python
from google.cloud import bigquery

def score_loan_risk(loan_data):
    client = bigquery.Client()
    
    query = """
    SELECT
      ML.PREDICT(MODEL risk_models.default_prediction,
        (SELECT 
          num_30_day_delinq,
          max_delinquency,
          original_ltv,
          current_ltv,
          credit_score,
          debt_to_income_ratio,
          unemployment_rate,
          home_price_index
         FROM UNNEST(@loans) as loan_features))
    """
    
    job_config = bigquery.QueryJobConfig(
        query_parameters=[
            bigquery.ArrayQueryParameter("loans", "STRUCT", loan_data),
        ]
    )
    
    return client.query(query, job_config=job_config)
```

2. Create automated reporting pipeline:
```sql
-- Generate risk summary reports
CREATE OR REPLACE TABLE reporting.risk_summary
AS
SELECT
  reporting_date,
  risk_segment,
  COUNT(loan_id) as total_loans,
  AVG(default_probability) as avg_default_prob,
  SUM(CASE WHEN default_probability > 0.1 THEN 1 ELSE 0 END) as high_risk_loans,
  SUM(current_balance) as total_exposure
FROM risk_scoring.loan_risk_scores
GROUP BY reporting_date, risk_segment;
```

### Phase 5: Compliance & Audit
#### Regulatory Compliance Implementation
1. Configure data access controls:
```sql
-- Set up row-level security
CREATE OR REPLACE ROW ACCESS POLICY rap_sensitive_data
ON mortgage_data.loan_details
GRANT TO ('group:risk_analysts@fanniemae.com')
FILTER USING (
  REGEXP_CONTAINS(current_user(), r'@fanniemae\.com$')
  AND geography IN (
    SELECT allowed_geography 
    FROM access_control.user_permissions 
    WHERE user_email = current_user()
  )
);
```

2. Implement audit logging:
```sql
-- Create audit log table
CREATE OR REPLACE TABLE audit.data_access_logs
(
  timestamp TIMESTAMP,
  user_email STRING,
  action STRING,
  resource STRING,
  status STRING,
  details STRING
);

-- Log access events
CREATE OR REPLACE TRIGGER log_data_access
AFTER SELECT, INSERT, UPDATE, DELETE
ON mortgage_data.loan_details
BEGIN
  INSERT INTO audit.data_access_logs
  VALUES(
    CURRENT_TIMESTAMP(),
    current_user(),
    TG_OP,
    TG_TABLE_NAME,
    'SUCCESS',
    TO_JSON(NEW)
  );
END;
```

## Security & Compliance Measures
- All data transfers encrypted using TLS 1.3
- PII data encrypted at rest using BigQuery default encryption
- Access controls implemented at dataset and table levels
- Audit logging enabled for all data access and modifications
- Regular compliance reviews and access recertification
- Data retention policies aligned with regulatory requirements

## Monitoring & Maintenance
1. Set up BigQuery monitoring:
```sql
-- Monitor query performance
CREATE OR REPLACE TABLE monitoring.query_stats
AS
SELECT
  creation_time,
  user_email,
  query,
  total_bytes_processed,
  total_slot_ms,
  status
FROM `region-us`.INFORMATION_SCHEMA.JOBS_BY_PROJECT
WHERE creation_time > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 24 HOUR);
```

2. Configure alerting thresholds:
- Query cost thresholds
- Processing time limits
- Error rate monitoring
- Data quality metrics

## Disaster Recovery
1. Configure cross-region backup:
```sql
-- Set up snapshot schedule
CREATE SNAPSHOT TABLE mortgage_data.loan_details_backup
  CLONE mortgage_data.loan_details
  FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
  OPTIONS(
    expiration_timestamp = TIMESTAMP_ADD(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
  );
```

2. Implement recovery procedures:
- Regular backup validation
- Recovery time objective (RTO) monitoring
- Recovery point objective (RPO) compliance checks
