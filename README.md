# Retail Sales Data Cleaning & Exploratory Analysis (Synthetic Data)

## Project Objective

The primary goal of this project is to practice and demonstrate data cleaning skills using a realistic retail dataset. 

## Why Synthetic Data?

I chose to generate synthetic data for this project because:

- **Realistic Practice**: Most datasets on platforms like Kaggle are already pre-cleaned, which doesn't reflect the messy data that data analysts encounter in real business environments
- **Controlled Learning**: Synthetic data allows me to intentionally introduce various data quality issues (missing values, duplicates, inconsistent formats, typos, etc.) to practice systematic cleaning approaches
- **Complete Control**: I can simulate real-world data problems without the constraints of working with someone else's cleaned dataset

This approach provides hands-on experience with the type of data quality challenges that are common in actual business scenarios,
I asked ChatGPT to generate a code that generates the data I used for this project without seeing the code, so I don't know what the problems are.

## Analysis Questions

After cleaning the data, the following key business questions will be explored:

1. **Data Quality Assessment**: Evaluate data quality issues and document cleaning processes across customer and transaction records
2. **Customer Acquisition Trends**: Analyze when customers typically join the store to uncover seasonal or temporal acquisition patterns
3. **Payment Method Preferences**: Identify the most popular payment methods used by customers and analyze usage patterns
4. **Store Performance Analysis**: Explore whether store size influences total sales volume or average transaction amounts
5. **Customer Segmentation**: Segment customers based on spending behavior and identify high-value customers for targeted marketing strategies

---

This document outlines the comprehensive data cleaning process applied to the retail store dataset, which consists of three main tables: transactions, customers, and stores.

## Dataset Overview

The merged dataset contains the following features:
- **Transaction data**: TxnID, custID, STORE_id, TxnDate, Amount, PaymentMethod
- **Customer data**: Customer Name, PreferredStore, join_date, Phone #
- **Store data**: Size, City, Opening

## Data Cleaning Steps

### 1. Transaction Data Cleaning (`monthly_transactions_df`)

#### Column Name Standardization
```python
monthly_transactions_df.columns = monthly_transactions_df.columns.str.strip()
```
- Removed leading/trailing whitespace from column names to prevent KeyError issues

#### Payment Method Standardization
```python
def clean_payment(val):
    patterns = [
        (re.compile(r'mobile'), 'mobile'),
        (re.compile(r'(credit|card|cc)'), 'credit_card'),
        (re.compile(r'cash'), 'cash'),
        (re.compile(r'unknown'), np.nan)
    ]
```
- **Purpose**: Standardize inconsistent payment method entries
- **Process**: Used regex patterns to categorize similar payment types
- **Result**: Consolidated variations like "credit", "card", "cc" into "credit_card"
- **Missing Data**: Converted "unknown" entries to NaN for proper handling

#### Data Type Conversions
- **Transaction Date**: Converted to datetime using mixed format parsing
- **Amount**: Cleaned by removing non-numeric characters (except digits, dots, and minus signs)
- **ID Fields**: Converted TxnID, custID, and STORE_id to numeric, using 'Int64' to handle potential NaN values

#### Duplicate Removal
- Applied `drop_duplicates()` to remove any duplicate transaction records

### 2. Customer Data Cleaning (`customers_by_store_df`)

#### Phone Number Standardization
The phone number cleaning process involved multiple sophisticated steps:

**Step 1: Initial Preparation**
```python
customers_by_store_df['Phone #'] = customers_by_store_df['Phone #'].fillna('').astype(str)
```
- Converted all entries to strings and filled NaN values with empty strings

**Step 2: Digit Extraction**
```python
customers_by_store_df['Phone #'] = customers_by_store_df['Phone #'].str.replace(r'[^0-9]', '', regex=True)
```
- Extracted only numeric digits from phone entries
- Removed all formatting characters, letters, and special symbols

**Step 3: Validation and Filtering**
```python
def validate_and_finalize_phone_number(cleaned_digits):
    if pd.isna(cleaned_digits) or not isinstance(cleaned_digits, str):
        return np.nan
    if cleaned_digits == '':
        return np.nan
    if not (MIN_DIGITS <= len(cleaned_digits) <= MAX_DIGITS):
        return np.nan
    return cleaned_digits
```
- **Length Validation**: Only accepted phone numbers with 7-15 digits
- **Content Validation**: Rejected entries that became empty strings after digit extraction
- **Result**: Invalid entries like "SHORT", "N/A", "NO PHONE" were converted to NaN

