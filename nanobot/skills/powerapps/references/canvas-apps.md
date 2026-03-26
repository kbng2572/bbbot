# Canvas App Patterns

## Table of Contents

- [App Structure](#app-structure)
- [Screen Patterns](#screen-patterns)
- [Gallery Patterns](#gallery-patterns)
- [Form Patterns](#form-patterns)
- [Component Library Patterns](#component-library-patterns)
- [Responsive Layout](#responsive-layout)
- [Theming and Styling](#theming-and-styling)
- [Offline Support](#offline-support)
- [Accessibility](#accessibility)

## App Structure

### App Object Properties

```
App.OnStart:
    // Initialize global state (runs once on app open)
    Set(gblCurrentUser, User());
    Set(gblIsAdmin, LookUp(Admins, Email = User().Email, true, false));
    ClearCollect(colDepartments, Departments);

App.StartScreen:
    // Conditional start screen (preferred over Navigate in OnStart)
    If(gblIsAdmin, AdminDashboard, HomeScreen)

App.Formulas:
    // Named formulas -- reactively recalculated
    UserDisplayName = gblCurrentUser.FullName;
    IsTablet = App.Width >= 1024;
```

### Screen Lifecycle

| Property | When it runs |
|----------|-------------|
| `OnVisible` | Each time the screen becomes visible |
| `OnHidden` | Each time the user navigates away |

Use `OnVisible` for data refresh:

```
Screen.OnVisible:
    ClearCollect(colTasks, Filter(Tasks, AssignedTo.Email = User().Email));
    Set(locLoading, false)
```

## Screen Patterns

### Master-Detail (Split Screen)

Left side: Gallery listing items.
Right side: Detail form for selected item.

```
// Gallery.OnSelect
Set(locSelectedItem, ThisItem)

// Detail form Items
{locSelectedItem}

// Conditional visibility for detail pane
Visible: !IsBlank(locSelectedItem)
```

### Browse-Detail-Edit (Three-Screen)

Standard pattern for CRUD applications:

- **BrowseScreen**: Gallery with search bar and add button.
- **DetailScreen**: Display form showing selected record.
- **EditScreen**: Edit form for create and modify operations.

```
// BrowseScreen Gallery.OnSelect
Navigate(DetailScreen, ScreenTransition.None, {locRecord: ThisItem})

// DetailScreen Edit button.OnSelect
Navigate(EditScreen, ScreenTransition.None, {locRecord: locRecord})

// EditScreen Save button.OnSelect
SubmitForm(frmEdit);

// EditScreen form.OnSuccess
Back()
```

### Loading Screen Pattern

```
// LoadingScreen.OnVisible
Concurrent(
    ClearCollect(colEmployees, Employees),
    ClearCollect(colDepartments, Departments),
    ClearCollect(colRoles, Roles),
    Set(gblConfig, LookUp(Config, Key = "AppSettings"))
);
Navigate(HomeScreen, ScreenTransition.Fade)
```

### Dialog / Popup Overlay

Use a container with conditional visibility instead of a separate screen:

```
// conDialog container
Visible: locShowDialog

// Background overlay
Fill: RGBA(0, 0, 0, 0.5)

// Confirm button.OnSelect
// ... perform action ...
UpdateContext({locShowDialog: false})

// Cancel button.OnSelect
UpdateContext({locShowDialog: false})
```

## Gallery Patterns

### Search + Filter Gallery

```
// Gallery.Items
SortByColumns(
    Filter(
        colEmployees,
        (IsBlank(drpDepartment.Selected.Value) Or Department = drpDepartment.Selected.Value),
        (IsBlank(txtSearch.Text) Or
         StartsWith(FirstName, txtSearch.Text) Or
         StartsWith(LastName, txtSearch.Text))
    ),
    "LastName", If(locSortAsc, SortOrder.Ascending, SortOrder.Descending)
)
```

### Gallery with Selection Highlighting

```
// Gallery template Fill
If(ThisItem.ID = locSelectedItem.ID, RGBA(0, 120, 212, 0.1), Transparent)
```

### Infinite Scroll / Pagination

Power Apps galleries load data incrementally. For server-side paging:

```
// Load more button
Collect(colItems, LastN(Filter(Items, ID > Last(colItems).ID), 50))
```

### Nested Gallery (Grouped Data)

```
// Outer gallery Items
Distinct(colTasks, Category)

// Inner gallery Items (inside outer template)
Filter(colTasks, Category = ThisItem.Value)
```

## Form Patterns

### Edit Form with Validation

```
// Save button.OnSelect
If(
    IsBlank(DataCardValue_Name.Text),
    Notify("Name is required", NotificationType.Error);
    SetFocus(DataCardValue_Name),

    Not(IsMatch(DataCardValue_Email.Text, Match.Email)),
    Notify("Invalid email", NotificationType.Error);
    SetFocus(DataCardValue_Email),

    // All valid
    SubmitForm(frmEmployee)
)

// Form.OnSuccess
Notify("Saved successfully", NotificationType.Success);
Back()

// Form.OnFailure
Notify("Error: " & frmEmployee.Error, NotificationType.Error)
```

### Default Values for New Records

```
// Form.Mode
If(IsBlank(locRecord), FormMode.New, FormMode.Edit)

// DataCard Default
If(frmEmployee.Mode = FormMode.New, User().FullName, ThisItem.CreatedBy)
```

### Cascading Dropdowns

```
// Country dropdown Items
Distinct(Locations, Country)

// State dropdown Items
Filter(Locations, Country = drpCountry.Selected.Value).State

// City dropdown Items
Filter(Locations, Country = drpCountry.Selected.Value, State = drpState.Selected.Value).City

// Reset dependent dropdowns on parent change
// drpCountry.OnChange
Reset(drpState);
Reset(drpCity)
```

### Multi-Step Form (Wizard)

```
// Track current step
UpdateContext({locStep: 1})

// Step container visibility
conStep1.Visible: locStep = 1
conStep2.Visible: locStep = 2
conStep3.Visible: locStep = 3

// Navigation
btnNext.OnSelect: UpdateContext({locStep: locStep + 1})
btnBack.OnSelect: UpdateContext({locStep: locStep - 1})

// Progress indicator
lblProgress.Text: "Step " & locStep & " of 3"
```

## Component Library Patterns

### Custom Component Definition

Components are reusable UI building blocks with custom input/output properties.

**Input properties**: data flows in (e.g., `Items`, `SelectedValue`, `Theme`).
**Output properties**: data flows out (e.g., `Selected`, `IsValid`).

### Header Component

```
// Custom input properties
cmpHeader.Title (Text, default: "App Title")
cmpHeader.ShowBack (Boolean, default: false)
cmpHeader.UserName (Text, default: "")

// Inside component
lblTitle.Text: cmpHeader.Title
btnBack.Visible: cmpHeader.ShowBack
lblUser.Text: cmpHeader.UserName

// Usage on screen
cmpHeader_1.Title: "Employee Directory"
cmpHeader_1.ShowBack: true
cmpHeader_1.UserName: gblCurrentUser.FullName
```

### Reusable Search Box Component

```
// Custom input properties
cmpSearch.Placeholder (Text)

// Custom output properties
cmpSearch.SearchText (Text) = txtSearch.Text

// Usage
Gallery.Items: Filter(DataSource, StartsWith(Name, cmpSearch_1.SearchText))
```

## Responsive Layout

### Container-Based Layout

Use **horizontal** and **vertical containers** with flexible sizing:

```
// Main container (vertical)
LayoutDirection: Vertical
Width: App.Width
Height: App.Height

// Header container (fixed height)
Height: 60
LayoutMinWidth: 0

// Content container (flexible)
Flexible height: true
Fill portion: 1

// Footer container (fixed height)
Height: 48
```

### Breakpoint-Based Responsiveness

```
// App.Formulas
IsPhone = App.Width < 600;
IsTablet = App.Width >= 600 And App.Width < 1024;
IsDesktop = App.Width >= 1024;

// Conditional layout
conSidebar.Visible: IsDesktop
conMobileNav.Visible: IsPhone
galItems.TemplateSize: If(IsPhone, 80, 60)
```

### Auto-Width and Wrap

```
// Horizontal container with wrapping
Wrap: true
Gap: 8
PaddingLeft: 16
PaddingRight: 16
```

## Theming and Styling

### Centralized Theme Record

```
// App.OnStart or App.Formulas
gblTheme = {
    Primary: ColorValue("#0078D4"),
    PrimaryDark: ColorValue("#005A9E"),
    Background: Color.White,
    Surface: ColorValue("#F3F2F1"),
    TextPrimary: ColorValue("#323130"),
    TextSecondary: ColorValue("#605E5C"),
    Error: ColorValue("#A4262C"),
    Success: ColorValue("#107C10"),
    BorderRadius: 4,
    FontFamily: Font.'Segoe UI',
    FontSizeSmall: 12,
    FontSizeBase: 14,
    FontSizeLarge: 18
};

// Usage
btnSave.Fill: gblTheme.Primary
btnSave.Color: Color.White
btnSave.BorderRadius: gblTheme.BorderRadius
lblTitle.Font: gblTheme.FontFamily
lblTitle.Size: gblTheme.FontSizeLarge
```

## Offline Support

### SaveData / LoadData

```
// Save to local device cache
SaveData(colTasks, "TasksCache")

// Load from cache on app start
LoadData(colTasks, "TasksCache", true)
// third param = ignore if missing

// Check connectivity
If(Connection.Connected,
    ClearCollect(colTasks, Tasks),
    LoadData(colTasks, "TasksCache", true)
)
```

### Offline Queue Pattern

```
// Queue changes locally
Collect(colPendingChanges, {
    Action: "Create",
    TableName: "Tasks",
    Data: {Title: txtTitle.Text, Status: "New"}
})

// Sync when back online
ForAll(
    colPendingChanges,
    If(Action = "Create",
        Patch(Tasks, Defaults(Tasks), Data)
    )
);
Clear(colPendingChanges)
```

## Accessibility

- Set `AccessibleLabel` on every interactive control.
- Ensure tab order (`TabIndex`) follows logical reading order.
- Maintain 4.5:1 contrast ratio for text (WCAG AA).
- Use `SetFocus()` to direct keyboard users to error fields after validation.
- Provide `Tooltip` for icon-only buttons.
- Test with screen reader (Narrator on Windows).
- Avoid conveying information through color alone -- add icons or text.
