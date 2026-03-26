# Data Connections

## Table of Contents

- [Dataverse](#dataverse)
- [SharePoint](#sharepoint)
- [SQL Server](#sql-server)
- [Excel and CSV](#excel-and-csv)
- [Custom Connectors](#custom-connectors)
- [Connection Reference Best Practices](#connection-reference-best-practices)

## Dataverse

### When to Use

Dataverse is the recommended data platform for Power Apps. Choose it when:

- Building enterprise apps that need role-based security at table/row/column level.
- Data model requires relationships (1:N, N:N), calculated/rollup columns.
- App needs business rules, workflows, or model-driven features.
- Data volume may exceed SharePoint list limits (>20M rows supported).

### Table Design

```
// Common column types
Single Line of Text    -- names, codes, short strings (max 4000 chars)
Multiple Lines         -- descriptions, notes, rich text
Whole Number           -- integer values
Decimal Number         -- fixed precision (up to 10 decimal places)
Currency               -- locale-aware monetary values
Date Only / Date+Time  -- temporal data
Choice / Choices       -- picklist (single or multi-select)
Yes/No                 -- boolean
Lookup                 -- foreign key to another table (creates 1:N relationship)
Customer               -- polymorphic lookup (Account or Contact)
File / Image           -- binary attachments (up to 128 MB per file column)
Formula                -- server-side calculated column (Power Fx)
```

### Relationships

```
// 1:N (one-to-many)
Account (1) --> Contacts (N)
// Lookup column on Contact points to Account

// N:N (many-to-many)
Students <--> Courses
// Creates an intersect table automatically

// Self-referential
Employee.Manager -> Employee
```

### CRUD Operations

```
// Read
LookUp(Accounts, 'Account Name' = "Contoso")
Filter(Contacts, 'Account'.'Account Name' = "Contoso")

// Create
Patch(Contacts, Defaults(Contacts), {
    'First Name': "Ada",
    'Last Name': "Lovelace",
    'Email': "ada@example.com",
    'Account': LookUp(Accounts, 'Account Name' = "Contoso")
})

// Update
Patch(Contacts, LookUp(Contacts, Email = "ada@example.com"), {
    'Job Title': "Chief Scientist"
})

// Delete
Remove(Contacts, LookUp(Contacts, Email = "ada@example.com"))

// Bulk create
ForAll(
    importTable,
    Patch(Contacts, Defaults(Contacts), {
        'First Name': ThisRecord.FirstName,
        'Last Name': ThisRecord.LastName
    })
)
```

### Dataverse Views

Use Dataverse views for pre-filtered, pre-sorted datasets:

```
// Reference a view directly (delegable)
Filter(Contacts, 'Contacts (Views)'.'Active Contacts')

// Better performance than client-side Filter for complex queries
```

### Dataverse Calculated and Rollup Columns

**Calculated columns**: computed server-side, available immediately.

**Rollup columns**: aggregate child records (Sum, Count, Min, Max, Avg); recalculated on schedule (default 12 hours) or on-demand via workflow.

### Choices (Option Sets)

```
// Global choice (shared across tables)
'Status Reason' column using global choice

// Local choice (table-specific)
Priority: {Low: 1, Medium: 2, High: 3, Critical: 4}

// Reference in formula
If(ThisItem.Priority = 'Priority (Contacts)'.Critical, Color.Red, Color.Black)

// Dropdown items
Choices(Contacts.Priority)
```

## SharePoint

### When to Use

Choose SharePoint when:

- Team already uses SharePoint for document management.
- Data is simple (flat lists, < 20 columns).
- Row count stays under ~5,000 for optimal performance (hard limit ~30M but delegation issues above 5K).
- No complex relationships needed.

### Connecting to SharePoint Lists

```
// After adding SharePoint connection, reference list directly
Filter(EmployeeDirectory, Department.Value = "Engineering")

// SharePoint-specific: Choice columns return records
ThisItem.Status.Value          // text value
ThisItem.Department.Value      // text value

// Person columns
ThisItem.Manager.DisplayName
ThisItem.Manager.Email

// Lookup columns (cross-list)
ThisItem.Project.Value
```

### SharePoint Delegation Limits

Default row limit: 500 (configurable to 2000 in app settings).

**Delegable operations on SharePoint**:
- `Filter` with `=`, `<>`, `<`, `>`, `<=`, `>=`
- `StartsWith` on text columns
- `Sort` on single column
- `IsBlank` / `IsEmpty`

**Non-delegable on SharePoint**:
- `Search` (always client-side)
- `in` / `exactin`
- `Or` conditions across different columns
- `Len`, `Left`, `Right`, `Mid` in filter predicates

### Workarounds for Delegation

```
// Use SharePoint view filtering
Filter(EmployeeDirectory, 'Created By'.Email = User().Email)

// Index columns for performance (SharePoint admin)
// Create indexed columns on frequently filtered fields

// Use a Power Automate flow for complex server-side queries
```

### Attachments

```
// Gallery of attachments
Gallery.Items: ThisItem.Attachments

// Display attachment
Image.Image: ThisItem.AbsoluteUri & "?access_token=" & ThisItem.Value

// Add attachment via AddMediaButton control
```

## SQL Server

### When to Use

Choose SQL Server (or Azure SQL) when:

- Data lives in existing relational databases.
- Complex joins, stored procedures, or views are needed.
- Transactional integrity is critical.
- Data volume is very large (millions of rows).

### Connection Types

- **Direct**: connects with current user credentials (SSO) or SQL auth.
- **On-premises data gateway**: required for on-prem SQL Server.

### CRUD Operations

```
// Read (table/view)
Filter('[dbo].[Employees]', DepartmentID = 3)
LookUp('[dbo].[Employees]', EmployeeID = 42)

// Create
Patch('[dbo].[Employees]', Defaults('[dbo].[Employees]'), {
    FirstName: "Ada",
    LastName: "Lovelace",
    DepartmentID: 3
})

// Update
Patch('[dbo].[Employees]', First(Filter('[dbo].[Employees]', EmployeeID = 42)), {
    JobTitle: "Lead Engineer"
})

// Delete
Remove('[dbo].[Employees]', First(Filter('[dbo].[Employees]', EmployeeID = 42)))
```

### Stored Procedures

```
// Call stored procedure (returns table)
'YourDatabase'.dbo.usp_GetActiveEmployees({@DepartmentID: 3})

// Use result
ClearCollect(colResults, 'YourDatabase'.dbo.usp_GetActiveEmployees({@DepartmentID: 3}))
```

### SQL Delegation

Most filter and sort operations delegate to SQL Server:
- All comparison operators
- `And`, `Or`, `Not`
- `StartsWith`, `EndsWith` (text columns)
- `IsBlank`
- `Sort`, `SortByColumns`
- `In`

Non-delegable: `Search`, `Trim`, `TrimEnds`, `Len`, regex-based operations.

### SQL Views

Create SQL views for complex joins, then connect Power Apps to the view as if it were a table. Read-only by default; use `INSTEAD OF` triggers to make views updatable.

## Excel and CSV

### When to Use

Use Excel/CSV only for:

- Prototyping and demos.
- Small, static datasets (< 500 rows).
- Import/export scenarios.

Not recommended for production apps due to locking issues and performance.

### Excel as Data Source

```
// Excel table in OneDrive or SharePoint
// Requires data to be formatted as an Excel Table (Insert > Table)
Filter(Table1, Status = "Active")

// Limitations:
// - No delegation (all data pulled client-side)
// - File locking issues with concurrent users
// - Max ~2000 rows practical limit
```

### Import CSV to Collection

```
// Use a flow or manual import
// Power Automate: parse CSV and return JSON to app
ClearCollect(colImported, YourFlow.Run(fileContent))
```

## Custom Connectors

### When to Use

Create custom connectors to integrate with any REST API not covered by built-in connectors.

### Definition

Custom connectors are defined via OpenAPI (Swagger) specification:

```
// Required elements:
// - Base URL
// - Authentication type (API Key, OAuth 2.0, Basic Auth)
// - Actions (operations)
// - Request/response schemas

// Authentication types supported:
// - No auth
// - API Key (header or query parameter)
// - Basic authentication
// - OAuth 2.0 (Authorization Code, Client Credentials)
```

### Usage in Power Fx

```
// After adding custom connector as data source
Set(gblWeather, YourConnector.GetWeather({city: "London"}))

// Access response fields
lblTemp.Text: gblWeather.temperature & "°C"
```

### Tips

- Define response schemas accurately for Power Apps to generate correct column types.
- Use `x-ms-summary` and `x-ms-visibility` in OpenAPI spec to control how fields appear in the maker UI.
- Test with Postman or similar before building the connector.

## Connection Reference Best Practices

- Use **connection references** (not embedded connections) for solution-aware apps.
- Each environment should have its own connection configured in the connection reference.
- Store connection reference details in the solution, not hardcoded in formulas.
- For multi-environment deployments, update connection references as part of the ALM pipeline.
- Limit the number of data sources per app (each adds startup overhead); aim for < 10.
- Use `Concurrent()` to parallelize initial data loads from multiple sources.
