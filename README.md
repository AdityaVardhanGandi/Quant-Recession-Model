# Bond Market Data Logger

## Overview
Automated webscraping system for collecting bond market data from India and USA sources.

## Files Overview

### Data Templates (CSV)
- `TenorBucketDef.csv` - Editable tenor bucket definitions (3M, 6M, 1Y, 2Y, 3Y, 5Y, 10Y, 30Y)
- `YieldDaily.csv` - Daily yield data by tenor bucket (empty template)
- `AuctionDaily.csv` - Daily auction results (empty template) 
- `SecurityMaster.csv` - Bond security master data (empty template)
- `SecondaryTrade.csv` - Individual secondary market trades (empty template)
- `SecondaryVolumeDaily.csv` - Pre-aggregated daily volumes by bucket (empty template)
- `OutstandingBySecurityMonthly.csv` - Monthly outstanding by ISIN (empty template)
- `OutstandingDaily.csv` - Daily outstanding by bucket (empty template)
- `DailyCurveFeatures.csv` - Final wide-format output template

### Python Scripts
- `utils_bucket_mapper.py` - Utility functions for maturity→bucket mapping and aggregation
- `build_daily_features.py` - Main script to build the wide daily features table

# QRM (minimal)

This workspace contains only the minimal scripts and data for the three-step
goal:

1. Ingest US bond market data: `bond_market_data.csv`.
2. Compute spreads and serialise outputs: `analysis/generate_spreads_and_correlations.py`.
3. Render correlation visualisations (Matplotlib PNGs) with red→blue gradient
   over range [-1, 1] and white for NaNs: `analysis/visualize_correlation_matplotlib.py`.

Advanced features include prototyping time-ordered rolling-correlation tensors
across sovereign yield-spread pairs for evolving tenor-network analysis with
graph/eigen features; integrating Arbitrage-Free Nelson-Siegel with VAR/VARX
for macro-consistent yield-curve scenarios propagated to spreads and correlations
for stress testing; and building regime-detection on yield-curve factors using
change-point and Markov-switching methods to identify tenor-curve regime shifts.

Usage (examples):

Render a single-day heatmap:

```powershell
& D:\Downloads\QRM\.venv\Scripts\python.exe analysis\visualize_correlation_matplotlib.py \
  --npz outputs\correlations_usa.npz \
  --mode daily-heatmap --from-date 2020-03-16 --to-date 2020-03-16 \
  --output outputs\corr_usa_2020-03-16.png
```

Render a timeline heatmap (downsampling applied automatically for very long histories):

```powershell
& D:\Downloads\QRM\.venv\Scripts\python.exe analysis\visualize_correlation_matplotlib.py \
  --npz outputs\correlations_usa.npz --mode timeline-heatmap \
  --output outputs\corr_usa_timeline.png
```

Render a capped 3D volume (use `--max-dates` to control date slices):

```powershell
& D:\Downloads\QRM\.venv\Scripts\python.exe analysis\visualize_correlation_matplotlib.py \
  --npz outputs\correlations_usa.npz --mode volume --max-dates 365 \
  --output outputs\corr_usa_volume.png
```

Requirements are in `requirements.txt`.

**OutstandingDaily.csv** - Daily outstanding amounts:
```csv
date,tenor_bucket,outstanding_amount_cr
2023-01-01,3M,50000
2023-01-01,6M,75000
```

**AuctionDaily.csv** - Auction results (optional):
```csv
date,isin,auction_type,accepted_amount_cr,weighted_avg_yield_pct,cutoff_yield_pct,bid_cover_ratio
2023-01-01,IN1234567890,primary,1000,5.25,5.30,2.1
```

### 3. Run the Feature Builder
```bash
python build_daily_features.py
```

This creates `DailyCurveFeatures_built.csv` with columns:
- `y_*` = yields by bucket (e.g., y_3M, y_6M, y_1Y, ...)
- `vol_*` = secondary turnover (₹ cr) by bucket
- `stock_*` = outstanding amounts (₹ cr) by bucket  
- `turnover_float_*` = turnover/outstanding ratios by bucket
- `auction_amount_total`, `auction_yield_avg`, `auction_bid_cover_avg` = daily auction aggregates

