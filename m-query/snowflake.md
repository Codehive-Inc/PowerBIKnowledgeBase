# Snowflake M Query Patterns

## Confirmed Working Pattern

```m
let
    Source = Snowflake.Databases(
        "serverabc",
        "abc",
        [
            Query = "SELECT DateLabel, SnapshotDate, Future, Latest#(lf)FROM Xtable#(lf)WHERE DateLabel IS NOT NULL#(lf)GROUP BY DateLabel, SnapshotDate, Future, Latest",
            CreateNavigationProperties = false
        ]
    ),
    #"Filtered Rows" = Table.SelectRows(Source, each true)
in
    #"Filtered Rows"
```

---

## Untested Patterns

### Basic M Query (Connecting to a Table)

```m
let
    // 1. Establish the Snowflake connection
    // Replace with your actual server, database, and warehouse.
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    
    // 2. Navigate to the desired schema
    // Case sensitivity matters! Use uppercase if defined that way in Snowflake.
    Schema = Source{[Name="YOUR_SCHEMA_NAME", Kind="Schema"]}[Data],
    
    // 3. Select the desired table
    YourTable = Schema{[Name="YOUR_TABLE_NAME", Kind="Table"]}[Data]
in
    YourTable
```

### Using a Custom SQL Query (Native Query)

```m
let
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    
    // Use Value.NativeQuery to execute custom SQL
    // Query folding can still work for subsequent M steps if the native query is simple enough
    CustomSQLResult = Value.NativeQuery(
        Source,
        "SELECT
            CUSTOMER_ID,
            ORDER_DATE,
            PRODUCT_NAME,
            QUANTITY * PRICE AS LINE_TOTAL
         FROM YOUR_DATABASE_NAME.YOUR_SCHEMA_NAME.ORDERS_TABLE
         WHERE ORDER_DATE >= '2023-01-01';",
        null,
        [EnableFolding = true]
    )
in
    CustomSQLResult
```

### Connecting to a View

```m
let
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    Schema = Source{[Name="YOUR_SCHEMA_NAME", Kind="Schema"]}[Data],
    
    // Select the desired view (note Kind="View")
    YourView = Schema{[Name="YOUR_VIEW_NAME", Kind="View"]}[Data]
in
    YourView
```

### Custom Query with M Transformations (Alternative to Native Query)

```m
let
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    Schema = Source{[Name="YOUR_SCHEMA_NAME", Kind="Schema"]}[Data],
    
    // Connect to base table or VIEW (use Kind="View" for views)
    YourBaseTableOrView = Schema{[Name="YOUR_TABLE_OR_VIEW_NAME", Kind="Table"]}[Data],
    
    // Apply M transformations - Power BI will try to fold these back to Snowflake
    #"Filtered Rows" = Table.SelectRows(YourBaseTableOrView, each [ORDER_DATE] >= #date(2023, 1, 1)),
    #"Removed Columns" = Table.RemoveColumns(#"Filtered Rows", {"SOME_UNUSED_COLUMN"}),
    #"Added Line Total" = Table.AddColumn(#"Removed Columns", "LINE_TOTAL", each [QUANTITY] * [PRICE], type number)
in
    #"Added Line Total"
```

---

## Not Recommended: ODBC Connection

> ⚠️ This approach loses Snowflake connector optimizations and is generally not recommended.

```m
let
    // Generic ODBC connection (less optimized)
    Source = Odbc.DataSource(
        "Driver={SnowflakeDSIIDriver};Server=your_snowflake_account.snowflakecomputing.com;Database=YOUR_DATABASE_NAME;Warehouse=YOUR_WAREHOUSE_NAME",
        [HierarchicalNavigation = true]
    ),
    
    CustomSQLResult = Value.NativeQuery(
        Source,
        "SELECT CUSTOMER_ID, ORDER_DATE FROM YOUR_DATABASE_NAME.YOUR_SCHEMA_NAME.ORDERS_TABLE WHERE ORDER_DATE >= '2023-01-01';",
        null,
        [EnableFolding = true]
    )
in
    CustomSQLResult
```
