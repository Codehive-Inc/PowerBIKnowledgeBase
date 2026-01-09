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
