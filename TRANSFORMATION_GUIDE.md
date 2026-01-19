# Transformation Guide (1–17)

Current date: 2026-01-19

This document describes 17 standard data transformations used in our pipelines. For each transformation you'll find:
- Purpose
- YAML config example
- Sample INPUT and OUTPUT
- Note on row/column impact

---

## 1) TRIM STRINGS
Purpose: Removes leading and trailing whitespace from string columns.

YAML Config:
```yaml
trim_strings:
  enabled: true
  columns: [first_name, last_name]
```

INPUT:
+------------+-----------+
| first_name | last_name |
+------------+-----------+
|   John     | Doe       |
|  Alice     |   Smith   |
+------------+-----------+

OUTPUT:
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| John       | Doe       |
| Alice      | Smith     |
+------------+-----------+

Note: Row count unchanged.

---

## 2) REGEX CLEAN
Purpose: Cleans data using regular expressions — removes unwanted characters from numbers, standardizes emails.

YAML Config:
```yaml
regex_clean:
  enabled: true
  number_columns:
    - name: phone_raw
      out: phone_clean
      pattern_remove: "[^0-9]"
      trim: true
  email_columns:
    - name: email_raw
      out: email
      trim: true
      lowercase: true
```

INPUT:
+------------------+----------------------+
| phone_raw        | email_raw            |
+------------------+----------------------+
| (555) 123-4567   |   JOHN@GMAIL.COM     |
| +1-800-555-0199  | Alice@Yahoo.COM      |
+------------------+----------------------+

OUTPUT:
+------------------+----------------------+-------------+------------------+
| phone_raw        | email_raw            | phone_clean | email            |
+------------------+----------------------+-------------+------------------+
| (555) 123-4567   |   JOHN@GMAIL.COM     | 5551234567  | john@gmail.com   |
| +1-800-555-0199  | Alice@Yahoo.COM      | 18005550199 | alice@yahoo.com  |
+------------------+----------------------+-------------+------------------+

Note: Row count unchanged; cleaned columns added.

---

## 3) PARSE NUMERIC
Purpose: Converts string columns to numeric types (int, double, etc.) after cleaning non-numeric characters.

YAML Config:
```yaml
parse_numeric:
  enabled: true
  columns:
    - name: balance_raw
      out: balance
      cast_type: double
```

INPUT:
+--------------+
| balance_raw  |
+--------------+
| $1,234.56    |
| -500.00 USD  |
| 10000        |
+--------------+

OUTPUT:
+--------------+---------+
| balance_raw  | balance |
+--------------+---------+
| $1,234.56    | 1234.56 |
| -500.00 USD  | -500.0  |
| 10000        | 10000.0 |
+--------------+---------+

Note: Row count unchanged; numeric column added/converted.

---

## 4) PARSE DATE
Purpose: Parses date strings into a canonical date format, trying multiple formats until one matches.

YAML Config:
```yaml
parse_date:
  enabled: true
  columns:
    - name: date_raw
      out: date_clean
      formats: ["yyyy-MM-dd", "dd/MM/yyyy", "MM-dd-yyyy"]
```

INPUT:
+--------------+
| date_raw     |
+--------------+
| 2024-01-15   |
| 25/12/2023   |
| 07-04-2024   |
+--------------+

OUTPUT:
+--------------+------------+
| date_raw     | date_clean |
+--------------+------------+
| 2024-01-15   | 2024-01-15 |
| 25/12/2023   | 2023-12-25 |
| 07-04-2024   | 2024-07-04 |
+--------------+------------+

Note: Row count unchanged; parsed date column added.

---

## 5) NORMALIZE STATUS
Purpose: Maps various status values to standardized values using a lookup mapping.

YAML Config:
```yaml
normalize_status:
  enabled: true
  columns:
    - name: status_raw
      out: status
      mapping:
        a: "Active"
        active: "Active"
        c: "Closed"
        closed: "Closed"
```

