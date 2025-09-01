
# Ensuring Data Quality in an ETL System

This document uses an **ETL system that extracts shopping information from emails** as an example to illustrate how to ensure overall data quality.  
The automated data quality system includes: **data acquisition, feature matching, data trend detection, false positive optimization, and AI enhancement**.

---

## Data Flow

## 1. Data Feature Analysis

### 1.1 Email Data Characteristics
- Email data is **unstructured**.  
- The core method for converting unstructured to structured data is **regular expressions**.  
- Typical errors include **missing information** or **incorrectly extracted information**.

### 1.2 Extracted Data
- Stored in **JSON format**.  
- Large volumes of similar data form **big data**, which shows intrinsic properties such as:
  - **Temporal patterns (time series)**  
  - **Field-level trends and statistical regularities**

---

## 2. Strategies

- The goal of data quality management is to **detect erroneous data**.  
- Two complementary strategies:
  - **Forward Strategy (Detection):** Identify suspicious or potentially incorrect information.  
    - Principle: *Better to flag too much than to miss errors.*  
  - **Reverse Strategy (Filtering):** Remove false positives from the forward strategy.  
    - Principle: *Better to miss a few than to flag correct data as errors.*

---

## 3. Methods

### 3.1 Parallel Regular Expressions
Use multiple regex patterns in parallel to increase robustness.

### 3.2 Data Feature Analysis

#### 3.2.1 Text
- Two typical approaches:  
  1. **Regular Expressions**  
  2. **AI-based NLP**

#### 3.2.2 Money
- Anomalies can appear in two forms:  
  1. **Out-of-range values** (too high or too low)  
  2. **Contextual mismatches**, e.g., *total price ≠ unit price × quantity* or inconsistency with related fields

#### 3.2.3 IDs
- IDs from the same merchant often follow **patterns**.  
- Can be modeled by **ID length** or **abstract regex patterns**.

#### 3.2.4 Others
- Other data items should be analyzed based on **business-specific characteristics**, then rules can be derived to describe inherent patterns.

### 3.3 AI Enhancement
- Combine results from **AI processors** to validate data accuracy.  
- Provide richer **contextual information** for better decision-making.

---

## 4. Data Trend Monitoring

### 4.1 Anomaly Trend Indicators
- Use statistical measures such as:
  - Mean  
  - Variance  
  - Covariance coefficient  
  - Approximate line slope  
- These help detect distribution and trend anomalies.

### 4.2 Periodic Data Filtering
- Normalize data first.  
- Use differences in **daily/weekly/monthly covariance coefficients** to detect periodic patterns.

### 4.3 Human-Labeled Filtering
- Leverage **historical human-labeled data**.  
- Evaluate whether new data points **significantly impact overall trends**.

---

## 5. Implementation

### 5.1 System Architecture