**Step 4: Formatting**
```python
def format_phone_number(phone_num):
    if len(phone_num_str) == 10:  # US style
        return f"({phone_num_str[0:3]}) {phone_num_str[3:6]}-{phone_num_str[6:]}"
    elif len(phone_num_str) == 11 and phone_num_str.startswith('01'):  # Egyptian mobile style
        return f"{phone_num_str[0:3]}-{phone_num_str[3:7]}-{phone_num_str[7:]}"
```
- Applied standardized formatting based on phone number patterns
- US numbers: (XXX) XXX-XXXX format
- Egyptian mobile numbers: XXX-XXXX-XXXX format

#### Customer Name Cleaning
```python
customers_by_store_df['Customer Name'] = customers_by_store_df['Customer Name'].apply(
    lambda x: re.sub(r"[^A-Za-z\s]", '', x) if isinstance(x, str) else x
).str.strip().str.title()
```
- **Character Filtering**: Removed all non-alphabetic characters except spaces
- **Formatting**: Applied title case for consistent name presentation
- **Whitespace**: Removed leading/trailing spaces

#### Date Conversion
- Converted `join_date` to datetime format using ISO standard ('%Y-%m-%d')

### 3. Store Data Cleaning (`stores_by_city_df`)

#### Size Standardization
```python
stores_by_city_df['Size'] = stores_by_city_df['Size'].str.title()
```
- Applied title case to store size categories for consistency

#### City Name Standardization
```python
city_mapping_title = {
    'Alex.': 'Alexandria',
    'Alexandira': 'Alexandria',
    'Al-Iskandariyah': 'Alexandria',
    'Cai': 'Cairo',
    'Al-Qahira': 'Cairo',
    'Cario': 'Cairo',
    'Gizeh': 'Giza',
    'Gizah': 'Giza',
    'Jizah': 'Giza',
}
```
- **Purpose**: Standardized city name variations and abbreviations
- **Process**: Created a mapping dictionary to handle common misspellings and variations
- **Result**: Consolidated multiple representations of the same city into standard names

#### Date Processing
- Converted store `Opening` dates to a datetime format

#### Column Management
- Attempted to remove the unnecessary `StoreName` column if present
- Used try-except block to handle cases where the column might not exist

### 4. Data Quality Assessment

#### Orphan Record Detection
```python
orphan_r_cutomer_df = pd.merge(monthly_transactions_df, customers_by_store_df, how='left')
orphan_r_store_df = pd.merge(monthly_transactions_df, stores_by_city_df, how='left')
cust_orphan = orphan_r_cutomer_df[orphan_r_cutomer_df['join_date'].isna()]['custID'].values
store_orphan = orphan_r_store_df[orphan_r_store_df['Size'].isna()]['custID'].values
```

**Purpose**: Identify transactions that lack corresponding customer or store records

**Analysis Results**:
- Found orphan records in both customer and store relationships
- Discovered that the same transaction records were orphaned in both tables
- **Decision**: These orphan records will be excluded from analysis to maintain data integrity

## Key Achievements

1. **Standardized Data Types**: Ensured all numeric fields are properly typed and date fields are datetime objects
2. **Cleaned Text Data**: Removed special characters and standardized formatting
3. **Handled Missing Data**: Appropriately converted invalid entries to NaN
4. **Geographic Consistency**: Standardized city names across Egyptian locations
5. **Data Integrity**: Identified and planned the exclusion of orphan records
6. **Phone Number Validation**: Implemented robust validation for international phone formats

## Data Quality Metrics

- **Phone Numbers**: Validated length (7-15 digits) and applied region-specific formatting
- **Payment Methods**: Consolidated into 4 main categories: mobile, credit_card, cash, and other
- **Geographic Data**: Standardized to 3 main cities: Alexandria, Cairo, and Giza
- **Orphan Records**: Identified matching orphan records across tables for exclusion

This cleaning process ensures the dataset is ready for reliable analysis and visualization while maintaining data quality and consistency across all tables.
