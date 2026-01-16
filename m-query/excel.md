# Excel M Query Patterns

## Confirmed Working Pattern

```m
let
    Source = Excel.Workbook(File.Contents("C:\Users\File.xlsx"), null, true),
    Sheet = Source{[Item="All Dealers", Kind="Sheet"]}[Data],
    #"Promoted Headers" = Table.PromoteHeaders(Sheet, [PromoteAllScalars=true]),
    #"Replaced Value" = Table.ReplaceValue(#"Promoted Headers", null, "NA", Replacer.ReplaceValue, {"Dealer Rating"})
in
    #"Replaced Value"
```

## Untested Patterns


#!/usr/bin/env python3
"""
Extract and consolidate data from Fitch Excel report.
Output format: Account, Company (sheet name), Currency (row 13), Amount
"""

import re
import pandas as pd
from pathlib import Path


def parse_account(raw: str) -> str:
    """Extract account code from format like '[A00010][10409]' -> '10409'"""
    match = re.search(r'\]\[(\d+)\]', str(raw))
    return match.group(1) if match else str(raw).strip()


def parse_currency(raw: str) -> str:
    """Extract currency code from format like 'ADP - Andorran Peseta' -> 'ADP'"""
    return str(raw).split(' - ')[0].split('.')[-1].strip('[]')


def extract_fitch_data(excel_path: str, output_path: str = "consolidated_output.xlsx"):
    """Extract and consolidate data from multi-sheet Excel file."""
    
    xlsx = pd.ExcelFile(excel_path)
    all_records = []
    total_sheets = len(xlsx.sheet_names)
    
    for idx, sheet_name in enumerate(xlsx.sheet_names, 1):
        print(f"Processing sheet {idx}/{total_sheets}: {sheet_name}", end="\r")
        df = pd.read_excel(xlsx, sheet_name=sheet_name, header=None)
        
        # Row 13 (0-indexed = row 12) contains currency headers
        currency_row = df.iloc[12, 1:].values  # Skip column A
        accounts = df.iloc[15:, 0].values  # Account codes in column A
        data_values = df.iloc[15:, 1:].values  # Data starts from column B
        
        for row_idx, raw_account in enumerate(accounts):
            if pd.isna(raw_account) or str(raw_account).strip() == "":
                continue
            account = parse_account(raw_account)
                
            for col_idx, raw_currency in enumerate(currency_row):
                if pd.isna(raw_currency) or str(raw_currency).strip() == "":
                    continue
                currency = parse_currency(raw_currency)
                amount = data_values[row_idx, col_idx] if row_idx < len(data_values) else None
                
                if pd.isna(amount):
                    continue
                
                all_records.append({
                    "Account": account,
                    "Company": str(sheet_name).strip(),
                    "Currency": currency,
                    "Amount": amount
                })
    
    # Create consolidated DataFrame and save to Excel
    result_df = pd.DataFrame(all_records)
    result_df.to_excel(output_path, index=False, sheet_name="Consolidated")
    
    print(f"\n✓ Extracted {len(result_df):,} records to '{output_path}'")
    print(f"  Sheets processed: {total_sheets}")
    print(f"  Unique accounts: {result_df['Account'].nunique()}")
    print(f"  Unique currencies: {result_df['Currency'].nunique()}")
    
    return result_df


def main():
    import argparse
    
    parser = argparse.ArgumentParser(description="Extract Fitch Excel data to consolidated Excel")
    parser.add_argument("excel_file", help="Path to the input Excel file")
    parser.add_argument("-o", "--output", default="consolidated_output.xlsx", 
                        help="Output Excel file path (default: consolidated_output.xlsx)")
    args = parser.parse_args()
    
    if not Path(args.excel_file).exists():
        print(f"Error: File '{args.excel_file}' not found")
        return
    
    extract_fitch_data(args.excel_file, args.output)


if __name__ == "__main__":
    main()