## Data Flow Options

The toolkit supports flexible data ingestion:

### Option 1: Direct Bucket Data
- Populate `YieldDaily.csv`, `SecondaryVolumeDaily.csv`, `OutstandingDaily.csv` directly
- Data already aggregated by tenor bucket
- Fastest processing

### Option 2: Security-Level Data  
- Populate `SecondaryTrade.csv` with individual trades
- Populate `SecurityMaster.csv` with bond details
- Script maps maturity→bucket and aggregates automatically
- More flexible but slower

### Option 3: Mixed Approach
- Use direct bucket data where available
- Fill gaps with security-level data where needed
- Script handles both automatically

## Key Features

### Automatic Bucket Mapping
- Maps remaining maturity to tenor buckets based on `TenorBucketDef.csv`
- Handles securities rolling between buckets over time
- Supports custom bucket definitions

### Volume-Weighted Aggregation
- Aggregates individual trades using volume weighting
- Calculates meaningful average yields by bucket
- Handles missing or sparse data gracefully

### Turnover Analysis
- Calculates turnover/float ratios automatically
- Identifies liquidity patterns by tenor
- Normalizes trading activity by outstanding amounts

### Flexible Date Handling
- Outer joins across all data sources
- Handles missing dates and partial data
- Maintains chronological ordering

## Advanced Usage

### Custom Bucket Definitions
Edit `TenorBucketDef.csv` to change tenor breakpoints:
```python
# Example: Add ultra-short bucket
bucket_name,min_years,max_years
1M,0.0,0.083
3M,0.083,0.25
...
```

### Programmatic Usage
```python
from utils_bucket_mapper import *
from build_daily_features import *

# Load your data
buckets = load_bucket_definitions()
security_master = pd.read_csv('SecurityMaster.csv')

# Map a specific security to bucket
remaining_years = calculate_remaining_maturity_years(
    '2020-01-01', '2025-01-01', '2023-06-01'
)
bucket = map_maturity_to_bucket(remaining_years, buckets)
```

### Integration with Analysis
The output `DailyCurveFeatures_built.csv` is ready for:
- Correlation analysis
- Principal component analysis  
- Term structure modeling
- Risk factor extraction
- Machine learning features

## Data Quality Checks

The script provides diagnostic output:
```
=== Results ===
Output saved to: DailyCurveFeatures_built.csv
Rows: 250
Columns: 35
Date range: 2023-01-01 to 2023-12-31

Column summary:
  date: 250/250 non-null values
  y_3M: 245/250 non-null values
  vol_3M: 180/250 non-null values
  ...
```

Review the column summary to identify data gaps and ensure coverage meets your requirements.

## Next Steps

After generating `DailyCurveFeatures_built.csv`, you can:

1. **Correlation Analysis**: Build correlation matrices across tenors
2. **Shrinkage Estimation**: Apply Ledoit-Wolf shrinkage for robust correlation estimates  
3. **Volume Weighting**: Create volume-weighted correlation matrices
4. **Tensor Construction**: Build 3D tensors for advanced modeling
5. **Risk Modeling**: Extract principal components or risk factors

## Troubleshooting

### Common Issues

**"No data loaded" error:**
- Check that CSV files exist and have correct headers
- Ensure date columns are in YYYY-MM-DD format
- Verify numeric columns don't contain text/special characters

**"No features generated" error:**
- At least one of YieldDaily, SecondaryVolumeDaily, or OutstandingDaily must have data
- Check date formats match across files
- Ensure bucket names in data match TenorBucketDef.csv

**Missing buckets in output:**
- Some buckets may have no data for your date range
- Check maturity mapping with shorter date ranges
- Verify SecurityMaster contains bonds across all desired tenors

### Performance Tips

- Use pre-aggregated data (SecondaryVolumeDaily) when possible
- Limit individual trades data to necessary date ranges
- Remove unnecessary columns from input files
- Use SSD storage for large datasets

## File Formats

All CSV files should use:
- UTF-8 encoding
- Comma separators  
- Date format: YYYY-MM-DD
- Numeric format: Decimal numbers (no commas or currency symbols)
- Headers exactly as specified in templates
