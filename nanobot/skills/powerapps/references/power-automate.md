# Power Automate Integration

## Table of Contents

- [Triggering Flows from Power Apps](#triggering-flows-from-power-apps)
- [Common Flow Patterns](#common-flow-patterns)
- [Approval Flows](#approval-flows)
- [Scheduled and Automated Flows](#scheduled-and-automated-flows)
- [Error Handling in Flows](#error-handling-in-flows)
- [Performance and Limits](#performance-and-limits)

## Triggering Flows from Power Apps

### Creating a Flow for Power Apps

1. In Power Automate, create an **Instant cloud flow** with the trigger **PowerApps (V2)**.
2. Add input parameters the app will provide (Text, Number, Yes/No, File, Email, Date).
3. Add actions (send email, create record, call API, etc.).
4. Add a **Respond to a PowerApp or flow** action to return data to the app.

### Calling a Flow from Power Fx

```
// Fire-and-forget (no return value)
MyFlow.Run(txtName.Text, txtEmail.Text)

// With return value
Set(locResult, MyFlow.Run(txtOrderID.Text))
lblStatus.Text: locResult.status
lblMessage.Text: locResult.message
```

### Passing Complex Data

```
// Pass a JSON string for complex objects
Set(locResult, MyFlow.Run(
    JSON(colSelectedItems, JSONFormat.IgnoreBinaryData)
))

// In the flow, use Parse JSON to deserialize
```

### Handling Flow Responses

The **Respond to a PowerApp or flow** action defines the return schema. Return types:

| Type | Power Fx access |
|------|----------------|
| Text | `flowResult.textOutput` |
| Number | `flowResult.numberOutput` |
| Yes/No | `flowResult.boolOutput` |
| File | `flowResult.fileContent` |

```
// Pattern: call flow and handle result
UpdateContext({locProcessing: true});
Set(locResult, SubmitOrderFlow.Run(
    JSON(colCart, JSONFormat.IgnoreBinaryData),
    gblCurrentUser.Email
));
UpdateContext({locProcessing: false});
If(
    locResult.success,
    Navigate(ConfirmationScreen, ScreenTransition.Fade);
    Notify("Order submitted!", NotificationType.Success),
    Notify("Error: " & locResult.errorMessage, NotificationType.Error)
)
```

## Common Flow Patterns

### Send Email with Attachments

```
// Flow steps:
// 1. PowerApps V2 trigger (inputs: To, Subject, Body, FileName, FileContent)
// 2. Send an email (V2) - Office 365 Outlook
//    - To: triggerBody()['text_1']
//    - Subject: triggerBody()['text_2']
//    - Body: triggerBody()['text_3']
//    - Attachments Name: triggerBody()['text_4']
//    - Attachments Content: triggerBody()['file']
```

### Generate PDF

```
// Flow pattern:
// 1. PowerApps V2 trigger (inputs: record ID)
// 2. Get record from Dataverse
// 3. Populate Word template (Word Online connector)
// 4. Convert to PDF (Word Online connector)
// 5. Save to SharePoint / OneDrive
// 6. Respond to PowerApp with file URL
```

### Bulk Operations

```
// Flow pattern:
// 1. PowerApps V2 trigger (input: JSON array)
// 2. Parse JSON
// 3. Apply to each -> process items
//    (or use Dataverse batch operations for better performance)
// 4. Respond with summary (count processed, errors)
```

### File Upload to SharePoint

```
// In Power Apps:
// Use Add Media button to capture file
// Send to flow as File content

UploadFlow.Run(
    AddMediaButton1.Media,     // file content
    txtFileName.Text,          // file name
    drpFolder.Selected.Value   // target folder
)
```

## Approval Flows

### Basic Approval

```
// Flow steps:
// 1. PowerApps V2 trigger (inputs: Title, Details, ApproverEmail)
// 2. Start and wait for an approval
//    - Type: Approve/Reject - First to respond
//    - Title: triggerBody()['text_1']
//    - Assigned to: triggerBody()['text_3']
//    - Details: triggerBody()['text_2']
// 3. Condition: outcome eq 'Approve'
//    - Yes: update record status to Approved, send confirmation
//    - No: update record status to Rejected, send rejection notice
// 4. Respond to PowerApp with outcome
```

### Multi-Stage Approval

```
// Flow steps:
// 1. PowerApps trigger
// 2. Stage 1: Manager approval (wait for response)
// 3. If approved -> Stage 2: Director approval (wait for response)
// 4. If approved -> Stage 3: Finance approval (wait for response)
// 5. Update record with final status at each stage
// 6. Respond to PowerApp

// Each approval step uses "Start and wait for an approval"
// with Condition actions for branching
```

### Approval with Timeout

```
// Use parallel branch:
// Branch 1: Start and wait for an approval
// Branch 2: Delay (e.g., 7 days) -> Terminate as cancelled

// Configure: Settings -> Timeout on the approval action
// ISO 8601 duration format: PT72H (72 hours), P7D (7 days)
```

## Scheduled and Automated Flows

### Automated Flow (Event-Triggered)

Triggered by Dataverse/SharePoint/other events -- no Power Apps trigger needed, but affects data the app displays.

```
// When a record is created in Dataverse:
// 1. Trigger: When a row is added
// 2. Send welcome email
// 3. Create related records
// 4. Update status

// When a SharePoint item is modified:
// 1. Trigger: When an item is created or modified
// 2. Check conditions
// 3. Send notifications
```

### Scheduled Flow

```
// Recurrence trigger:
// - Every day at 8:00 AM
// - Every Monday at 9:00 AM
// - Every 1st of month

// Common uses:
// - Daily report generation
// - Data cleanup / archival
// - Reminder emails for overdue items
// - Sync data between systems
```

### Power Apps + Scheduled Flow Combo

Pattern: App submits request, scheduled flow processes batch.

```
// App: create a queue record
Patch(ProcessingQueue, Defaults(ProcessingQueue), {
    RequestType: "Report",
    Parameters: JSON({dateRange: locDateRange, department: locDept}),
    Status: "Pending"
})

// Scheduled flow: process pending queue items
// 1. Recurrence: every 15 minutes
// 2. List rows where Status = "Pending"
// 3. Apply to each: process and update Status to "Completed"
```

## Error Handling in Flows

### Try-Catch Pattern

```
// Scope: "Try"
//   - Main actions here
//   - Configure Run After: only on success

// Scope: "Catch" (runs after Try)
//   - Configure Run After: has failed, has timed out, is skipped
//   - Log error details
//   - Send error notification
//   - Respond to PowerApp with error info

// Scope: "Finally" (runs after Catch)
//   - Configure Run After: has succeeded, has failed, has timed out, is skipped
//   - Cleanup actions
```

### Retry Policy

```
// On individual actions -> Settings -> Retry Policy
// Types:
// - None: no retries
// - Fixed interval: retry N times with fixed delay
// - Exponential interval: retry with exponential backoff
// - Default: 4 retries with exponential backoff

// Useful for transient failures (API rate limits, network issues)
```

### Error Response to Power Apps

```
// Respond to PowerApp action in catch scope:
{
    "success": false,
    "errorMessage": "@{actions('Create_Record')?['error']?['message']}",
    "errorCode": "@{actions('Create_Record')?['statusCode']}"
}
```

## Performance and Limits

### Flow Limits

| Limit | Value |
|-------|-------|
| Actions per flow run | 500 (nested loops count) |
| Flow run duration | 30 days max |
| API calls per connection per 24h | ~6,000 (varies by license) |
| Concurrent flow runs | 25 per user (can queue more) |
| Apply to each concurrency | 1-50 (default 20) |

### Optimization Tips

- Use `Select` action to reshape data before passing between actions (reduces payload).
- Enable `Apply to each` concurrency (Settings -> Concurrency Control) for independent iterations.
- Use batch operations (Dataverse `Perform a changeset request`) instead of looping individual creates.
- Avoid deeply nested conditions -- flatten with `Switch` or separate flows.
- Use `Compose` to build complex expressions once, then reference the output.
- For Dataverse, use `Perform a bound/unbound action` for server-side operations.
- Minimize flow-to-app response payload -- return only needed fields.
