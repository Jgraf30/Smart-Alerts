# Smart Alerts — Power Automate Flow
# Flow Name: SmartAlerts_ListMonitor
# Trigger: When an item is created or modified in SharePoint

## Flow Overview

This flow monitors a SharePoint list for changes, retrieves the subscriber's
preferences, calls Copilot Studio to generate an AI summary, and delivers
the alert via email and/or Teams.

---

## Trigger

**Connector:** SharePoint  
**Action:** When an item is created or modified  
**Site Address:** [Configured per deployment]  
**List Name:** [Configured per deployment — or use a variable for multi-list support]

---

## Step 1 — Get previous item values (for before/after comparison)

**Action:** Get changes for an item or a file (properties only)  
**Site Address:** @triggerBody()?['odata.metadata']  
**List Name:** @triggerBody()?['odata.type']  
**ID:** @triggerBody()?['ID']  
**Since:** @triggerOutputs()?['headers']['x-ms-file-last-modified']

*Capture which fields changed and store before/after values as JSON.*

---

## Step 2 — Look up active subscriptions for this list

**Action:** Get items (SharePoint — AlertSubscriptions list)  
**Filter Query:**
```
ListOrLibraryID eq '@{triggerBody()?['ParentListId']}' and IsActive eq 1
```

---

## Step 3 — Apply to each subscription

**Action:** Apply to each  
**Input:** @outputs('Get_items')?['body/value']

### Step 3a — Check trigger condition

**Action:** Condition  

**If TriggerCondition = "Any Change":**  
→ Continue to Step 3b

**If TriggerCondition = "Specific Columns":**  
→ Check if any WatchedColumns appear in the ChangedFields list  
→ If yes: Continue to Step 3b  
→ If no: Exit this iteration (do not send alert)

**If TriggerCondition = "Keyword Match":**  
→ Check if any Keywords appear in the item's Title or changed field values  
→ If yes: Continue to Step 3b  
→ If no: Exit this iteration (do not send alert)

### Step 3b — Build the change payload

**Action:** Compose  
**Inputs:**
```json
{
  "ItemTitle": "@{triggerBody()?['Title']}",
  "ListName": "@{triggerBody()?['odata.etag']}",
  "ChangeType": "@{if(equals(triggerOutputs()?['headers']['x-ms-trigger-type'], 'Created'), 'Created', 'Modified')}",
  "ChangedBy": "@{triggerBody()?['Editor']?['DisplayName']}",
  "ChangeTimestamp": "@{utcNow()}",
  "ChangedFields": "@{string(body('Get_changes_for_an_item_or_a_file')?['value'])}",
  "BeforeValues": "@{string(body('Get_changes_for_an_item_or_a_file')?['previousValues'])}",
  "AfterValues": "@{string(body('Get_changes_for_an_item_or_a_file')?['currentValues'])}",
  "SiteURL": "@{triggerBody()?['odata.metadata']}",
  "ItemURL": "@{concat(items('Apply_to_each')?['SiteURL'], '/lists/', items('Apply_to_each')?['ListOrLibraryName'], '/DispForm.aspx?ID=', triggerBody()?['ID'])}"
}
```

### Step 3c — Call Copilot Studio for AI summary

**Action:** HTTP  
**Method:** POST  
**URI:** [Your Copilot Studio topic HTTP endpoint]  
**Headers:**
```json
{
  "Content-Type": "application/json",
  "Authorization": "Bearer @{body('Get_token')?['access_token']}"
}
```
**Body:** @{outputs('Compose_change_payload')}

**Parse Response:**  
**Action:** Parse JSON  
**Schema:** Extract `AlertSummary` string from response body

### Step 3d — Deliver alert

#### If DeliveryMethod = Email or Both:

**Action:** Send an email (V2) — Office 365 Outlook  
**To:** @{items('Apply_to_each')?['SubscriberEmail']}  
**Subject:** Smart Alert: @{triggerBody()?['Title']} was @{outputs('Change_type')} in @{items('Apply_to_each')?['ListOrLibraryName']}  
**Body (HTML):**
```html
<div style="font-family:Segoe UI,Arial,sans-serif;max-width:600px;padding:24px;">
  <h2 style="color:#0078d4;margin-bottom:4px;">Smart Alert</h2>
  <p style="color:#666;font-size:13px;margin-top:0;">
    @{items('Apply_to_each')?['ListOrLibraryName']} · @{formatDateTime(utcNow(), 'dddd, MMMM d yyyy h:mm tt')}
  </p>
  <hr style="border:none;border-top:1px solid #e0e0e0;margin:16px 0;"/>
  <p style="font-size:15px;line-height:1.6;color:#333;">
    @{body('Parse_Copilot_Response')?['AlertSummary']}
  </p>
  <hr style="border:none;border-top:1px solid #e0e0e0;margin:16px 0;"/>
  <p style="font-size:13px;color:#666;">
    Changed by: <strong>@{triggerBody()?['Editor']?['DisplayName']}</strong><br/>
    Item: <a href="@{outputs('Item_URL')}">@{triggerBody()?['Title']}</a>
  </p>
  <p style="font-size:11px;color:#999;margin-top:24px;">
    Smart Alerts · Manage your subscriptions in the Smart Alerts app
  </p>
</div>
```

#### If DeliveryMethod = Teams or Both:

**Action:** Post adaptive card in a chat or channel — Microsoft Teams  
**Post as:** Flow bot  
**Post in:** Channel  
**Team:** @{items('Apply_to_each')?['TeamsChannelID']}  
**Adaptive Card:**
```json
{
  "type": "AdaptiveCard",
  "version": "1.4",
  "body": [
    {
      "type": "TextBlock",
      "text": "Smart Alert",
      "weight": "Bolder",
      "size": "Medium",
      "color": "Accent"
    },
    {
      "type": "TextBlock",
      "text": "{{AlertSummary}}",
      "wrap": true,
      "spacing": "Medium"
    },
    {
      "type": "FactSet",
      "facts": [
        { "title": "Changed by", "value": "{{ChangedBy}}" },
        { "title": "List", "value": "{{ListName}}" },
        { "title": "When", "value": "{{ChangeTimestamp}}" }
      ]
    }
  ],
  "actions": [
    {
      "type": "Action.OpenUrl",
      "title": "View Item",
      "url": "{{ItemURL}}"
    }
  ]
}
```

### Step 3e — Log the alert

**Action:** Create item (SharePoint — AlertLog list)  
**Fields:**
- Title: Smart Alert — @{triggerBody()?['Title']}
- SubscriberEmail: @{items('Apply_to_each')?['SubscriberEmail']}
- ListOrLibraryName: @{items('Apply_to_each')?['ListOrLibraryName']}
- ItemTitle: @{triggerBody()?['Title']}
- ChangedBy: @{triggerBody()?['Editor']?['DisplayName']}
- ChangeTimestamp: @{utcNow()}
- ChangeType: @{outputs('Change_type')}
- AISummary: @{body('Parse_Copilot_Response')?['AlertSummary']}
- DeliveryStatus: Sent
- DeliveryMethod: @{items('Apply_to_each')?['DeliveryMethod']}

---

## Error Handling

Wrap Steps 3c and 3d in a **Scope** action with a **Configure run after** set to handle failures:

- If Copilot Studio call fails: Log error, send a fallback alert without AI summary
- If email delivery fails: Log error with error details, attempt Teams delivery if configured
- If Teams delivery fails: Log error with error details

---

## Notes

- This flow triggers on **every** list item change across the monitored site
- The subscription lookup in Step 2 filters to only active subscriptions for the specific list
- To support multiple lists from a single flow, use the `ParentListId` from the trigger to match subscriptions
- Run this flow with a service account to avoid "Changed by = Flow Bot" scenarios