INPUT:
+------------+
| status_raw |
+------------+
| a          |
| Active     |
| ACTIVE     |
| c          |
| Closed     |
+------------+

OUTPUT:
+------------+--------+
| status_raw | status |
+------------+--------+
| a          | Active |
| Active     | Active |
| ACTIVE     | Active |
| c          | Closed |
| Closed     | Closed |
+------------+--------+

Note: Row count unchanged; normalized column added.

---

## 6) NULL HANDLING
Purpose: Drops rows where specified columns have NULL values.

YAML Config:
```yaml
null_handling:
  enabled: true
  drop_if_null: [customer_id, email]
```

INPUT:
+-------------+-------+------------------+
| customer_id | name  | email            |
+-------------+-------+------------------+
| C001        | John  | john@email.com   |
| NULL        | Alice | alice@email.com  |
| C003        | Bob   | NULL             |
| C004        | Eve   | eve@email.com    |
+-------------+-------+------------------+

OUTPUT:
+-------------+-------+------------------+
| customer_id | name  | email            |
+-------------+-------+------------------+
| C001        | John  | john@email.com   |
| C004        | Eve   | eve@email.com    |
+-------------+-------+------------------+

Note: Row count decreases (rows with NULLs in required columns are dropped).

---

## 7) DUPLICATE HANDLING
Purpose: Removes duplicate rows based on specified unique key columns.

YAML Config:
```yaml
duplicate_handling:
  enabled: true
  unique_keys: [customer_id]
  # optional: keep: "first" | "last"
```

INPUT:
+-------------+-------+------------+
| customer_id | name  | updated_at |
+-------------+-------+------------+
| C001        | John  | 2024-01-01 |
| C001        | John  | 2024-01-02 |
| C002        | Alice | 2024-01-01 |
| C002        | Alice | 2024-01-03 |
+-------------+-------+------------+

OUTPUT:
+-------------+-------+------------+
| customer_id | name  | updated_at |
+-------------+-------+------------+
| C001        | John  | 2024-01-01 |
| C002        | Alice | 2024-01-01 |
+-------------+-------+------------+

Note: Row count decreases (duplicates removed).

---

## 8) RENAME COLUMNS
Purpose: Renames columns from old names to new names.

YAML Config:
```yaml
rename_columns:
  enabled: true
  mapping:
    cust_id: customer_id
    amt: amount
    txn_dt: transaction_date
```

INPUT:
+---------+-------+------------+
| cust_id | amt   | txn_dt     |
+---------+-------+------------+
| C001    | 100.0 | 2024-01-15 |
+---------+-------+------------+

OUTPUT:
+-------------+--------+------------------+
| customer_id | amount | transaction_date |
+-------------+--------+------------------+
| C001        | 100.0  | 2024-01-15       |
+-------------+--------+------------------+

Note: Column names changed; row count unchanged.

---

## 9) STANDARDIZE COLUMN CASE
Purpose: Converts all column names to a consistent format (snake_case, lower, or upper).

YAML Config:
```yaml
standardize_column_case:
  enabled: true
  format: snake_case   # options: snake_case | lower | upper
```

INPUT:
+------------+-----------+-------------+
| FirstName  | Last Name | EmailAddr   |
+------------+-----------+-------------+
| John       | Doe       | john@x.com  |
+------------+-----------+-------------+

OUTPUT:
+------------+-----------+-------------+
| first_name | last_name | email_addr  |
+------------+-----------+-------------+
| John       | Doe       | john@x.com  |
+------------+-----------+-------------+

Note: Column names changed; row content unchanged.

---

## 10) FILL MISSING
Purpose: Fills NULL values with specified default values.

YAML Config:
```yaml
fill_missing:
  enabled: true
  columns:
    - name: country
      value: "Unknown"
    - name: age
      value: 0
```

