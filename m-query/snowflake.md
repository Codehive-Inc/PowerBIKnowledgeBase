Confirmed Working Pattern
------------------------------------------------------
let
    Source =Snowflake.Databases("serverabc", "abc", [Query="SELECT DateLabel, SnapshotDate, Future, Latest#(lf)FROM Xtable#(lf)WHERE DateLabel IS NOT NULL#(lf)GROUP BY DateLabel, SnapshotDate, Future, Latest", CreateNavigationProperties=false]),
    #"Filtered Rows" = Table.SelectRows(Source, each true)
in
    #"Filtered Rows"


------------------------------------------------------
Untested other patterns
------------------------------------------------------
Basic M Query for Snowflake (Connecting to a Table)
------------------------------------------------------
 
let
    // 1. Establish the Snowflake connection
    // Replace with your actual server, database, and warehouse.
    // Warehouse is critical for Snowflake performance and cost.
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com", // e.g., "xy12345.us-east-1" or "mycompany.east-us-2"
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true, // Ensure query folding is enabled
            Warehouse = "YOUR_WAREHOUSE_NAME" // Mandatory for Snowflake
        ]
    ),
    
    // 2. Navigate to the desired schema (e.g., 'PUBLIC' or 'SALES')
    // Case sensitivity matters here! Use uppercase if that's how it's defined in Snowflake.
    Schema = Source{[Name="YOUR_SCHEMA_NAME",Kind="Schema"]}[Data],
    
    // 3. Select the desired table within that schema
    // Case sensitivity matters here too!
    YourTable = Schema{[Name="YOUR_TABLE_NAME",Kind="Table"]}[Data]
in
    YourTable
 
 
M Query for Snowflake (Using a Custom SQL Query)
------------------------------------------------------
 
let
    // 1. Establish the Snowflake connection (same as above)
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    
    // 2. Use Value.NativeQuery to execute custom SQL
    // The query folding *can still work* for subsequent M steps if the native query is simple enough.
    CustomSQLResult = Value.NativeQuery(
        Source, // Pass the initial connection
        "SELECT
            CUSTOMER_ID,
            ORDER_DATE,
            PRODUCT_NAME,
            QUANTITY * PRICE AS LINE_TOTAL
         FROM YOUR_DATABASE_NAME.YOUR_SCHEMA_NAME.ORDERS_TABLE
         WHERE ORDER_DATE >= '2023-01-01';",
        null, // No parameters to pass to the SQL query
        [EnableFolding = true] // Enable folding for the custom query and subsequent steps
    )
in
    CustomSQLResult
 
M Query for Snowflake (Connecting to a View)
------------------------------------------------------
let
    Source = Snowflake.Database(
        "your_snowflake_account.snowflakecomputing.com",
        "YOUR_DATABASE_NAME",
        [
            EnableFolding = true,
            Warehouse = "YOUR_WAREHOUSE_NAME"
        ]
    ),
    Schema = Source{[Name="YOUR_SCHEMA_NAME",Kind="Schema"]}[Data],
    
    // 3. Select the desired view within that schema
    YourView = Schema{[Name="YOUR_VIEW_NAME",Kind="View"]}[Data] // Note Kind="View"
in
    YourView
 ------------------------------------------------------
For Custom Query --> alternative to Native QUery 
 ------------------------------------------------------
let

    Source = Snowflake.Database(

        "your_snowflake_account.snowflakecomputing.com",

        "YOUR_DATABASE_NAME",

        [

            EnableFolding = true,

            Warehouse = "YOUR_WAREHOUSE_NAME"

        ]

    ),

    Schema = Source{[Name="YOUR_SCHEMA_NAME",Kind="Schema"]}[Data],

    // Connect to your base table or a pre-defined VIEW in Snowflake

    // This is the starting point.

    YourBaseTableOrView = Schema{[Name="YOUR_TABLE_OR_VIEW_NAME",Kind="Table"]}[Data], 

    // If it's a view, use Kind="View" here

    // Apply subsequent M transformations.

    // Power BI will try to fold these back to Snowflake.

    #"Filtered Rows" = Table.SelectRows(YourBaseTableOrView, each [ORDER_DATE] >= #date(2023, 1, 1)),

    #"Removed Columns" = Table.RemoveColumns(#"Filtered Rows",{"SOME_UNUSED_COLUMN"}),

    #"Added Line Total" = Table.AddColumn(#"Removed Columns", "LINE_TOTAL", each [QUANTITY] * [PRICE], type number)

in

    #"Added Line Total"
 
// THIS IS GENERALLY NOT RECOMMENDED FOR SNOWFLAKE IN POWER BI

// It might work for ODBC but loses Snowflake connector optimizations.

let

    // Establish a generic ODBC connection to Snowflake (less optimized)

    Source = Odbc.DataSource(

        "Driver={SnowflakeDSIIDriver};Server=your_snowflake_account.snowflakecomputing.com;Database=YOUR_DATABASE_NAME;Warehouse=YOUR_WAREHOUSE_NAME",

        [HierarchicalNavigation=true]

    ),

    // Now you might be able to use Value.NativeQuery on the ODBC connection

    CustomSQLResult = Value.NativeQuery(

        Source, // Pass the ODBC connection

        "SELECT CUSTOMER_ID, ORDER_DATE FROM YOUR_DATABASE_NAME.YOUR_SCHEMA_NAME.ORDERS_TABLE WHERE ORDER_DATE >= '2023-01-01';",

        null,

        [EnableFolding=true]

    )

in

    CustomSQLResult
 