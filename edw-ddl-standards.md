# Enterprise Data Warehouse DDL Standards & Conventions Analysis

## Executive Summary
This document analyzes the current DDL conventions used in the Enterprise Data Warehouse and proposes standards for consistent database object naming, structure, and implementation.

## Current Conventions Observed

### 1. Naming Conventions

#### Tables
- **Prefix Standards**: Tables use descriptive prefixes to indicate their purpose:
  - `D_` - Dimension tables (e.g., `D_CUSTOMER`, `D_PRODUCT`)
  - `F_` - Fact tables (e.g., `F_ORDER`, `F_ORDER_LINE`)
  - `B_` - Bridge tables (e.g., `B_RATING_FEEDBACK`)
  - `RPT_` - Reporting/transient tables (e.g., `RPT_DATA_RIGHTS_REQUESTS`)
  - `SRC_` - Source/staging tables (e.g., `SRC_SHOPPER_HIRE_DATE`)
  - `STG_` - Staging tables (e.g., `STG_BUNDLE_ORDER_STATUS_COUNTS_TLMD`)
  - `VW_` - Views (e.g., `VW_CUSTOMER_SEGMENTATION`)

#### Columns
- **UPPER_SNAKE_CASE** is consistently used
- Common suffixes include:
  - `_ID` for identifiers
  - `_SK` for surrogate keys
  - `_NK` for natural keys
  - `_AT` for timestamps
  - `_DATE` for dates
  - `_FLAG` for boolean indicators
  - `_COUNT`, `_AMT`, `_NUM` for numeric measures

### 2. Data Type Standards

#### Common Patterns
- **VARCHAR(16777216)** - Default for text fields (Snowflake maximum)
- **NUMBER(38,0)** - Standard for integer IDs
- **TIMESTAMP_TZ(9)** - Standard for timestamps with timezone
- **BOOLEAN** - For flags/indicators
- **FLOAT** - For monetary amounts and measures
- **BINARY(64)** or **BINARY(8388608)** - For hash values

### 3. Metadata & Governance

#### Audit Columns (Commonly Present)
- `EDW_CREATED_AT` - Record creation timestamp
- `EDW_UPDATED_AT` - Last update timestamp
- `EDW_PROCESSED_AT` - ETL processing timestamp
- `BEGIN_DATE` / `END_DATE` - SCD Type 2 validity period
- `CURRENT_RECORD_FLAG` - Active record indicator
- `HASH_CHECK_VALUE` - Data integrity hash

#### Data Privacy & Masking
- Consistent use of masking policies for PII:
  - `NAME_MASK` for names
  - `EMAIL_MASK` for emails
  - `PHONE_MASK` for phone numbers
  - `ADDRESS_MASK` for addresses
  - Privacy category tags (e.g., `SNOWFLAKE.CORE.PRIVACY_CATEGORY`)

## Recommended Standards

### 1. Table Naming Standards

| Prefix | Usage | Example |
|--------|-------|---------|
| `D_` | Dimension tables | `D_CUSTOMER` |
| `F_` | Fact tables | `F_ORDER` |
| `B_` | Bridge/junction tables | `B_RATING_FEEDBACK` |
| `A_` | Aggregate tables | `A_DAILY_SALES` |
| `RPT_` | Reporting tables | `RPT_MONTHLY_REVENUE` |
| `SRC_` | Source system extracts | `SRC_SHOPIFY_ORDERS` |
| `STG_` | Staging tables | `STG_ORDER_PROCESSING` |
| `REF_` | Reference/lookup tables | `REF_COUNTRY_CODES` |
| `HIST_` | Historical/archive tables | `HIST_ORDER_2023` |

### 2. Column Naming Standards

#### Required Metadata Columns
Every table should include:
```sql
EDW_CREATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
EDW_UPDATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
EDW_PROCESSED_AT TIMESTAMP_TZ(9)
```

#### SCD Type 2 Tables Additional Requirements
```sql
BEGIN_DATE TIMESTAMP_TZ(9) NOT NULL,
END_DATE TIMESTAMP_TZ(9),
CURRENT_RECORD_FLAG BOOLEAN NOT NULL,
HASH_CHECK_VALUE BINARY(64)
```

### 3. Key Standards

#### Primary Keys
- Dimension tables: Use `_SK` suffix for surrogate keys
- Fact tables: Composite keys or unique transaction IDs
- All tables MUST have a defined primary key

#### Foreign Keys
- Name pattern: `FK_{child_table}_{parent_table}`
- Must be defined but can be `NOT ENFORCED` for performance
- Example: `FK_F_ORDER_D_CUSTOMER`

### 4. Data Type Guidelines

| Data Element | Recommended Type | Notes |
|--------------|------------------|-------|
| IDs (Natural) | VARCHAR(100) | Accommodate external systems |
| IDs (Surrogate) | NUMBER(38,0) | Auto-increment where possible |
| Names/Descriptions | VARCHAR(500) | Limit unless justified |
| Long Text | VARCHAR(16777216) | Only when necessary |
| Amounts/Money | NUMBER(18,2) | 2 decimal precision |
| Percentages | NUMBER(5,2) | 0-100 with decimals |
| Counts | NUMBER(18,0) | Integer only |
| Dates | DATE | Without time component |
| Timestamps | TIMESTAMP_TZ(9) | Always with timezone |
| Flags | BOOLEAN | Prefer over CHAR(1) |

### 5. View Standards

- Views should be prefixed with `VW_`
- Materialized views should be prefixed with `MV_`
- Views should include comments explaining business logic
- Complex views should reference base tables, not other views

### 6. Documentation Requirements

#### Table Comments
```sql
COMMENT ON TABLE table_name IS 'Business purpose and data refresh frequency';
```