INPUT:
+-------+---------+------+
| name  | country | age  |
+-------+---------+------+
| John  | USA     | 30   |
| Alice | NULL    | NULL |
| Bob   | UK      | NULL |
+-------+---------+------+

OUTPUT:
+-------+---------+------+
| name  | country | age  |
+-------+---------+------+
| John  | USA     | 30   |
| Alice | Unknown | 0    |
| Bob   | UK      | 0    |
+-------+---------+------+

Note: Row count unchanged; missing values filled.

---

## 11) SURROGATE KEY
Purpose: Generates a unique UUID for each row as a surrogate key.

YAML Config:
```yaml
surrogate_key:
  enabled: true
  column: customer_sk
  method: uuid
```

INPUT:
+-------------+-------+
| customer_id | name  |
+-------------+-------+
| C001        | John  |
| C002        | Alice |
+-------------+-------+

OUTPUT:
+-------------+-------+--------------------------------------+
| customer_id | name  | customer_sk                          |
+-------------+-------+--------------------------------------+
| C001        | John  | 550e8400-e29b-41d4-a716-446655440000 |
| C002        | Alice | 6ba7b810-9dad-11d1-80b4-00c04fd430c8 |
+-------------+-------+--------------------------------------+

Note: Row count unchanged; surrogate key column added.

---

## 12) OUTLIER CAPPING
Purpose: Caps numeric values within specified min/max bounds.

YAML Config:
```yaml
outlier_capping:
  enabled: true
  columns:
    - name: balance
      min: 0
      max: 100000
```

INPUT:
+-------------+----------+
| customer_id | balance  |
+-------------+----------+
| C001        | 5000     |
| C002        | -500     |
| C003        | 150000   |
| C004        | 75000    |
+-------------+----------+

OUTPUT:
+-------------+----------+
| customer_id | balance  |
+-------------+----------+
| C001        | 5000     |
| C002        | 0        |
| C003        | 100000   |
| C004        | 75000    |
+-------------+----------+

Note: Row count unchanged; values adjusted to bounds.

---

## 13) FUZZY NORMALIZE
Purpose: Normalizes text by removing punctuation, collapsing spaces, and converting to lowercase.

YAML Config:
```yaml
fuzzy_normalize:
  enabled: true
  rules:
    remove_punctuation: true
    collapse_spaces: true
    lowercase: true
  columns:
    - name: company_name
      out: company_clean
```

INPUT:
+---------------------------+
| company_name              |
+---------------------------+
| Apple,  Inc.              |
| GOOGLE   LLC!!!           |
| Microsoft   Corporation.  |
+---------------------------+

OUTPUT:
+---------------------------+------------------------+
| company_name              | company_clean          |
+---------------------------+------------------------+
| Apple,  Inc.              | apple inc              |
| GOOGLE   LLC!!!           | google llc             |
| Microsoft   Corporation.  | microsoft corporation  |
+---------------------------+------------------------+

Note: Row count unchanged; normalized text added.

---

## 14) FLATTEN JSON
Purpose: Flattens nested JSON/struct fields into top-level columns and explodes arrays into separate rows.

YAML Config:
```yaml
flatten_json:
  enabled: true
```

INPUT:
+------+---------------------------+----------------+
| id   | address                   | tags           |
+------+---------------------------+----------------+
| 1    | {city: NYC, zip: 10001}   | [a, b]         |
| 2    | {city: LA, zip: 90001}    | [x]            |
+------+---------------------------+----------------+

OUTPUT (after flattening struct and exploding array):
+------+--------------+--------------+------+
| id   | address_city | address_zip  | tags |
+------+--------------+--------------+------+
| 1    | NYC          | 10001        | a    |
| 1    | NYC          | 10001        | b    |
| 2    | LA           | 90001        | x    |
+------+--------------+--------------+------+

Note: Row count may increase due to exploding arrays; nested fields become columns.

---

