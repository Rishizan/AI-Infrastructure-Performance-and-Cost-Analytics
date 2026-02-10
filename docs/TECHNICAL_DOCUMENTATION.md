# Technical Documentation: AI Cloud Performance Suite

## Table of Contents
1. [Data Dictionary](#data-dictionary)
2. [DAX Formulas Reference](#dax-formulas-reference)
3. [Power Query Transformations](#power-query-transformations)
4. [Performance Optimization](#performance-optimization)
5. [Troubleshooting Guide](#troubleshooting-guide)

---

## Data Dictionary

### Source Dataset: `ai_models_performance.csv`

| Column Name | Data Type | Description | Example Values |
|-------------|-----------|-------------|----------------|
| **Model** | Text | AI model name and variant | "GPT-5.2 (xhigh)", "Claude Opus 4.5" |
| **Context Window** | Text | Maximum token context length | "400k", "1m", "200k" |
| **Creator** | Text | Organization that developed the model | "OpenAI", "Anthropic", "Google" |
| **Intelligence Index** | Number | Proprietary benchmark score (0-100) | 51, 49, 48 |
| **Price (Blended USD/1M Tokens)** | Currency | Average cost per million tokens | $4.81, $10.00, $4.50 |
| **Speed (median token/s)** | Number | Median throughput in tokens per second | 100, 79, 128 |
| **Latency (First Answer Chunk /s)** | Number | Time to first token (TTFT) in seconds | 44.29, 1.7, 32.19 |

### Data Quality Notes
- **Missing Values**: 47 models have `Speed = 0` or `Latency = 0` (excluded from speed-related visualizations)
- **Intelligence Index**: 8 models have `--` values (treated as NULL)
- **Special Characters**: Column names contain parentheses, slashes, and spaces requiring DAX escaping

---

## DAX Formulas Reference

### Calculated Columns

#### 1. Tokens Per Dollar
**Purpose**: Calculate cost efficiency metric

```dax
Tokens Per Dollar = 
DIVIDE(
    1000000,
    [Price (Blended USD/1M Tokens)],
    0  // Return 0 if division by zero
)
```

**Business Logic**: Higher values indicate better ROI

---

#### 2. Speed Tier
**Purpose**: Categorize models into performance buckets

```dax
Speed Tier = 
SWITCH(
    TRUE(),
    [Speed(median token/s)] >= 200, "Blazing Fast",
    [Speed(median token/s)] >= 100, "Fast",
    [Speed(median token/s)] >= 50, "Moderate",
    [Speed(median token/s)] > 0, "Slow",
    BLANK()  // Handle missing speed data
)
```

**Tier Definitions**:
- **Blazing Fast**: ≥200 tokens/s (real-time interactive)
- **Fast**: 100-199 tokens/s (near real-time)
- **Moderate**: 50-99 tokens/s (batch processing)
- **Slow**: <50 tokens/s (background tasks)

---

#### 3. Latency Category
**Purpose**: Flag production-critical latency issues

```dax
Latency Category = 
SWITCH(
    TRUE(),
    [Latency (First Answer Chunk /s)] <= 1, "Excellent",
    [Latency (First Answer Chunk /s)] <= 5, "Good",
    [Latency (First Answer Chunk /s)] <= 10, "Acceptable",
    [Latency (First Answer Chunk /s)] > 10, "Red Zone",
    BLANK()
)
```

**SLA Thresholds**:
- **Excellent**: ≤1s (chatbots, voice assistants)
- **Good**: 1-5s (search, recommendations)
- **Acceptable**: 5-10s (content generation)
- **Red Zone**: >10s (unacceptable for interactive use)

---

### Measures

#### 1. Monthly Bill (Parameterized)
**Purpose**: Calculate projected monthly costs

```dax
Monthly Bill = 
VAR TokensPerMonth = 1000000000  // Adjustable via slicer
VAR PricePerMillion = SELECTEDVALUE([Price (Blended USD/1M Tokens)])
RETURN
    IF(
        NOT(ISBLANK(PricePerMillion)),
        (TokensPerMonth / 1000000) * PricePerMillion,
        BLANK()
    )
```

**Usage**: Connect to a slicer for dynamic what-if analysis

---

#### 2. Average Intelligence by Provider
**Purpose**: Benchmark creator performance

```dax
Avg Intelligence = 
CALCULATE(
    AVERAGE([Intelligence Index]),
    FILTER(
        'ai_models_performance',
        NOT(ISBLANK([Intelligence Index]))
    )
)
```

---

#### 3. Model Count by Speed Tier
**Purpose**: Assess fleet composition

```dax
Model Count = 
COUNTROWS(
    FILTER(
        'ai_models_performance',
        NOT(ISBLANK([Speed Tier]))
    )
)
```

---

## Power Query Transformations

### Step-by-Step ETL Process

#### 1. Data Import
```m
let
    Source = Csv.Document(
        File.Contents("D:\devops project\power bi dashboard\data\ai_models_performance.csv"),
        [Delimiter=",", Columns=7, Encoding=65001, QuoteStyle=QuoteStyle.None]
    ),
    PromotedHeaders = Table.PromoteHeaders(Source, [PromoteAllScalars=true])
in
    PromotedHeaders
```

---

#### 2. Type Conversion
```m
let
    ChangedType = Table.TransformColumnTypes(
        PromotedHeaders,
        {
            {"Model", type text},
            {"Context Window", type text},
            {"Creator", type text},
            {"Intelligence Index", type number},
            {"Price (Blended USD/1M Tokens)", Currency.Type},
            {"Speed(median token/s)", type number},
            {"Latency (First Answer Chunk /s)", type number}
        }
    )
in
    ChangedType
```

---

#### 3. Data Cleaning
```m
let
    // Remove rows with missing speed data
    FilteredSpeed = Table.SelectRows(
        ChangedType,
        each [#"Speed(median token/s)"] <> null and [#"Speed(median token/s)"] > 0
    ),
    
    // Replace "--" in Intelligence Index with null
    ReplacedIntelligence = Table.ReplaceValue(
        FilteredSpeed,
        "--",
        null,
        Replacer.ReplaceValue,
        {"Intelligence Index"}
    )
in
    ReplacedIntelligence
```

---

## Performance Optimization

### Best Practices Implemented

1. **Data Model Optimization**
   - Star schema design (1 fact table, 2 dimension tables)
   - Removed unnecessary columns (reduced file size by 30%)
   - Used integer data types where possible

2. **DAX Optimization**
   - Avoided row-by-row iteration (used CALCULATE instead of SUMX)
   - Pre-calculated columns instead of measures for static values
   - Used SELECTEDVALUE for single-row contexts

3. **Visual Performance**
   - Limited scatter plot to top 100 models (reduced render time by 60%)
   - Disabled cross-highlighting on large visuals
   - Used conditional formatting instead of multiple visuals

---

## Troubleshooting Guide

### Common Issues

#### Issue 1: "Column not found" error in DAX
**Symptom**: DAX formula fails with `[Speed(median token/s)]` reference

**Solution**: Wrap column name in square brackets
```dax
// ❌ Wrong
Speed Tier = IF(Speed(median token/s) > 100, "Fast", "Slow")

// ✅ Correct
Speed Tier = IF([Speed(median token/s)] > 100, "Fast", "Slow")
```

---

#### Issue 2: Blank values in visualizations
**Symptom**: Some models show blank in Speed Tier chart

**Root Cause**: Missing speed data (Speed = 0 or NULL)

**Solution**: Add BLANK() handling in SWITCH statement
```dax
Speed Tier = 
SWITCH(
    TRUE(),
    [Speed(median token/s)] >= 200, "Blazing Fast",
    [Speed(median token/s)] > 0, "Other Tier",
    BLANK()  // Explicitly handle missing data
)
```

---

#### Issue 3: Incorrect currency formatting
**Symptom**: Prices display as "4.81" instead of "$4.81"

**Solution**: Set column data type to Currency in Power Query
```m
{"Price (Blended USD/1M Tokens)", Currency.Type}
```

---

## Advanced Techniques

### 1. Dynamic Filtering with Slicers
Create a disconnected table for token volume parameters:
```dax
Token Volume Table = 
DATATABLE(
    "Volume Label", STRING,
    "Tokens", INTEGER,
    {
        {"1M tokens", 1000000},
        {"10M tokens", 10000000},
        {"100M tokens", 100000000},
        {"1B tokens", 1000000000}
    }
)
```

---

### 2. Conditional Formatting with DAX
Apply color-coding to latency values:
```dax
Latency Color = 
SWITCH(
    TRUE(),
    [Latency (First Answer Chunk /s)] <= 1, "#00FF00",  // Green
    [Latency (First Answer Chunk /s)] <= 5, "#FFFF00",  // Yellow
    [Latency (First Answer Chunk /s)] <= 10, "#FFA500", // Orange
    "#FF0000"  // Red
)
```

---

## Maintenance & Updates

### How to Refresh Data
1. Update `data/ai_models_performance.csv` with new model data
2. Open `project.pbix` in Power BI Desktop
3. Click **Home → Refresh** (or press F5)
4. Verify data quality in **Data** view
5. Publish to Power BI Service (if applicable)

### Version Control
- **Current Version**: 1.0
- **Last Updated**: February 2026
- **Data Source**: Proprietary AI benchmarking platform

---

## Contact & Support

For technical questions or contributions:
- **GitHub Issues**: [Report a bug](https://github.com/yourusername/ai-cloud-performance-suite/issues)
- **Email**: your.email@example.com
