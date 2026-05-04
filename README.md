# Smart-Alerts
Smart Alerts — AI-powered SharePoint notifications using Power Automate and Copilot Studio
# Smart Alerts
### AI-powered SharePoint alerts using Power Automate + Copilot Studio

SharePoint's native alert system is unreliable, limited, and tells you almost nothing useful. Smart Alerts replaces it with an intelligent notification system that monitors SharePoint lists and document libraries, uses Copilot Studio to generate a plain-English summary of what changed and why it matters, and delivers it via email and Microsoft Teams.

---

## What it does

- Monitors SharePoint **list item changes** and **document library updates**
- Passes change data to **Copilot Studio** which generates a human-readable summary
- Delivers alerts via **email** and/or **Teams adaptive card**
- Provides a **Power Apps admin UI** where users manage their own subscriptions — what to watch, how to be notified, and what conditions trigger an alert
- Logs all alerts sent for audit and review

---

## Architecture

```
SharePoint Change
       │
       ▼
Power Automate Flow
  ├── Captures: item name, changed fields, before/after values, changed by, timestamp
  ├── Calls: Copilot Studio topic (via HTTP action)
  │         └── Returns: AI-generated plain-English summary
  └── Delivers: Email + Teams adaptive card
       │
       ▼
Alert Log (SharePoint list or Dataverse)

Power Apps Admin UI
  ├── Manage subscriptions (which lists/libraries to watch)
  ├── Set delivery preferences (email, Teams, or both)
  ├── Set trigger conditions (any change, specific columns, keywords)
  └── View alert history log
```

---

## Components

| Component | Description |
|---|---|
| `flows/SmartAlerts_ListMonitor.json` | Power Automate flow — monitors SharePoint list changes |
| `flows/SmartAlerts_LibraryMonitor.json` | Power Automate flow — monitors document library updates |
| `copilot/SmartAlerts_Topic.yaml` | Copilot Studio topic definition for AI summary generation |
| `powerapps/SmartAlerts_AdminUI.msapp` | Power Apps canvas app for subscription management |
| `datamodel/AlertSubscriptions_Schema.md` | Data model for subscriptions and alert log |
| `docs/DeploymentGuide.md` | Step-by-step deployment instructions |

---

## Prerequisites

- Microsoft 365 license with Power Automate and Power Apps
- Copilot Studio license (or Microsoft 365 Copilot)
- SharePoint Online site with lists/libraries to monitor
- Teams channel for alert delivery (optional)
- Dataverse environment or SharePoint list for subscription storage

---

## Quick Start

1. **Deploy the data model** — Create the AlertSubscriptions list and AlertLog list in SharePoint (see `datamodel/AlertSubscriptions_Schema.md`)
2. **Import the flows** — Import both Power Automate flows and update the SharePoint site connection
3. **Configure Copilot Studio** — Import the topic and connect to your Power Automate flow via the HTTP trigger
4. **Import the Power App** — Import `SmartAlerts_AdminUI.msapp` and connect to your data sources
5. **Test** — Make a change to a monitored list item and confirm you receive an AI-summarized alert

---

## Why this exists

SharePoint's built-in alerts have been broken or limited for years. They don't tell you what specifically changed, they fire inconsistently, and they give you a wall of metadata instead of something a human can actually act on. Smart Alerts fixes that by adding an AI layer that reads the change, understands the context, and writes the alert the way a thoughtful colleague would.

---

## Author

**Jessica Graf** — Platform Architect | M365 & Power Platform  
[linkedin.com/in/jessicagraf](https://linkedin.com/in/jessicagraf) | [jgraf30.github.io](https://jgraf30.github.io)