## 15) SCHEMA VALIDATION
Purpose: Validates that required columns exist in the DataFrame. Raises an error if missing.

YAML Config:
```yaml
schema_validation:
  required_columns: [customer_id, email, created_at]
```

INPUT DataFrame columns: [customer_id, email, name]

OUTPUT:
❌ Error: Missing required columns: ['created_at']

Note: Pipeline should stop / raise an error if required columns are absent.

---

## 16) JSON KEY DEFAULT
Purpose: Adds missing columns with default values (useful when JSON keys might be absent).

YAML Config:
```yaml
json_key_default:
  columns:
    - name: country
      default: "Unknown"
    - name: verified
      default: false
```

INPUT:
+-------------+-------+
| customer_id | name  |
+-------------+-------+
| C001        | John  |
+-------------+-------+

OUTPUT:
+-------------+-------+---------+----------+
| customer_id | name  | country | verified |
+-------------+-------+---------+----------+
| C001        | John  | Unknown | false    |
+-------------+-------+---------+----------+

Note: Row count unchanged; missing columns added.

---

## 17) ARRAY SIZE VALIDATION
Purpose: Replaces empty arrays with NULL (useful before explode operations to avoid empty results).

YAML Config:
```yaml
array_size_validation:
  columns: [tags, categories]
```

INPUT:
+------+----------+------------+
| id   | tags     | categories |
+------+----------+------------+
| 1    | [a, b]   | []         |
| 2    | []       | [x, y]     |
| 3    | [c]      | [z]        |
+------+----------+------------+

OUTPUT:
+------+----------+------------+
| id   | tags     | categories |
+------+----------+------------+
| 1    | [a, b]   | NULL       |
| 2    | NULL     | [x, y]     |
| 3    | [c]      | [z]        |
+------+----------+------------+

Note: Row count unchanged; empty arrays replaced with NULL.

---

# Summary Table

| #  | Transformation            | Purpose                                     | Row Count Change               |
|----|---------------------------|---------------------------------------------|--------------------------------|
| 1  | trim_strings              | Remove leading/trailing whitespace          | Same                           |
| 2  | regex_clean               | Clean using regex (phones, emails)          | Same                           |
| 3  | parse_numeric             | Convert strings to numeric                  | Same                           |
| 4  | parse_date                | Parse date strings                          | Same                           |
| 5  | normalize_status          | Map statuses to standard values             | Same                           |
| 6  | null_handling             | Drop rows with NULLs in required columns    | Decreases                      |
| 7  | duplicate_handling        | Remove duplicate rows                       | Decreases                      |
| 8  | rename_columns            | Rename columns                              | Same                           |
| 9  | standardize_column_case   | Make column names snake_case/lowercase      | Same                           |
| 10 | fill_missing              | Fill NULLs with defaults                    | Same                           |
| 11 | surrogate_key             | Add UUID surrogate key                      | Same (adds column)             |
| 12 | outlier_capping           | Cap numeric values within bounds            | Same                           |
| 13 | fuzzy_normalize           | Normalize text (punctuation/spacing/case)   | Same                           |
| 14 | flatten_json              | Flatten nested fields & explode arrays      | May increase (explode)         |
| 15 | schema_validation         | Validate required columns exist             | Same (or Error stops pipeline) |
| 16 | json_key_default          | Add missing keys with defaults              | Same (adds columns)            |
| 17 | array_size_validation     | Replace empty arrays with NULL              | Same                           |

---

What this file is and next steps
- This Markdown file is a ready-to-add documentation artifact for your repository.
- Next I can:
  - generate runnable code (PySpark, Scala Spark, or Pandas) for any transformation or the full pipeline,
  - produce a combined YAML pipeline that chains selected transformations,
  - add unit tests or sample datasets.

Tell me which format (PySpark, Pandas, Scala) or which transformations you want implemented next — I will produce code and tests ready to add to the repo.