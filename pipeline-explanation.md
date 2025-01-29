# Understanding the Mortgage Default Risk Analytics Pipeline

## Phase 1: Data Ingestion & Validation
### What it's trying to achieve:
This phase is like setting up a careful filtering system for all incoming data. Think of it as the quality control checkpoint where we gather all the necessary mortgage information and make sure it's accurate and complete before moving forward.

### How it works:
- Collects data from multiple sources including:
  - Past mortgage performance records
  - Current loan information
  - Credit reports
  - Property values
  - Economic data (like unemployment rates, housing market trends)
  - Payment records

- BigQuery Data Transfer API automatically moves this data into the system on a regular schedule (like every 6 hours)
- Runs automatic checks to catch problems like:
  - Missing information
  - Invalid numbers (like negative loan amounts)
  - Inconsistent dates
  - Duplicate records
- Flags any issues for review
- Keeps detailed records of where data came from and any changes made

## Phase 2: Data Transformation & Enrichment
### What it's trying to achieve:
This phase is like a data kitchen where raw ingredients (basic data) are combined and processed into more meaningful information that can help predict mortgage defaults.

### How it works:
- Creates new useful data points by combining existing information, such as:
  - Calculating how many times a borrower has been late on payments
  - Working out the current loan-to-value ratio
  - Combining property value with local market trends
  - Creating risk indicators based on multiple factors
- Standardizes all information into a consistent format
- Uses BigQuery to perform these calculations efficiently
- Keeps track of how each new data point was created
- Stores different versions of the data for historical analysis

## Phase 3: Model Development & Training
### What it's trying to achieve:
This phase is like teaching a smart system to recognize patterns that might indicate a mortgage is at risk of default, based on all the processed data from the previous phase.

### How it works:
- Uses BigQuery ML to:
  - Build statistical models that can predict default risk
  - Test these models on historical data to see how accurate they are
  - Continuously improve predictions as new data comes in
- Compares different prediction methods to find the most accurate one
- Keeps track of how well each model performs
- Documents why certain factors are important in predicting defaults
- Ensures models are fair and unbiased across different borrower groups

## Phase 4: Risk Scoring & Reporting
### What it's trying to achieve:
This phase puts the trained models to work, analyzing current mortgages to identify potential risks and creating clear reports for decision makers.

### How it works:
- Regularly analyzes all active mortgages to:
  - Assign risk scores to each loan
  - Identify mortgages that might need attention
  - Group loans by risk level
- Creates automated reports showing:
  - Overall portfolio risk levels
  - Trends in default risk
  - Areas or groups that might need special attention
  - Early warning signs of potential problems
- Makes this information easily accessible to authorized users through BigQuery Studio
- Updates risk assessments as new information comes in

## Phase 5: Compliance & Audit
### What it's trying to achieve:
This phase ensures everything is done according to regulations and creates a clear trail of all activities for auditors and regulators.

### How it works:
- Controls who can see and use different types of information:
  - Restricts access to sensitive personal data
  - Tracks who looks at what information
  - Ensures data isn't modified inappropriately
- Maintains detailed records of:
  - All changes to data and models
  - Who made each change and why
  - When and how data was accessed
  - Any unusual activities or security concerns
- Regularly checks that all processes follow regulations
- Creates reports specifically for auditors and regulators
- Ensures data is kept only as long as legally required

## Disaster Recovery and Maintenance
### What it's trying to achieve:
This ongoing process ensures the system stays reliable and can recover quickly from any problems.

### How it works:
- Creates regular backups of all data and systems
- Tests recovery procedures to ensure they work
- Monitors system performance and usage
- Fixes problems before they affect operations
- Updates systems and security regularly
- Has plans ready for different types of emergencies
- Uses BigQuery's built-in tools to:
  - Monitor performance
  - Track costs
  - Identify potential problems
  - Keep systems running efficiently
