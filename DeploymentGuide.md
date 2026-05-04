# Smart Alerts — Deployment Guide

## Prerequisites

Before you begin, confirm you have:

- [ ] Microsoft 365 license with Power Automate (Per User or included in M365)
- [ ] Power Apps license (Per User or included in M365)
- [ ] Copilot Studio license (standalone or Microsoft 365 Copilot)
- [ ] SharePoint Online site where you want to monitor changes
- [ ] SharePoint site collection admin rights on that site
- [ ] Power Platform environment (default or dedicated)
- [ ] Teams channel for alert delivery (optional)

---

## Step 1 — Create the SharePoint lists

### AlertSubscriptions list

1. Navigate to your SharePoint site
2. Create a new list named `AlertSubscriptions`
3. Add the following columns (all column names are case-sensitive):

| Column Name | Type | Settings |
|---|---|---|
| SubscriberEmail | Single line of text | Required |
| SubscriberName | Single line of text | |
| SiteURL | Single line of text | Required |
| ListOrLibraryID | Single line of text | Required |
| ListOrLibraryName | Single line of text | Required |
| MonitorType | Choice | Options: List Item, Document Library |
| DeliveryMethod | Choice | Options: Email, Teams, Both |
| TeamsChannelID | Single line of text | |
| TriggerCondition | Choice | Options: Any Change, Specific Columns, Keyword Match |
| WatchedColumns | Multiple lines of text | |
| Keywords | Multiple lines of text | |
| IsActive | Yes/No | Default: Yes |
| CreatedDate | Date and Time | |
| LastModified | Date and Time | |

4. Set list permissions: all users should have Contribute access

### AlertLog list

1. Create a new list named `AlertLog`
2. Add the following columns:

| Column Name | Type | Settings |
|---|---|---|
| SubscriberEmail | Single line of text | |
| SiteURL | Single line of text | |
| ListOrLibraryName | Single line of text | |
| ItemID | Number | |
| ItemTitle | Single line of text | |
| ChangedBy | Single line of text | |
| ChangedByEmail | Single line of text | |
| ChangeTimestamp | Date and Time | |
| ChangeType | Choice | Options: Created, Modified, Deleted, Uploaded, Checked In, Checked Out |
| ChangedFields | Multiple lines of text | |
| BeforeValues | Multiple lines of text | |
| AfterValues | Multiple lines of text | |
| AISummary | Multiple lines of text | |
| DeliveryStatus | Choice | Options: Sent, Failed, Partial |
| DeliveryMethod | Choice | Options: Email, Teams, Both |
| ErrorDetails | Multiple lines of text | |

3. Set list permissions: users should have Read access to their own items only (use item-level permissions)

---

## Step 2 — Configure Copilot Studio

1. Open [Copilot Studio](https://copilotstudio.microsoft.com)
2. Create a new agent or open an existing one
3. Create a new topic named `GenerateAlertSummary`
4. Set the trigger to **None** (this topic is called programmatically, not conversationally)
5. Add a **Generative answers** node with the prompt from `copilot/SmartAlerts_Topic.yaml`
6. Add input variables as defined in the YAML file
7. Configure the output variable `AlertSummary`
8. **Publish** the agent
9. Note the **HTTP endpoint URL** from the agent settings — you'll need this for the flows

---

## Step 3 — Import the Power Automate flows

### List Monitor flow

1. Open [Power Automate](https://make.powerautomate.com)
2. Select **My flows** → **Import** → **Import Package**
3. Upload `flows/SmartAlerts_ListMonitor.zip` (export from the flow definition)
4. Map connections:
   - SharePoint connection → your tenant's SharePoint connection
   - Office 365 Outlook connection → your mail connection
   - Microsoft Teams connection → your Teams connection
5. After import, open the flow and update:
   - **Trigger:** Set your SharePoint site URL
   - **AlertSubscriptions lookup:** Set the site URL and list name
   - **Copilot Studio HTTP action:** Paste the HTTP endpoint URL from Step 2
   - **AlertLog logging:** Set the site URL and list name
6. Save and **Turn on** the flow

### Library Monitor flow

Repeat the above process for `flows/SmartAlerts_LibraryMonitor.zip`

---

## Step 4 — Import the Power Apps

1. Open [Power Apps](https://make.powerapps.com)
2. Select **Apps** → **Import canvas app**
3. Upload `powerapps/SmartAlerts_AdminUI.msapp`
4. Map data connections to your SharePoint site and lists
5. Open the app in edit mode and update the SharePoint site URL variable:
   ```powerfx
   Set(varSiteURL, "https://yourtenant.sharepoint.com/sites/yoursite")
   ```
6. **Save** and **Publish** the app
7. Share the app with your users

---

## Step 5 — Test the deployment

1. Open the Smart Alerts app
2. Create a test subscription:
   - Site: your test site
   - List: any list with a few items
   - Delivery: Email
   - Trigger: Any Change
3. Go to the monitored list and modify an item
4. Wait up to 5 minutes for the flow to trigger
5. Confirm you receive an email with an AI-generated summary
6. Check the AlertLog list to confirm the entry was logged

---

## Troubleshooting

| Issue | Likely cause | Fix |
|---|---|---|
| Flow doesn't trigger | List trigger not connected to correct site/list | Re-check trigger configuration in flow |
| No AI summary in alert | Copilot Studio HTTP endpoint unreachable | Verify endpoint URL and authentication |
| "AISummary is empty" in log | Copilot Studio returned empty response | Check the topic prompt and test in Copilot Studio directly |
| Email not delivered | Outlook connection expired | Re-authenticate the Outlook connection in flow |
| Teams card not posted | Webhook URL invalid or channel deleted | Update TeamsChannelID in the subscription |
| Power App can't load lists | SharePoint connection permissions | Ensure app connection has access to both AlertSubscriptions and AlertLog |

---

## Governance considerations

- Run the Power Automate flows under a **service account**, not a personal account, to avoid the alert reading "Changed by: Flow Service Account"
- Set up a **retention policy** on the AlertLog list — entries older than 90 days can be archived or deleted
- Review Copilot Studio **AI content moderation settings** to ensure summaries meet your organization's acceptable use policy
- If deploying in a **GovCloud or regulated environment**, confirm your Copilot Studio agent is provisioned in the appropriate sovereign cloud region
- Document the solution architecture and data flows for any required **data privacy impact assessment**

---

## Version history

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial release — list monitoring, document library monitoring, Copilot Studio AI summary, Power Apps admin UI |