#### Column Comments
```sql
COMMENT ON COLUMN table_name.column_name IS 'Business definition and valid values';
```

### 7. Security & Privacy Standards

#### PII Columns Must Have:
1. Appropriate masking policy applied
2. Privacy category tags
3. Semantic category tags

Example:
```sql
CUSTOMER_EMAIL VARCHAR(500) 
  WITH MASKING POLICY EMAIL_MASK 
  WITH TAG (
    SNOWFLAKE.CORE.PRIVACY_CATEGORY='IDENTIFIER',
    SNOWFLAKE.CORE.SEMANTIC_CATEGORY='EMAIL'
  )
```

## Identified Gaps & Recommendations

### Current Gaps

1. **Inconsistent VARCHAR lengths** - Some use 16777216 unnecessarily
2. **Missing comments** - Many tables/columns lack business descriptions
3. **Inconsistent key naming** - Some FKs don't follow naming patterns
4. **Mixed case conventions** - Some objects use different cases
5. **Missing primary keys** - Not all tables have explicit PKs defined

### Recommended Actions

1. **Standardize VARCHAR lengths** - Create standard sizes (100, 500, 4000, MAX)
2. **Implement comment requirements** - All objects need business descriptions
3. **Enforce key naming patterns** - Automated checks for compliance
4. **Create naming validation** - Pre-deployment checks for standards
5. **Mandatory audit columns** - Template for all new tables
6. **Version control integration** - Track DDL changes systematically

## Implementation Guidelines

### New Object Checklist

- [ ] Follows naming convention
- [ ] Has appropriate prefix
- [ ] Includes required audit columns
- [ ] Has primary key defined
- [ ] Has foreign keys defined (if applicable)
- [ ] Includes table comment
- [ ] Includes column comments for business fields
- [ ] PII fields have masking policies
- [ ] PII fields have privacy tags
- [ ] Data types follow standards
- [ ] Reviewed for performance implications

### Governance Process

1. All DDL must be reviewed before implementation
2. Use provided templates for common patterns
3. Document exceptions with business justification
4. Regular audits for compliance
5. Maintain this document as living standard

## Appendix A: Template DDL

### Dimension Table Template
```sql
CREATE OR REPLACE TABLE D_[ENTITY_NAME] (
    -- Surrogate Key
    [ENTITY_NAME]_SK NUMBER(38,0) AUTOINCREMENT PRIMARY KEY,
    
    -- Natural/Business Key
    [ENTITY_NAME]_ID VARCHAR(100) NOT NULL,
    
    -- Attributes
    [ENTITY_NAME]_NAME VARCHAR(500),
    [ENTITY_NAME]_DESCRIPTION VARCHAR(4000),
    
    -- SCD Type 2 Columns
    BEGIN_DATE TIMESTAMP_TZ(9) NOT NULL,
    END_DATE TIMESTAMP_TZ(9),
    CURRENT_RECORD_FLAG BOOLEAN NOT NULL DEFAULT TRUE,
    HASH_CHECK_VALUE BINARY(64),
    
    -- Audit Columns
    SOURCE_CREATED_AT TIMESTAMP_TZ(9),
    SOURCE_UPDATED_AT TIMESTAMP_TZ(9),
    EDW_CREATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
    EDW_UPDATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
    EDW_PROCESSED_AT TIMESTAMP_TZ(9)
) COMMENT = 'Dimension table for [business entity description]';
```

### Fact Table Template
```sql
CREATE OR REPLACE TABLE F_[PROCESS_NAME] (
    -- Foreign Keys
    [DIMENSION_NAME]_SK NUMBER(38,0) NOT NULL,
    DATE_SK NUMBER(38,0) NOT NULL,
    
    -- Degenerate Dimensions
    [TRANSACTION_ID] VARCHAR(100),
    
    -- Measures
    [MEASURE_NAME]_AMT NUMBER(18,2),
    [MEASURE_NAME]_COUNT NUMBER(18,0),
    
    -- Audit Columns
    EDW_CREATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
    EDW_UPDATED_AT TIMESTAMP_TZ(9) DEFAULT CURRENT_TIMESTAMP(),
    EDW_PROCESSED_AT TIMESTAMP_TZ(9),
    
    -- Keys
    PRIMARY KEY ([DIMENSION_NAME]_SK, DATE_SK, [TRANSACTION_ID]),
    CONSTRAINT FK_F_[PROCESS_NAME]_D_[DIMENSION_NAME] 
        FOREIGN KEY ([DIMENSION_NAME]_SK) 
        REFERENCES D_[DIMENSION_NAME]([DIMENSION_NAME]_SK) NOT ENFORCED,
    CONSTRAINT FK_F_[PROCESS_NAME]_D_DATE 
        FOREIGN KEY (DATE_SK) 
        REFERENCES D_DATE(DATE_SK) NOT ENFORCED
) COMMENT = 'Fact table capturing [business process description]';
```

## Appendix B: Common Abbreviations

| Abbreviation | Meaning |
|--------------|---------|
| AMT | Amount |
| AVG | Average |
| CNT/COUNT | Count |
| CURR | Current |
| CUST | Customer |
| DESC | Description |
| DT/DATE | Date |
| ID | Identifier |
| MAX | Maximum |
| MIN | Minimum |
| NK | Natural Key |
| NUM | Number |
| PCT | Percentage |
| PREV | Previous |
| QTY | Quantity |
| REF | Reference |
| SK | Surrogate Key |
| SRC | Source |
| STG | Staging |
| TGT | Target |
| TS | Timestamp |
| TXN | Transaction |
| VAL | Value |

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2024-01-01 | Data Architecture Team | Initial version |

---
*This document should be reviewed quarterly and updated as needed to reflect evolving best practices and organizational requirements.*