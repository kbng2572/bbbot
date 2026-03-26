---
name: powerapps
description: "Develop, troubleshoot, and optimize Microsoft Power Apps. Use for canvas apps, model-driven apps, Power Fx formulas, data connections (Dataverse, SharePoint, SQL), component libraries, responsive layouts, Power Automate integration, solution management, and ALM. Triggers: Power Apps, PowerApps, canvas app, model-driven, Power Fx, Dataverse, Power Platform, low-code app."
---

# Power Apps Development

Guide for building, debugging, and optimizing Microsoft Power Apps solutions.

## Quick Reference

| Task | Reference |
|------|-----------|
| Power Fx formulas, operators, functions | [references/power-fx.md](references/power-fx.md) |
| Canvas app patterns and controls | [references/canvas-apps.md](references/canvas-apps.md) |
| Data connections and Dataverse | [references/data-connections.md](references/data-connections.md) |
| Power Automate integration | [references/power-automate.md](references/power-automate.md) |
| Solution management and ALM | [references/alm.md](references/alm.md) |

## Core Concepts

Power Apps has two main app types:

- **Canvas apps** -- pixel-level control over layout; connect to 200+ data sources; formulas drive all behavior.
- **Model-driven apps** -- metadata-driven UI built on Dataverse; forms, views, dashboards auto-generated from table definitions.

All logic uses **Power Fx**, a strongly-typed, declarative, spreadsheet-inspired formula language.

## Development Workflow

1. **Define data model** -- Dataverse tables, SharePoint lists, or external sources.
2. **Build screens** -- galleries, forms, navigation.
3. **Wire formulas** -- `OnSelect`, `OnChange`, `Items`, `Default`, `Visible`.
4. **Connect flows** -- Power Automate for background processing, approvals, email.
5. **Test** -- use Monitor to trace data calls and formula evaluation.
6. **Package** -- export as managed/unmanaged solution for deployment.

## Common Patterns

### Navigation

```
Navigate(ScreenName, ScreenTransition.Fade, {paramName: value})
Back()
```

### Filtering and Sorting

```
SortByColumns(
    Filter(Employees, Department = "Engineering", Status = "Active"),
    "LastName", SortOrder.Ascending
)
```

### Form Submit with Validation

```
If(
    IsBlank(txtName.Text) Or IsBlank(txtEmail.Text),
    Notify("Fill all required fields", NotificationType.Error),
    SubmitForm(frmEmployee);
    Navigate(SuccessScreen)
)
```

### Delegation Awareness

Power Fx operations on large datasets delegate to the server only for supported functions. Non-delegable operations process at most 500 (or 2000) rows client-side.

**Delegable**: `Filter`, `Sort`, `=`, `<>`, `<`, `>`, `>=`, `<=`, `And`, `Or`, `Not`, `In`, `StartsWith` (some sources).

**Non-delegable**: `Search`, `LookUp` with complex predicates, `CountIf`, `Sum` with Filter, `First(Filter(...))`.

Workaround: use Dataverse views, stored procedures, or Power Automate flows for server-side aggregation.

## Debugging

- **Monitor** (Settings > Advanced > Monitor): real-time trace of data operations, network calls, formula errors.
- **App checker**: flags accessibility, performance, and formula errors at design time.
- **Experimental features**: turn on formula-level error bars for inline debugging.

## Performance Tips

- Use `Concurrent()` to parallelize data loads on screen `OnVisible`.
- Pre-load lookup data into collections with `ClearCollect` in `App.OnStart` or `App.StartScreen`.
- Prefer `LookUp` over `Filter` when only one record is needed.
- Minimize controls per screen (< 300); use components for reuse.
- Avoid `CountRows(Filter(...))` on large datasets -- use server-side counts.
- Use `With()` to cache intermediate results within a formula.
- Enable delayed load for off-screen controls.

## Model-Driven Apps

- Define Dataverse tables with appropriate column types, relationships, and business rules.
- Configure forms (Main, Quick Create, Quick View) and views (Active, Inactive, custom).
- Use business rules for simple field-level validation without code.
- Use JavaScript web resources only when business rules are insufficient.
- Configure dashboards and charts for reporting.

## Security Model

- **Environment-level**: security roles grant CRUD at table and field level.
- **Record-level**: business units, teams, sharing rules control row access.
- **Column-level**: field security profiles restrict sensitive columns.
- Canvas apps respect the signed-in user's Dataverse permissions automatically.
