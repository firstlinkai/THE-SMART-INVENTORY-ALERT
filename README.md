# 🏨 Smart Inventory Alert
### *Telegram-Powered Supply Chain Automation for Resort Operations*







***

## 📋 Table of Contents

- [Overview](#-overview)
- [The Problem It Solves](#-the-problem-it-solves)
- [Tech Stack](#-tech-stack)
- [Workflow Architecture](#-workflow-architecture)
- [How Staff Uses It](#-how-staff-uses-it)
- [Intelligent Threshold System](#-intelligent-threshold-system)
- [Weekly Shopping List](#-weekly-shopping-list)
- [Owner Confirmation Flow](#-owner-confirmation-flow)
- [Google Sheets Setup](#-google-sheets-setup)
- [n8n Blueprint Setup](#-n8n-blueprint-setup)
- [Credentials Required](#-credentials-required)
- [Placeholder Reference](#-placeholder-reference)
- [Workflow Node Map](#-workflow-node-map)
- [Item Alias Support](#-item-alias-support)
- [Activity Logging](#-activity-logging)
- [Error Handling](#-error-handling)
- [Extending the System](#-extending-the-system)

***

## 🧭 Overview

**Smart Inventory Alert** is a lightweight, no-ERP operational tool designed for small resort businesses to manage critical consumable supplies — ice, charcoal, drinking water, soft drinks — using nothing more than a Telegram message.

It bridges the communication gap between on-site staff and the resort owner by automating stock tracking, threshold monitoring, and weekend shopping preparation — all without requiring staff to learn any software.

> *"Staff texts a command. Google Sheets updates. The owner gets alerted. That's the entire interface."*

***

## 🚨 The Problem It Solves

**Operational Snaps** — the nightmare scenario where a resort runs out of basic amenities (ice for drinks, charcoal for grilling, drinking water) during peak weekend hours when resupply is difficult.

Traditional fixes fail because:
- ERP software is too complex for seasonal resort staff
- Verbal updates are forgotten or inconsistent
- Manual spreadsheet entry requires technical access and training
- The owner gets no advance warning before weekends

This system professionalizes supply-chain communication with zero training required for staff.

***

## 🛠 Tech Stack

| Layer | Tool | Purpose |
|---|---|---|
| **Staff Interface** | Telegram Bot | Input commands for stock updates |
| **Workflow Engine** | n8n | Orchestrates all automation logic |
| **Intelligence Layer** | n8n Code Nodes (JavaScript) | Parsing, threshold logic, Friday multiplier |
| **Database** | Google Sheets | Live inventory ledger + activity log |
| **Owner Notifications** | Telegram (Owner Bot) | Alerts, shopping list, confirmation callbacks |

***

## 🗺 Workflow Architecture

The system runs **three independent automation pipelines**:

```
┌─────────────────────────────────────────────────────────────────┐
│  PIPELINE 1: Real-Time Staff Stock Update (Event-Triggered)     │
│                                                                 │
│  Telegram Staff Bot → Parse Message → Validate Command          │
│    → Read Inventory → Compute New Stock → Update Sheet          │
│    → Log Activity → Reply to Staff → Check Threshold            │
│    → [If Low] → Alert Owner via Telegram                        │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PIPELINE 2: Weekly Shopping List (Thursday 4PM Schedule)       │
│                                                                 │
│  Thursday 4PM Cron → Read Full Inventory                        │
│    → Build Shopping List → Send to Owner via Telegram           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│  PIPELINE 3: Owner Confirmation (Callback-Triggered)            │
│                                                                 │
│  Owner Clicks Confirm Button → Handle Callback                  │
│    → Reply Confirmation to Owner                                │
└─────────────────────────────────────────────────────────────────┘
```

### Schema Diagram

Below is the actual n8n workflow canvas layout for reference:

```
[Telegram Staff Bot] → [Parse Staff Message] → [Is Valid Command?]
                                                   ├── FALSE → [Reply Invalid Command]
                                                   └── TRUE  → [Read Inventory Sheet]
                                                                   → [Compute New Stock]
                                                                       → [Item Found?]
                                                                           ├── FALSE → [Reply Item Not Found]
                                                                           └── TRUE  → [Update Inventory Sheet]
                                                                                           → [Log to Activity Sheet]
                                                                                               → [Build Staff Reply]
                                                                                                   → [Reply to Staff]
                                                                                                       → [Needs Owner Alert?]
                                                                                                           └── TRUE → [Build Owner Alert]
                                                                                                                          → [Alert Owner (Low Stock)]

[Thursday 4PM Schedule] → [Read Full Inventory] → [Build Shopping List] → [Send Weekly List to Owner]

[Owner Confirmation Callback] → [Handle Owner Callback] → [Reply Owner Confirmation]
```

***

## 💬 How Staff Uses It

Staff only need to know **two commands**, sent to the Staff Telegram Bot:

### Command 1: Log Usage
```
/used <quantity> <item>
```
Deducts the given quantity from the current stock.

**Examples:**
```
/used 5 ice         → Deducts 5 from Ice stock
/used 3 water       → Deducts 3 from Drinking Water
/used 2 charcoal    → Deducts 2 from Charcoal
```

### Command 2: Set Stock Level
```
/stock <item> <quantity>
```
Sets the current stock to an exact quantity (e.g., after a manual restock).

**Examples:**
```
/stock charcoal 10   → Sets Charcoal stock to 10
/stock ice 25        → Sets Ice stock to 25
/stock soda 15       → Sets Soft Drinks stock to 15
```

### Staff Confirmation Reply

After a valid command, staff receives immediate confirmation:

```
✅ Stock Updated!
Item: Ice
Previous Stock: 20 bags
New Stock: 15 bags
Updated by: Juan
```

If stock drops below threshold, the reply also includes a warning emoji:
```
⚠️ LOW STOCK ALERT: Ice is now at 3 bags (Min: 5)
```

***

## 🧠 Intelligent Threshold System

The system goes beyond simple min-threshold alerts. It applies a **predictive Friday multiplier** to account for higher weekend demand.

### How It Works

```javascript
// Default threshold check
const isLowStock = newStock <= minThreshold;

// Friday predictive logic: raise effective threshold by 30%
const isFriday = new Date().getDay() === 5;
const effectiveThreshold = isFriday 
  ? Math.ceil(minThreshold * 1.3) 
  : minThreshold;
```

| Day | Min Threshold (Example) | Effective Threshold | Why |
|---|---|---|---|
| Mon–Thu | 5 bags | 5 bags | Normal ops |
| **Friday** | 5 bags | **7 bags** | +30% for weekend demand |

### Alert Levels

| Condition | Description | Action |
|---|---|---|
| `isLowStock` | Stock ≤ effective threshold | Owner gets a low stock alert |
| `isCritical` | Stock ≤ 50% of effective threshold | Marked as critical in the alert |

### Owner Alert Message (Example)

```
🚨 LOW STOCK ALERT

Item: Ice
Current Stock: 3 bags
Min Threshold: 5 bags
Status: ⚠️ CRITICAL

Updated by: Maria
Time: 2025-01-17 14:32:00
```

The alert includes an inline Telegram button for owner acknowledgment.

***

## 📋 Weekly Shopping List

Every **Thursday at 4:00 PM**, n8n automatically:

1. Reads the full inventory from Google Sheets
2. Compares each item's current stock against its `Weekend Target` column
3. Computes how many units need to be purchased
4. Sends a formatted shopping list to the owner

### Sample Shopping List Message

```
📋 WEEKEND PREP ALERT
Thursday Shopping List

Based on current stock levels, prepare the following:

🧊 Ice: BUY 15 bags (Current: 10 | Target: 25)
🔥 Charcoal: BUY 5 packs (Current: 3 | Target: 8)
💧 Drinking Water: BUY 10 cases (Current: 5 | Target: 15)
🥤 Soft Drinks: BUY 12 bottles (Current: 8 | Target: 20)

Total Items: 4 | Ready for Weekend: NO

[✅ Confirm Receipt]
```

The **Confirm Receipt** button triggers Pipeline 3, logging the owner's acknowledgment.

***

## ✅ Owner Confirmation Flow

When the owner taps **Confirm Receipt** on any Telegram alert or shopping list:

1. Telegram sends a **callback query** to the `Owner Confirmation Callback` trigger node
2. The `Handle Owner Callback` code node processes the callback data
3. `Reply Owner Confirmation` sends an acknowledgment back to the owner

This creates a **two-way communication loop** — the system knows the owner has seen and acknowledged the alert.

***

## 📊 Google Sheets Setup

Create a Google Sheets document with **two tabs/sheets**:

### Sheet 1: `Inventory`

This is the live inventory ledger. Create the following exact column headers in **Row 1**:

| Column | Header | Description |
|---|---|---|
| A | `Item` | Item name (e.g., Ice, Charcoal) |
| B | `Current Stock` | Live quantity on hand |
| C | `Min Threshold` | Low stock trigger level |
| D | `Weekend Target` | Target quantity before weekends |
| E | `Unit` | Unit of measurement (bags, cases, etc.) |
| F | `Last Updated` | Timestamp of last stock change |
| G | `Last Updated By` | Staff name from Telegram |

**Sample Data:**

| Item | Current Stock | Min Threshold | Weekend Target | Unit | Last Updated | Last Updated By |
|---|---|---|---|---|---|---|
| Ice | 20 | 5 | 25 | bags | 2025-01-16 | Maria |
| Charcoal | 8 | 3 | 10 | packs | 2025-01-16 | Juan |
| Drinking Water | 12 | 4 | 15 | cases | 2025-01-15 | Staff |
| Soft Drinks | 15 | 6 | 20 | bottles | 2025-01-15 | Staff |

> ⚠️ **Column headers must match exactly** (case-sensitive). The n8n nodes reference these headers by name.

### Sheet 2: `Activity Log`

This sheet auto-populates via the `Log to Activity Sheet` node. Create the following headers in Row 1:

| Column | Header |
|---|---|
| A | `Timestamp` |
| B | `Staff Name` |
| C | `Staff ID` |
| D | `Item` |
| E | `Action` |
| F | `Quantity` |
| G | `Previous Stock` |
| H | `New Stock` |

No data needs to be pre-populated — n8n appends every transaction automatically.

***

## 🔧 n8n Blueprint Setup

### Step 1: Import the Workflow

1. Open your n8n instance
2. Go to **Workflows → Import from File**
3. Upload `RRE-AUTO-02-_-Smart-Inventory-Alert.json`
4. The workflow will appear with all nodes pre-configured

### Step 2: Set Up Credentials

Before activating, create the following credentials in **n8n Settings → Credentials**:

#### Telegram API (Staff Bot)
- Credential Type: `Telegram API`
- Name it: `Telegram Staff Bot`
- API Token: Token from [@BotFather](https://t.me/botfather) for your staff-facing bot

#### Telegram API (Owner Bot)
- Credential Type: `Telegram API`
- Name it: `Telegram Owner Bot`
- API Token: Token from [@BotFather](https://t.me/botfather) for your owner-facing bot

> You can use a single bot for both staff and owner if preferred — just use the same credential in all Telegram nodes.

#### Google Sheets OAuth2
- Credential Type: `Google Sheets OAuth2 API`
- Follow n8n's Google OAuth2 setup guide to authorize your Google account

### Step 3: Replace Placeholder Values

Search for and replace the following in the workflow (or directly in the node parameters):

| Placeholder | Replace With |
|---|---|
| `YOUR_GOOGLE_SHEET_ID_HERE` | Your Google Sheet ID (from the URL) |
| `YOUR_OWNER_CHAT_ID` | The owner's Telegram Chat ID |

> **How to find your Google Sheet ID:** Open your Google Sheet. The URL looks like: `https://docs.google.com/spreadsheets/d/`**`1BxiMVs0XRA5nFMdKvBdBZjgmUUqptlbs74OgVE2upms`**`/edit` — the bold section is the ID.

> **How to find your Telegram Chat ID:** Message [@userinfobot](https://t.me/userinfobot) on Telegram. It will reply with your Chat ID.

### Step 4: Assign Credentials to Nodes

After creating credentials, assign them to the corresponding nodes:

| Node | Credential Type | Which Credential |
|---|---|---|
| `Telegram Staff Bot Trigger` | Telegram API | Staff Bot credential |
| `Reply Invalid Command` | Telegram API | Staff Bot credential |
| `Reply Item Not Found` | Telegram API | Staff Bot credential |
| `Reply to Staff` | Telegram API | Staff Bot credential |
| `Alert Owner (Low Stock)` | Telegram API | Owner Bot credential |
| `Send Weekly Shopping List to Owner` | Telegram API | Owner Bot credential |
| `Reply Owner Confirmation` | Telegram API | Owner Bot credential |
| `Read Inventory Sheet` | Google Sheets OAuth2 | Sheets credential |
| `Update Inventory Sheet` | Google Sheets OAuth2 | Sheets credential |
| `Log to Activity Sheet` | Google Sheets OAuth2 | Sheets credential |
| `Read Full Inventory` | Google Sheets OAuth2 | Sheets credential |

### Step 5: Activate the Workflow

Toggle the workflow to **Active** in the top-right of the n8n editor. The Telegram webhook and Thursday cron schedule will begin listening immediately.

***

## 🔑 Credentials Required

| Credential | Type | How to Get |
|---|---|---|
| Staff Telegram Bot Token | `Telegram API` | Create bot via [@BotFather](https://t.me/botfather) |
| Owner Telegram Bot Token | `Telegram API` | Create bot via [@BotFather](https://t.me/botfather) |
| Google Sheets OAuth2 | `Google Sheets OAuth2 API` | Via Google Cloud Console + n8n OAuth setup |

***

## 📌 Placeholder Reference

All values you must replace before going live:

```
YOUR_GOOGLE_SHEET_ID_HERE  → Google Sheets document ID
YOUR_OWNER_CHAT_ID         → Owner's Telegram numeric chat ID
```

Find these in multiple nodes. Use **Ctrl+F** (or n8n's search) to locate all instances.

***

## 🗂 Workflow Node Map

A complete reference of every node in the workflow:

| # | Node Name | Type | Function |
|---|---|---|---|
| 1 | `Telegram Staff Bot Trigger` | Telegram Trigger | Listens for staff messages |
| 2 | `Parse Staff Message` | Code (JS) | Extracts command, item, quantity, and staff info |
| 3 | `Is Valid Command?` | IF | Routes valid vs. invalid commands |
| 4 | `Reply Invalid Command` | Telegram | Sends usage instructions back to staff |
| 5 | `Read Inventory Sheet` | Google Sheets | Fetches current inventory data |
| 6 | `Compute New Stock` | Code (JS) | Calculates new stock, applies Friday multiplier |
| 7 | `Item Found?` | IF | Routes found vs. not-found items |
| 8 | `Reply Item Not Found` | Telegram | Notifies staff of unknown item |
| 9 | `Update Inventory Sheet` | Google Sheets | Writes new stock, timestamp, staff name |
| 10 | `Log to Activity Sheet` | Google Sheets | Appends transaction to audit log |
| 11 | `Build Staff Reply` | Code (JS) | Constructs confirmation message for staff |
| 12 | `Reply to Staff` | Telegram | Sends stock confirmation to staff |
| 13 | `Needs Owner Alert?` | IF | Checks if low-stock threshold is breached |
| 14 | `Build Owner Alert Message` | Code (JS) | Formats owner alert with details |
| 15 | `Alert Owner (Low Stock)` | Telegram | Sends alert to owner with confirm button |
| 16 | `Thursday 4PM Schedule` | Schedule Trigger | Fires every Thursday at 4:00 PM |
| 17 | `Read Full Inventory` | Google Sheets | Reads all inventory for shopping list |
| 18 | `Build Shopping List` | Code (JS) | Computes what needs to be purchased |
| 19 | `Send Weekly Shopping List to Owner` | Telegram | Sends formatted shopping list to owner |
| 20 | `Owner Confirmation Callback` | Telegram Trigger | Listens for owner button clicks |
| 21 | `Handle Owner Callback` | Code (JS) | Processes callback data |
| 22 | `Reply Owner Confirmation` | Telegram | Acknowledges owner's confirmation |

***

## 🔤 Item Alias Support

The `Parse Staff Message` node handles **case-insensitive aliases** so staff can type naturally:

| What Staff Types | Normalized To |
|---|---|
| `ice`, `ices` | `Ice` |
| `charcoal`, `coal` | `Charcoal` |
| `water`, `drinking water` | `Drinking Water` |
| `softdrink`, `soft drink`, `soft drinks`, `soda` | `Soft Drinks` |

New aliases can be added in the `itemAliases` object inside the `Parse Staff Message` code node:

```javascript
const itemAliases = {
  ice: 'Ice',
  ices: 'Ice',
  charcoal: 'Charcoal',
  coal: 'Charcoal',
  water: 'Drinking Water',
  'drinking water': 'Drinking Water',
  softdrink: 'Soft Drinks',
  'soft drink': 'Soft Drinks',
  'soft drinks': 'Soft Drinks',
  soda: 'Soft Drinks',
  // Add your own aliases below:
  // beer: 'Beer',
};
```

***

## 📓 Activity Logging

Every stock update — whether a `/used` deduction or `/stock` override — is automatically appended to the **Activity Log** sheet with the following data:

- `Timestamp` — ISO 8601 datetime of the transaction
- `Staff Name` — First name from Telegram profile
- `Staff ID` — Telegram user ID (for accountability)
- `Item` — Normalized item name
- `Action` — `used` or `stock`
- `Quantity` — Quantity involved in the action
- `Previous Stock` — Stock before the update
- `New Stock` — Stock after the update

This creates a complete, tamper-evident audit trail for owner review.

***

## ⚠️ Error Handling

The system gracefully handles the two most common staff input errors:

### Invalid Command Format

If staff sends a message that doesn't match `/used` or `/stock` syntax:

```
❌ Invalid command, Juan! Use:
/used <qty> <item>  — to log usage
/stock <item> <qty>  — to set current stock

Example: /used 5 ice
Example: /stock charcoal 10
```

### Item Not Found

If the item name (even after alias normalization) doesn't match any row in the Inventory sheet:

```
⚠️ Item "beer" not found in inventory. Please check the item name.
```

In both cases, the workflow stops gracefully and no Google Sheets writes occur.

***

## 🔌 Extending the System

### Add New Items

Simply add a new row in the `Inventory` Google Sheet with all required columns. No changes to n8n are needed — the workflow reads items dynamically.

### Add New Item Aliases

Edit the `itemAliases` object in the `Parse Staff Message` Code node (see [Item Alias Support](#-item-alias-support)).

### Change the Weekly Schedule

Edit the `Thursday 4PM Schedule` node. Click the node, change the cron schedule to your preferred day and time.

### Add More Alert Recipients

Duplicate the `Alert Owner (Low Stock)` node and set a different `chatId` to notify multiple people (e.g., a manager + the owner).

### Adjust the Friday Multiplier

In the `Compute New Stock` Code node, change the `1.3` multiplier to any value:

```javascript
// Change 1.3 to increase/decrease the Friday buffer
const effectiveThreshold = isFriday ? Math.ceil(minThreshold * 1.3) : minThreshold;
```

***

## 📁 Repository Structure

```
RRE-AUTO-02-Smart-Inventory-Alert/
├── README.md                                      ← This file
├── RRE-AUTO-02-_-Smart-Inventory-Alert.json       ← n8n workflow blueprint
└── RRE-AUTO-02-_-Smart-Inventory-Alert-Schema.jpg ← Workflow canvas screenshot
```

***
<img width="1722" height="641" alt="RRE-AUTO-02 _ Smart Inventory Alert Schema" src="https://github.com/user-attachments/assets/34bc5c63-8de4-4260-a7f0-4088f061ed9e" />

## 🏷 Project Metadata

| Field | Value |
|---|---|
| **Project Code** | RRE-AUTO-02 |
| **Project Name** | Smart Inventory Alert |
| **Category** | Supply Chain / Resort Operations |
| **Automation Platform** | n8n |
| **Version** | 1.0 |
| **Status** | Ready to Deploy |

***

*Built with n8n · Google Sheets · Telegram*
