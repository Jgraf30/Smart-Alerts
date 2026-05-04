# Smart Alerts — Power Apps Admin UI
# App Name: SmartAlerts_AdminUI
# Type: Canvas App

## Overview

The Smart Alerts Admin UI is a Power Apps canvas app that gives users
full control over their alert subscriptions without needing to touch
a SharePoint list directly. It also provides administrators a view
of all subscriptions and the alert history log.

---

## Screens

### Screen 1: Home / My Subscriptions

**Purpose:** Landing screen showing the current user's active subscriptions.

**Layout:**
- Header: "Smart Alerts" title + user's display name (Office365Users.MyProfile().DisplayName)
- "+ New Subscription" button (navigates to Screen 2)
- Gallery of current user's subscriptions, showing:
  - List/Library name
  - Site name
  - Delivery method (Email / Teams / Both) as icon badges
  - Trigger condition
  - Active/Inactive toggle
  - Edit button → Screen 2
  - Delete button → confirmation dialog → delete record

**Data source:** AlertSubscriptions (filtered to current user's email)

**Key formulas:**
```powerfx
// Load current user's subscriptions
ClearCollect(
    colMySubscriptions,
    Filter(
        AlertSubscriptions,
        SubscriberEmail = User().Email,
        IsActive = true
    )
)

// Toggle active status
Patch(
    AlertSubscriptions,
    ThisItem,
    {IsActive: !ThisItem.IsActive}
)
```

---

### Screen 2: New / Edit Subscription

**Purpose:** Create a new subscription or edit an existing one.

**Layout — Step 1: Choose what to monitor**

- Site URL input field
  - On change: call SharePoint connector to validate site and load available lists/libraries
- Dropdown: Select a list or document library
  - Populated from: `SharePoint.GetLists(SiteURL).value`
  - Display: list title | filter out hidden system lists
- Toggle: List Items / Document Library
  - Auto-set based on selected list type

**Layout — Step 2: Choose how to be notified**

- Toggle group: Email / Teams / Both
- If Teams selected:
  - Dropdown: Select Teams channel
  - Populated from: Teams connector - list channels the user belongs to
  - Or: manual webhook URL input field
- Preview: "You will receive alerts via [method] when items change in [list name]"

**Layout — Step 3: Set trigger conditions**

- Radio buttons:
  - Any change (default)
  - Specific columns only
  - Keyword match
- If "Specific columns":
  - Multi-select checkbox list of available columns
  - Populated from: SharePoint.GetListColumns(SiteURL, ListID)
- If "Keyword match":
  - Tags input for keywords
  - Helper text: "Alert will fire when any of these words appear in changed content"

**Layout — Step 4: Review and save**

- Summary card showing all selected options
- "Save Subscription" button
- On save:

```powerfx
// Save new subscription
If(
    IsBlank(varEditingSubscription),
    // New record
    Patch(
        AlertSubscriptions,
        Defaults(AlertSubscriptions),
        {
            Title: txtSubscriptionName.Text,
            SubscriberEmail: User().Email,
            SubscriberName: Office365Users.MyProfile().DisplayName,
            SiteURL: txtSiteURL.Text,
            ListOrLibraryID: drpListLibrary.Selected.Id,
            ListOrLibraryName: drpListLibrary.Selected.Title,
            MonitorType: If(togMonitorType.Value, "Document Library", "List Item"),
            DeliveryMethod: Switch(togDelivery.Value, "Email", "Email", "Teams", "Teams", "Both"),
            TeamsChannelID: txtTeamsWebhook.Text,
            TriggerCondition: radTrigger.Selected.Value,
            WatchedColumns: Concat(galColumns.AllItems, If(chkColumn.Value, ThisRecord.Name & ",", "")),
            Keywords: txtKeywords.Text,
            IsActive: true,
            CreatedDate: Now()
        }
    ),
    // Update existing record
    Patch(
        AlertSubscriptions,
        varEditingSubscription,
        {
            // same fields as above
        }
    )
);
Navigate(Screen1, ScreenTransition.Fade)
```

---

### Screen 3: Alert History

**Purpose:** View a log of all alerts sent to the current user.

**Layout:**
- Date range filter (default: last 30 days)
- List/Library filter dropdown
- Gallery of alert log entries showing:
  - Date and time
  - Item title (as clickable link)
  - List/Library name
  - Change type badge (Created / Modified / Deleted / Uploaded)
  - Changed by
  - AI Summary (truncated to 2 lines, expand on tap)
  - Delivery status indicator (green = sent, red = failed)

**On tap → expand row:**
- Full AI summary text
- Full list of changed fields
- Before/after values
- Link to view item in SharePoint

**Data source:** AlertLog (filtered to current user's email, sorted by ChangeTimestamp descending)

**Key formulas:**
```powerfx
// Load alert history with filters
ClearCollect(
    colAlertHistory,
    SortByColumns(
        Filter(
            AlertLog,
            SubscriberEmail = User().Email,
            ChangeTimestamp >= datepickerStart.SelectedDate,
            ChangeTimestamp <= datepickerEnd.SelectedDate,
            Or(
                IsBlank(drpListFilter.Selected),
                ListOrLibraryName = drpListFilter.Selected.Value
            )
        ),
        "ChangeTimestamp",
        Descending
    )
)
```

---

### Screen 4: Admin View (role-gated)

**Purpose:** Administrators can view and manage all subscriptions across all users.

**Access control:**
```powerfx
// Gate access to admin screen
If(
    !IsEmpty(
        Filter(
            'SmartAlerts_Admins',  // SharePoint list of admin emails
            AdminEmail = User().Email
        )
    ),
    Navigate(Screen4, ScreenTransition.None),
    Notify("You don't have access to this area.", NotificationType.Error)
)
```

**Layout:**
- All subscriptions gallery (all users, all lists)
- Filter by: user, list, delivery method, active/inactive
- Bulk actions: deactivate selected, delete selected
- Alert log across all users
- Summary metrics:
  - Total active subscriptions
  - Alerts sent today / this week / this month
  - Delivery failure rate
  - Most watched lists

---

## Theming

Use the default Microsoft Fluent theme or the organization's custom theme.

Recommended color palette:
- Primary: #0078D4 (Microsoft Blue)
- Success: #107C10 (Green — for sent status)
- Warning: #FFB900 (Yellow — for partial delivery)
- Error: #D13438 (Red — for failed delivery)
- Background: #F3F2F1 (Light grey)
- Surface: #FFFFFF

---

## Connections Required

| Connector | Purpose |
|---|---|
| SharePoint | Read/write AlertSubscriptions and AlertLog lists; enumerate site lists |
| Office 365 Users | Get current user's display name and email |
| Microsoft Teams | Enumerate channels for subscription setup |
| Office 365 Outlook | (Optional) Send test alert from within the app |

---

## Accessibility Notes

- All interactive elements have accessible labels set
- Color is not used as the only indicator of status (icons + text alongside)
- Gallery items are keyboard navigable
- Minimum touch target size: 44x44px for mobile use
