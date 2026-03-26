# Power Fx Reference

## Table of Contents

- [Data Types](#data-types)
- [Operators](#operators)
- [Core Functions](#core-functions)
- [Table Functions](#table-functions)
- [Text Functions](#text-functions)
- [Date and Time Functions](#date-and-time-functions)
- [Math Functions](#math-functions)
- [Logical Functions](#logical-functions)
- [Variables and Collections](#variables-and-collections)
- [Error Handling](#error-handling)
- [Named Formulas and User-Defined Functions](#named-formulas-and-user-defined-functions)

## Data Types

| Type | Examples | Notes |
|------|----------|-------|
| Number | `42`, `3.14` | IEEE 754 double |
| Text | `"hello"` | Unicode string |
| Boolean | `true`, `false` | |
| Date | `Date(2026, 3, 15)` | Date only |
| Time | `Time(14, 30, 0)` | Time only |
| DateTime | `DateTimeValue("2026-03-15T14:30:00")` | Combined |
| Record | `{Name: "Ada", Age: 30}` | Single row |
| Table | `[{x: 1}, {x: 2}]` | Collection of records |
| GUID | `GUID("...")` | Dataverse primary keys |
| Color | `Color.Red`, `RGBA(255, 0, 0, 1)` | |
| Blank | `Blank()` | Null equivalent |

## Operators

```
// Arithmetic
+ - * / ^

// Comparison
= <> < > <= >=

// Logical
And(a, b)  Or(a, b)  Not(a)
&&  ||  !    // alternative syntax

// Text concatenation
"Hello" & " " & "World"

// In operator (delegation-safe on some sources)
"value" in ColumnName
"value" exactin ColumnName

// Record scope (disambiguation)
[@FieldName]          // current record
TableName[@FieldName] // explicit table scope

// Self / Parent / ThisItem
Self.Text             // current control
Parent.Width          // parent container
ThisItem.Name         // current gallery item
```

## Core Functions

### Navigation

```
Navigate(Screen, Transition, {context vars})
Back()

// Transitions: ScreenTransition.Fade, .Cover, .CoverRight, .UnCover, .None
```

### Notifications

```
Notify("Saved!", NotificationType.Success)
Notify("Error", NotificationType.Error)
Notify("Warning", NotificationType.Warning)
Notify("Info", NotificationType.Information)
```

### Set and UpdateContext

```
Set(gblUserName, User().FullName)        // global variable
UpdateContext({locShowDialog: true})       // screen-scoped variable
```

## Table Functions

### Filter

```
Filter(DataSource, Condition1, Condition2)
// Multiple conditions are AND-ed
Filter(Employees, Department = "Eng", Salary > 100000)
```

### LookUp

```
LookUp(Employees, Email = "ada@example.com")           // returns first match (record)
LookUp(Employees, Email = "ada@example.com", FullName)  // returns single field
```

### Search

```
Search(Employees, SearchInput.Text, "FirstName", "LastName", "Email")
// Non-delegable -- operates on client-side data only
```

### Sort / SortByColumns

```
Sort(DataSource, ColumnName, SortOrder.Ascending)
SortByColumns(DataSource, "LastName", SortOrder.Ascending, "FirstName", SortOrder.Ascending)
```

### AddColumns / DropColumns / RenameColumns / ShowColumns

```
AddColumns(Employees, "FullName", FirstName & " " & LastName)
DropColumns(Employees, "SSN", "Salary")
RenameColumns(Employees, "Dept", "Department")
ShowColumns(Employees, "FirstName", "LastName", "Email")
```

### GroupBy / Ungroup

```
GroupBy(Sales, "Region", "RegionRecords")
Ungroup(GroupedTable, "RegionRecords")
```

### Distinct

```
Distinct(Employees, Department)
// Returns single-column table of unique values
```

### ForAll

```
ForAll(SelectedItems, Remove(DataSource, ThisRecord))
ForAll(Sequence(10), Patch(Tasks, Defaults(Tasks), {Title: "Task " & Value}))
```

### CountRows / CountIf / SumIf / AverageIf

```
CountRows(Employees)
CountIf(Employees, Department = "Eng")
Sum(Sales, Amount)
Average(Scores, Value)
```

### First / Last / FirstN / LastN

```
First(SortByColumns(Tasks, "DueDate", SortOrder.Ascending))
Last(Tasks)
FirstN(Tasks, 5)
LastN(Tasks, 10)
```

### Patch (Create / Update)

```
// Create
Patch(Employees, Defaults(Employees), {FirstName: "Ada", LastName: "Lovelace"})

// Update
Patch(Employees, LookUp(Employees, ID = 42), {Department: "Research"})

// Bulk patch
Patch(Employees, updateTable)
```

### Remove

```
Remove(Employees, ThisItem)
Remove(Employees, LookUp(Employees, ID = 42))
RemoveIf(Employees, Status = "Inactive")
```

### Collect / ClearCollect / Clear

```
Collect(colCart, {ProductID: 1, Qty: 2})
ClearCollect(colEmployees, Employees)  // refresh local copy
Clear(colCart)                          // empty the collection
```

## Text Functions

```
Len("hello")                        // 5
Left("hello", 3)                    // "hel"
Right("hello", 2)                   // "lo"
Mid("hello", 2, 3)                  // "ell"
Upper("hello")                      // "HELLO"
Lower("HELLO")                      // "hello"
Trim("  hi  ")                      // "hi"
Substitute("2026-01-01", "-", "/")  // "2026/01/01"
Text(Now(), "yyyy-mm-dd")           // "2026-03-26"
Value("42")                         // 42
Concatenate("a", "b", "c")         // "abc"  (or use & operator)
StartsWith("hello", "he")          // true
EndsWith("hello", "lo")            // true
IsBlank(txtInput.Text)             // true if empty or blank
IsMatch("abc123", Match.Alphanumeric) // pattern matching
Split("a,b,c", ",")               // ["a", "b", "c"]
```

## Date and Time Functions

```
Now()                               // current date+time
Today()                             // current date (midnight)
Year(myDate)                        // extract year
Month(myDate)                       // extract month (1-12)
Day(myDate)                         // extract day (1-31)
Hour(myDateTime)                    // extract hour (0-23)
DateAdd(Today(), 7, TimeUnit.Days)  // add 7 days
DateDiff(startDate, endDate, TimeUnit.Days)  // difference in days
Weekday(Today())                    // 1=Sun ... 7=Sat
Text(Now(), "dddd, mmmm d, yyyy")  // "Thursday, March 26, 2026"

// Construct dates
Date(2026, 3, 26)
Time(14, 30, 0)
DateTimeValue("2026-03-26T14:30:00Z")
```

## Math Functions

```
Abs(-5)         // 5
Round(3.14, 1)  // 3.1
RoundUp(3.11, 1)  // 3.2
RoundDown(3.19, 1)  // 3.1
Power(2, 10)    // 1024
Sqrt(16)        // 4
Mod(10, 3)      // 1
Max(1, 2, 3)    // 3
Min(1, 2, 3)    // 1
Rand()          // 0 to 1
RandBetween(1, 100)
Sequence(5)     // [1, 2, 3, 4, 5]
Sequence(5, 0)  // [0, 1, 2, 3, 4]
```

## Logical Functions

```
If(condition, thenValue, elseValue)
If(score >= 90, "A", score >= 80, "B", score >= 70, "C", "F")

Switch(status,
    "Active", Color.Green,
    "Pending", Color.Yellow,
    "Inactive", Color.Gray,
    Color.Black  // default
)

Coalesce(field1, field2, "default")    // first non-blank
IfError(1/0, -1)                       // error fallback
IsError(1/0)                           // true
IsBlank(value)                         // true if blank
IsEmpty(table)                         // true if no rows
IsNumeric("42")                        // true
```

## Variables and Collections

### Global Variables (`Set`)

```
Set(gblCurrentUser, User())
Set(gblTheme, "Dark")
// Available across all screens; prefix with gbl by convention
```

### Context Variables (`UpdateContext`)

```
UpdateContext({locDialogVisible: true, locSelectedItem: ThisItem})
// Scoped to current screen; prefix with loc by convention
```

### Collections

```
ClearCollect(colItems, DataSource)   // snapshot data locally
Collect(colItems, newRecord)         // add row
Remove(colItems, record)             // remove row
Patch(colItems, oldRecord, {Field: newValue})  // update row
Clear(colItems)                      // empty collection
// Collections persist across screens within a session
```

### Naming Conventions

| Prefix | Scope | Example |
|--------|-------|---------|
| `gbl` | Global variable | `gblCurrentUser` |
| `loc` | Context variable | `locShowDialog` |
| `col` | Collection | `colEmployees` |

## Error Handling

```
IfError(
    Patch(Employees, Defaults(Employees), {Name: txtName.Text}),
    Notify("Save failed: " & FirstError.Message, NotificationType.Error)
)

IsError(Value("not a number"))  // true

// Structured error info
FirstError.Kind     // ErrorKind enum
FirstError.Message  // human-readable message
AllErrors           // table of all errors from last operation
```

### ErrorKind Values

`Div0`, `InvalidArgument`, `MissingRequired`, `NotFound`, `NotApplicable`, `ReadPermission`, `CreatePermission`, `EditPermission`, `DeletePermission`, `Network`, `Sync`, `Validation`, `Custom`

## Named Formulas and User-Defined Functions

### Named Formulas (App-level)

Defined in `App.Formulas` -- recalculated reactively:

```
TotalEmployees = CountRows(Employees);
ActiveCount = CountIf(Employees, Status = "Active");
```

### User-Defined Functions (UDFs, preview)

```
CalcDiscount(price: Number, pct: Number): Number = price * (1 - pct / 100);

// Usage
Set(finalPrice, CalcDiscount(99.99, 15))
```
