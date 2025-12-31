# BMI SupplyAutomate - Wholesale Invoicing

## Overview

The **Wholesale Invoicing** system automates the receipt and processing of vendor invoices from wholesalers (Essendant and SP Richards). When a wholesaler sends an electronic invoice, the system imports the invoice data, validates it against received purchase orders, and can automatically post purchase invoices with accurate costs, quantities, and charges. This ensures that payables accurately reflect what was actually invoiced by the wholesaler, rather than what was originally ordered.

### Business Problem Solved

**Manual invoice processing challenges:**

- Manually entering wholesale invoices is time-consuming and error-prone
- Invoice amounts may differ from original PO amounts due to pricing changes
- Freight and handling charges must be manually allocated
- Risk of paying incorrect amounts
- Difficult to reconcile invoice details with received goods
- No automated validation against actual shipments

**Wholesale invoicing automation provides:**

- **Automatic invoice import** - Electronic invoices downloaded from wholesaler systems
- **Validation against receipts** - Compares invoice quantities/costs to what was received
- **Automatic charge allocation** - Freight, handling, and other charges posted to correct GL accounts
- **Cost accuracy** - Purchase invoices reflect actual wholesaler costs, not estimated PO costs
- **Three-way matching** - Validates PO → Receipt → Invoice consistency
- **Duplicate detection** - Prevents processing the same invoice twice

---

## Key Concepts

### What is Wholesale Invoicing?

**Wholesale Invoicing** is the automated process of:

1. Importing electronic invoice files from wholesalers
2. Matching invoice lines to purchase orders and receipts
3. Validating quantities and costs match what was received
4. Automatically posting purchase invoices in Business Central
5. Allocating freight and handling charges to appropriate GL accounts

### Invoice File Types

#### **Essendant Invoice Files**

- **Invoice Header** - Top-level invoice information (invoice number, dates, customer)
- **Invoice Detail** - Line-level item details (item, quantity, unit cost)
- **Invoice Summary** - Totals and charges (freight, drop ship, handling, taxes)
- **Format**: XML files via ELink API
- **Transmission**: Retrieved via web service

#### **SP Richards Invoice Files**

- **EZINV Invoice Header** - Header information (invoice number, PO number, dates)
- **EZINV Invoice Detail 1** - Line items (item, quantity, price)
- **EZINV Invoice Detail 2** - Additional line details
- **EZINV Invoice Totals** - Summary totals and charges
- **Format**: Fixed-width text files
- **Transmission**: Retrieved via FTP server

### Three-Way Match Validation

The system performs "three-way matching" to ensure consistency:

| Document | Validation Check |
| -------- | ---------------- |
| **Purchase Order** | What you ordered |
| **Purchase Receipt** | What you received (from ASN or manual receipt) |
| **Vendor Invoice** | What the wholesaler is billing |

**Validation Rules:**

- Invoice quantity must NOT exceed received quantity
- Invoice unit cost must match received unit cost
- Only received items can be invoiced
- Unit cost differences prevent automatic posting

### Wholesale Invoice Data Flags

Purchase orders have two key flags:

| Flag | Description | When Set |
| ---- | ----------- | -------- |
| **Wholesale Invoice Data Available** | Invoice file(s) have been imported for this PO | Set automatically when invoice imported |
| **Post Invoice with Wholesale Data** | User wants to auto-post using wholesale data | Set automatically (user can disable) |

**Workflow:**

```text
Invoice Import → Sets "Data Available" = true
                → Sets "Post with Data" = true (default)
                → User can toggle "Post with Data" = false to post manually
```

---

## How Wholesale Invoicing Works

### Automated Processing (Job Queue)

#### **Stage 1: Invoice Import & Translation**

##### **Job Queue Configuration**

- **Parameter**: `'INVOICE-IMPORT'`
- **Frequency**: Every hour (off-peak) or daily (overnight)
- **Purpose**: Downloads and imports new invoice files from wholesalers

##### **Process Flow**

**For Essendant:**

1. **Download Invoice Files**
   - Connects to Essendant ELink API
   - Retrieves invoice files from specified date range
   - Stores in raw Essendant invoice tables

2. **Parse Invoice Data**
   - Reads Invoice Header records (invoice number, dates, PO number)
   - Reads Invoice Detail records (items, quantities, unit costs)
   - Reads Invoice Summary records (totals, charges)

3. **Match to Purchase Orders**
   - Uses "Purchase Order Number" from invoice header
   - Finds corresponding BC purchase order
   - Sets `WholesaleInvDataAvailableBMU = true` on PO
   - Sets `PostInvWithWholesaleDataBMU = true` on PO

4. **Duplicate Detection**
   - Checks if invoice number already exists
   - Marks duplicate invoices (prevents re-processing)

**For SP Richards:**

1. **Download Invoice Files**
   - Connects to SP Richards FTP server
   - Downloads EZINV invoice files
   - Stores in SPR EZINV tables

2. **Parse Invoice Data**
   - Reads Invoice Header (INVCNO, CPONO, IDATE)
   - Reads Invoice Detail 1 (items, quantities, prices)
   - Reads Invoice Detail 2 (additional details)
   - Reads Invoice Totals (freight, handling, taxes)

3. **Match to Purchase Orders**
   - Uses CPONO (customer PO number) field
   - Matches to BC purchase order number
   - Sets wholesale invoice flags on PO

4. **Duplicate Detection**
   - Checks if INVCNO already imported
   - Marks `Duplicate Invoice = true` if found

---

#### **Stage 2: Automatic Invoice Posting**

This stage occurs when a user posts a purchase order with "Post Invoice with Wholesale Data" enabled.

##### **Trigger Conditions**

Automatic invoice posting is triggered when:

1. User posts a Purchase Order (Post → Receive and Invoice)
2. `WholesaleInvDataAvailableBMU = true` (invoice data exists)
3. `PostInvWithWholesaleDataBMU = true` (user wants auto-posting)
4. Unposted invoice exists for this PO

##### **Process Flow**

###### **Step 1: Retrieve Wholesale Invoice**

- Finds invoice record matching the purchase order
- For Essendant: Matches "Purchase Order Number"
- For SP Richards: Matches CPONO field
- Skips duplicate invoices
- Confirms invoice hasn't been posted yet

###### **Step 2: Build Validation Buffer**

Creates a temporary buffer (WholesaleInvtItemBuffer) containing:

**From Wholesale Invoice:**

- Item Number
- Wholesale Quantity (what wholesaler invoiced)
- Wholesale Line Amount (what wholesaler charges)
- Wholesale Unit Cost (per unit cost from invoice)

**From Purchase Order:**

- Item Number
- Quantity (what was received but not yet invoiced)
- Line Amount (BC calculated amount)
- Unit Cost (BC unit cost)

**Calculated Differences:**

- Quantity Difference = BC Quantity - Wholesale Quantity
- Line Amount Difference = BC Amount - Wholesale Amount
- Unit Cost Difference = BC Unit Cost - Wholesale Unit Cost

**Validation Flag:**

- `OK to Invoice = true` if:
  - Unit Cost Difference = 0 (costs match)
  - Quantity Difference >= 0 (received at least as much as invoiced)

###### **Step 3: Validate Invoice Data**

System removes any lines where `OK to Invoice = false`

**If validation fails (buffer empty after filtering):**

- Automatic posting is canceled
- Standard BC posting continues (user must manually enter quantities)
- User should review discrepancies

**Common validation failures:**

- Unit cost on invoice differs from unit cost on receipt
- Wholesaler invoiced more quantity than was received
- Item numbers don't match

###### **Step 4: Set Purchase Header Fields**

Updates purchase order header:

**For Essendant:**

- Posting Date = Invoice Date
- Vendor Invoice No. = Invoice Number
- Due Date = Terms Net Due Date

**For SP Richards:**

- Posting Date = IDATE (invoice date)
- Document Date = IDATE
- Vendor Invoice No. = INVCNO

###### **Step 5: Set Quantity to Invoice**

For each validated line in buffer:

1. Sets "Qty. to Invoice" on purchase lines
2. Matches items by item number
3. Distributes wholesale quantity across multiple PO lines if needed
4. If wholesale quantity > line quantity, moves to next line

**Example:**

```text
Wholesale Invoice: Item ABC, Qty 100

Purchase Order:
  Line 10: Item ABC, Qty Rcd Not Invoiced = 60 → Set Qty to Invoice = 60
  Line 20: Item ABC, Qty Rcd Not Invoiced = 40 → Set Qty to Invoice = 40

Result: All 100 units invoiced across 2 lines
```

###### **Step 6: Insert Wholesaler Charges**

System automatically creates purchase lines for charges:

**Essendant Charges:**

- Freight Charge
- Drop Ship Charge
- Wrap & Label Charge
- Small/Minimum Order Charge
- Sales Tax Charge
- Desk Top Delivery Charge
- Gift Wrap Charge
- Inside Delivery Charge
- Stenciling Charge
- Shrink Wrap Charge
- Azerty Processing Fee
- Azerty Small Order Charge
- Additional Misc. Charges (1, 2, 3)

**SP Richards Charges:**

- Freight Allowance
- Sales Tax Charge
- Handling Charge
- Freight Charge
- Misc. Charge

**Charge Insertion Logic:**

- Creates G/L Account purchase line
- Uses GL account from Purchases & Payables Setup
- Description = charge type
- Quantity = 1
- Direct Unit Cost = charge amount
- Skips if amount = 0 or GL account not configured

###### **Step 7: Post Invoice**

- System commits changes
- Standard BC posting continues
- Purchase invoice posted with:
  - Quantities from wholesale invoice
  - Costs from wholesale invoice
  - Charges allocated to GL accounts

---

### Manual Operations

#### **View Wholesale Invoice Data**

**From Purchase Order:**

1. Open **Purchase Order**
2. Look for "Wholesale" FastTab
3. Check `Wholesale Invoice Data Available` field
4. Click **Wholesale → Invoice** action
5. Opens invoice details page

**Essendant Invoice View:**

- Shows invoice header information
- Lists all detail lines
- Displays summary totals and charges
- Can drill into related PO

**SP Richards Invoice View:**

- Shows EZINV invoice header
- Lists detail lines
- Displays invoice totals
- Can navigate to related documents

#### **Disable Automatic Invoice Posting**

If you want to manually post the invoice:

1. Open **Purchase Order**
2. Go to "Wholesale" FastTab
3. Uncheck `Post Invoice with Wholesale Data`
4. Post invoice normally using BC quantities/costs

**When to disable:**

- Invoice costs differ from receipt costs (price disputes)
- Invoice quantities don't match receipts (discrepancies)
- Need to manually adjust quantities or costs
- Want to delay invoice posting

#### **Manual Invoice Import**

**For SP Richards:**

1. Navigate to: **Search → "SP Richards Invoice List"**
2. Click **Import Invoices** action
3. System requests date range
4. Downloads all invoices within range
5. Processes and matches to purchase orders

**For Essendant:**

- Typically automated via job queue
- Manual import available from invoice list pages if needed

---

## Setup and Configuration

### 1. Purchases & Payables Setup

Configure GL accounts for wholesaler charges:

Navigate to: **Search → "Purchases & Payables Setup"**

#### **Essendant Charge Accounts**

| Field | Description | Required |
| ----- | ----------- | -------- |
| **ESS Freight Acct** | GL Account for freight charges | Yes |
| **ESS Drop Ship Acct** | GL Account for drop ship charges | Yes |
| **ESS Wrap & Label Acct** | GL Account for wrap/label charges | Yes |
| **ESS Small Min Order Acct** | GL Account for small order charges | Yes |
| **ESS Sales Tax Acct** | GL Account for sales tax | Yes |
| **ESS Desk Top Delivery Acct** | GL Account for desktop delivery | No |
| **ESS Gift Wrap Acct** | GL Account for gift wrap | No |
| **ESS Inside Delivery Acct** | GL Account for inside delivery | No |
| **ESS Stenciling Acct** | GL Account for stenciling | No |
| **ESS Shrink Wrap Acct** | GL Account for shrink wrap | No |
| **ESS Azerty Processing Fee Acct** | GL Account for Azerty processing | No |
| **ESS Azerty Small Order Acct** | GL Account for Azerty small order | No |
| **ESS Additional Misc 1 Acct** | GL Account for misc charge 1 | No |
| **ESS Additional Misc 2 Acct** | GL Account for misc charge 2 | No |
| **ESS Additional Misc 3 Acct** | GL Account for misc charge 3 | No |

#### **SP Richards Charge Accounts**

| Field | Description | Required |
| ----- | ----------- | -------- |
| **SPR Freight Allowance Acct** | GL Account for freight allowance | Yes |
| **SPR Sales Tax Acct** | GL Account for sales tax | Yes |
| **SPR Handling Chge Acct** | GL Account for handling charges | Yes |
| **SPR Freight Chge Acct** | GL Account for freight charges | Yes |
| **SPR Misc Chge Acct** | GL Account for miscellaneous charges | No |

**Setup Guidelines:**

- Use expense accounts for charges
- Use liability accounts for sales tax
- Freight allowance may be contra-expense (negative charge)
- If GL account not configured, charge is skipped

---

### 2. Bumpdown Setup (Credentials)

Wholesaler API/FTP credentials configured in **Bumpdown Setup**:

Navigate to: **Search → "Bumpdown Setup"**

#### **Essendant ELink Settings** (for Invoicing)

| Field | Description | Required |
| ----- | ----------- | -------- |
| **ELink User ID** | Essendant ELink API username | Yes |
| **ELink Password** | Essendant ELink API password (masked) | Yes |

*Note: ELink handles both invoices and ASN data.*

#### **SP Richards Settings** (for Invoicing)

| Field | Description | Required |
| ----- | ----------- | -------- |
| **FTP Username** | FTP server username | Yes |
| **FTP Password** | FTP server password (masked) | Yes |

---

### 3. Job Queue Setup

Create job queue entry for automated invoice import:

Navigate to: **Search → "Job Queue Entries"**

#### **Invoice Import Job Queue Entry**

| Field | Value |
| ----- | ----- |
| **Object Type to Run** | Codeunit |
| **Object ID to Run** | 71552664 (JobQueue-BumpdownBMU) |
| **Parameter String** | `INVOICE-IMPORT` |
| **Status** | Ready |
| **Recurring Job** | Yes |
| **No. of Minutes between Runs** | 60-120 (1-2 hours) or daily |

**Recommended Schedule:**

- **High volume dealers**: Every 1-2 hours during business hours
- **Low volume dealers**: Once daily (overnight)
- **Month-end**: More frequent (every 30-60 minutes)

**Considerations:**

- Invoices typically arrive daily or weekly
- More frequent imports provide faster visibility
- Balance frequency with system load
- Consider running during off-peak hours

---

### 4. SPR Account Configuration

For SP Richards invoicing, configure account settings:

Navigate to: **Search → "SP Richards Accounts"**

| Field | Description |
| ----- | ----------- |
| **Include In Batch Invoice Download** | Include this account in automated invoice imports |
| **Last Invoice Request From Date** | Starting date for next invoice download |
| **Last Invoice Request To Date** | Ending date for last invoice download |

**Usage:**

- System tracks last import date automatically
- Next import starts from "To Date" + 1 day
- Can manually adjust dates if needed
- Uncheck "Include in Batch" to exclude account

---

## User Guide

### Daily Operations

#### **Monitor Imported Invoices**

**Morning Routine:**

1. Navigate to **Essendant Invoice List** or **SP Richards Invoice List**
2. Filter: `SystemCreatedAt = TODAY` (or recent date)
3. Review newly imported invoices
4. Check "Duplicate Invoice" field (should be false)

**What to look for:**

- Invoice count matches expected shipments
- No duplicate invoices
- Invoices matched to purchase orders
- Date ranges are current

#### **Review Purchase Orders Ready to Invoice**

1. Navigate to **Purchase Orders**
2. Filter: `Wholesale Invoice Data Available = true`
3. Filter: `Status = Released` or `Open`
4. Review orders with available invoice data
5. Post invoices (Receive and Invoice)

#### **Post Purchase Invoice with Wholesale Data**

**Standard Process:**

1. Open **Purchase Order**
2. Verify "Wholesale Invoice Data Available" = checked
3. Verify "Post Invoice with Wholesale Data" = checked
4. Click **Post → Receive and Invoice**
5. System automatically:
   - Sets quantities from wholesale invoice
   - Sets costs from wholesale invoice
   - Adds freight and handling charges
   - Posts purchase invoice

**Result:**

- Purchase invoice created with vendor invoice number
- Quantities match wholesale invoice
- Costs match wholesale invoice
- Charges allocated to correct GL accounts

#### **View Invoice Details Before Posting**

1. Open **Purchase Order**
2. Check "Wholesale Invoice Data Available" = true
3. Click **Wholesale → Invoice** action
4. Review invoice header, lines, and charges
5. Compare to purchase order quantities/costs
6. Decide whether to post with wholesale data

---

### Common Tasks

#### **Manually Import Invoices (SP Richards)**

1. Navigate to: **SP Richards Invoice List**
2. Click **Actions → Import Invoices**
3. System displays date range dialog
4. Accept default dates (from last import) or adjust
5. Click **OK**
6. System downloads and processes invoices
7. Review imported invoices in list

**Use when:**

- Job queue is not running
- Need invoices immediately
- Importing historical invoices
- Testing invoice import

#### **Review Invoice Validation Before Posting**

1. Open **Purchase Order** with invoice data available
2. Click **Wholesale → Invoice** action
3. Review invoice details
4. Compare invoice quantities to "Qty. Rcd. Not Invoiced"
5. Compare invoice unit costs to PO unit costs
6. Check for discrepancies

**Red flags:**

- Invoice quantity > Received quantity
- Invoice unit cost ≠ PO unit cost
- Missing invoice lines
- Unexpected charges

**Action if discrepancies found:**

1. Investigate with wholesaler
2. Uncheck "Post Invoice with Wholesale Data"
3. Post invoice manually with correct amounts
4. Document reason for manual adjustment

#### **Post Invoice Manually (Ignore Wholesale Data)**

If invoice validation fails or manual adjustments needed:

1. Open **Purchase Order**
2. Uncheck "Post Invoice with Wholesale Data"
3. Manually set "Qty. to Invoice" on lines
4. Manually create charge lines if needed
5. Post invoice using standard BC process

**Result:**

- Invoice posted with manual quantities/costs
- Wholesale invoice data ignored
- User controls all invoice details

#### **View Posted Purchase Invoices**

1. Navigate to: **Posted Purchase Invoices**
2. Filter by Vendor = Essendant or SP Richards
3. Review "Vendor Invoice No." (matches wholesale invoice number)
4. Verify amounts and charges
5. Drill into document for details

---

### Special Procedures

#### **Reprocess Duplicate Invoice**

If an invoice was incorrectly marked as duplicate:

1. Navigate to invoice list (Essendant or SP Richards)
2. Find the invoice record
3. Uncheck "Duplicate Invoice" field
4. System will re-enable processing
5. Purchase order flags will update

#### **Clear All Invoice Data (Troubleshooting)**

**⚠️ WARNING: This deletes ALL imported invoice data!**

1. Navigate to: **Codeunit 71552682** (Essendant) or **71552699** (SP Richards)
2. Run codeunit manually (developer task)
3. Or call `DeleteAllInvoiceData()` procedure
4. All invoice tables cleared
5. All PO invoice flags reset to false
6. Reimport invoices from wholesaler

**Use only when:**

- Testing invoice import process
- Major data corruption
- Directed by support

---

## Troubleshooting Guide

### Problem: Invoices Not Importing

**Symptom:** No new invoice records appearing in invoice list pages

**Diagnostic Steps:**

1. **Check Job Queue Status**
   - Navigate to: **Job Queue Entries**
   - Find entry with Parameter = `INVOICE-IMPORT`
   - Verify Status = **Ready** (not "On Hold" or "Error")
   - Check "Last Ready State" timestamp

2. **Check Job Queue Log**
   - Navigate to: **Job Queue Log Entries**
   - Filter by Object ID = 71552664
   - Review recent executions
   - Look for error messages

3. **Verify Credentials**
   - Open **Bumpdown Setup**
   - **Essendant:** Check ELink User ID and Password
   - **SP Richards:** Check FTP Username and Password
   - Test connection if possible

4. **Verify Date Ranges (SP Richards)**
   - Open **SP Richards Accounts**
   - Check "Last Invoice Request From Date" and "To Date"
   - Ensure dates are reasonable
   - Confirm "Include In Batch Invoice Download" is checked

5. **Check Wholesaler Systems**
   - Confirm wholesaler is generating invoices
   - Verify ELink/FTP access is active
   - Contact wholesaler if needed

**Common Causes:**

| Issue | Solution |
| ----- | -------- |
| Expired credentials | Update passwords in Bumpdown Setup |
| Job queue on hold | Set Status = Ready |
| Network/firewall issue | Contact IT to verify connectivity |
| Date range incorrect (SPR) | Adjust dates in SP Richards Account |
| No invoices to import | Normal - wait for invoices |
| Wholesaler system down | Contact wholesaler support |

---

### Problem: Invoice Imported But PO Flags Not Set

**Symptom:** Invoice exists in invoice list but "Wholesale Invoice Data Available" not set on PO

**Diagnostic Steps:**

1. **Verify Purchase Order Exists**
   - Check invoice's "Purchase Order Number" field
   - Search for matching PO in BC
   - Confirm PO number matches exactly

2. **Check Invoice Table Triggers**
   - Essendant: Triggers on `Essendant Invc Header Rec` table
   - SP Richards: Triggers on `SPR EZINV Invoice Header` table
   - Triggers should set flags on OnInsert

3. **Check for Duplicate Invoice**
   - Review "Duplicate Invoice" field
   - Duplicates do not set PO flags
   - May need to unmark as duplicate

4. **Manually Set Flags (Workaround)**
   - Open **Purchase Order**
   - Manually check "Wholesale Invoice Data Available"
   - Manually check "Post Invoice with Wholesale Data"
   - Verify invoice data is valid

**Common Causes:**

| Problem | Resolution |
| ------- | ---------- |
| PO number mismatch | Verify PO number on invoice matches BC PO |
| PO deleted | Cannot process - invoice orphaned |
| Duplicate invoice | Unmark duplicate if incorrect |
| Trigger failed | Manually set flags, report to support |

---

### Problem: Invoice Posting Fails Validation

**Symptom:** User attempts to post with wholesale data, but system reverts to manual posting

**Diagnostic Steps:**

1. **Check Wholesale Invoice Data**
   - Open PO, click **Wholesale → Invoice**
   - Review invoice quantities and unit costs
   - Compare to PO "Qty. Rcd. Not Invoiced" and "Unit Cost"

2. **Identify Validation Failures**
   - Invoice quantity > Received quantity
   - Invoice unit cost ≠ PO unit cost
   - Item numbers don't match

3. **Review Wholesale Invt Item Buffer (Developer)**
   - System builds buffer during posting
   - Lines with `OK to Invoice = false` are removed
   - If all lines removed, posting fails validation

**Common Causes:**

| Issue | Solution |
| ----- | -------- |
| Unit cost mismatch | Contact wholesaler, post manually with correct cost |
| Quantity mismatch | Verify receipt, post with received quantity |
| Item not received | Receive item first, then post invoice |
| Price change | Post manually, update unit cost |

**Workaround:**

1. Uncheck "Post Invoice with Wholesale Data"
2. Post invoice manually
3. Manually set quantities and costs
4. Document reason for manual posting

---

### Problem: Missing Freight or Handling Charges

**Symptom:** Invoice posted but freight/handling charges not included

**Diagnostic Steps:**

1. **Check GL Account Configuration**
   - Navigate to: **Purchases & Payables Setup**
   - Verify charge GL accounts are configured
   - **Essendant:** ESS Freight Acct, ESS Drop Ship Acct, etc.
   - **SP Richards:** SPR Freight Chge Acct, SPR Handling Chge Acct, etc.

2. **Check Invoice Summary Data**
   - View wholesale invoice details
   - Verify charge amounts are non-zero
   - **Essendant:** Check Invoice Summary table
   - **SP Richards:** Check Invoice Totals table

3. **Review Posted Invoice**
   - Open **Posted Purchase Invoice**
   - Look for G/L Account lines
   - Charges should appear as separate lines

**Common Causes:**

| Problem | Resolution |
| ------- | ---------- |
| GL account not configured | Set up charge GL accounts in Purch. & Payables Setup |
| Charge amount = 0 | Normal - wholesaler didn't charge |
| Charge skipped | System skips if GL account blank or amount = 0 |
| Wrong GL account | Update setup, reprocess invoice |

---

### Problem: Duplicate Invoice Detected

**Symptom:** Invoice marked as "Duplicate Invoice = true", PO flags not set

**Diagnostic Steps:**

1. **Verify True Duplicate**
   - Check if invoice number already exists in system
   - Look for posted purchase invoice with same vendor invoice no.
   - Determine if actually a duplicate

2. **Check Original Invoice**
   - If duplicate, find original invoice record
   - Verify original was already processed
   - Confirm no action needed on duplicate

3. **If Not Actually Duplicate**
   - Rare case: Same invoice number used for different PO
   - Or: Original import failed, retry marked as duplicate
   - Need to unmark duplicate flag

**Solutions:**

| Situation | Action |
| --------- | ------ |
| True duplicate | No action needed, system working correctly |
| False duplicate | Uncheck "Duplicate Invoice", reprocess |
| Original failed | Delete failed record, unmark duplicate |
| Wholesaler reused invoice no. | Contact wholesaler, request unique invoice number |

---

### Problem: Invoice Dates Incorrect

**Symptom:** Posted invoice has wrong posting date or document date

**Diagnostic Steps:**

1. **Check Wholesale Invoice Date Fields**
   - **Essendant:** "Invoice Date", "Terms Net Due Date"
   - **SP Richards:** IDATE field
   - Verify dates are reasonable

2. **Check System Date**
   - Posting uses invoice date from wholesale system
   - Not BC system date
   - Confirm date is in valid posting range

3. **Check BC Posting Date Restrictions**
   - Navigate to: **General Ledger Setup**
   - Check "Allow Posting From" and "Allow Posting To"
   - Invoice date must fall within allowed range

**Common Causes:**

| Issue | Solution |
| ----- | -------- |
| Invoice date outside posting range | Adjust posting date range, repost |
| Wholesaler date incorrect | Contact wholesaler, post manually with correct date |
| Date format issue | Verify date parsing, report to support |

---

## Monitoring and Reporting

### Key Pages

#### **1. Essendant Invoice List**

**Search:** "Essendant Invoice List"

**Shows:**

- All imported Essendant invoices
- Invoice number, date, PO number
- Duplicate invoice flag
- Summary totals

**Key Filters:**

- Invoice Date = specific date range
- Purchase Order Number = specific PO
- Duplicate Invoice = true/false
- Bill to Customer Number = specific customer

**Use for:**

- Daily monitoring of imports
- Verifying invoice import success
- Identifying duplicates
- Researching specific invoices

---

#### **2. SP Richards Invoice List**

**Search:** "SP Richards Invoice List" or "SPR EZINV Invoice"

**Shows:**

- All imported SPR invoices
- INVCNO, CPONO, IDATE
- Duplicate invoice flag
- Customer number

**Key Filters:**

- IDATE = invoice date range
- CPONO = customer PO number
- CUSTNO = SP Richards account number
- Duplicate Invoice = true/false

**Use for:**

- Daily monitoring
- Manual import triggering
- Invoice research
- Duplicate detection

---

#### **3. Purchase Order (Wholesale FastTab)**

**Shows:**

- Wholesale Order (yes/no)
- Wholesale Invoice Data Available (yes/no)
- Post Invoice with Wholesale Data (yes/no)

**Actions:**

- **Wholesale → Invoice** - View invoice details

**Use for:**

- Quick check if invoice data exists
- Enable/disable automatic posting
- Access invoice details
- Verify PO is wholesale order

---

#### **4. Posted Purchase Invoices**

**Filter by:**

- Buy-from Vendor No. = Essendant or SP Richards
- Posting Date = date range
- Vendor Invoice No. = specific invoice number

**Shows:**

- All posted invoices
- Vendor invoice number (matches wholesale invoice no.)
- Posting date from wholesale invoice
- Total amounts including charges

**Use for:**

- Confirming invoice was posted
- Verifying vendor invoice number
- Reviewing posted charges
- Reconciliation with wholesaler statements

---

### Health Check Procedures

#### **Daily Health Check**

Run each morning:

1. **Check Invoice Import Status**

   ```text
   Invoice List page
   Filter: SystemCreatedAt = TODAY or YESTERDAY
   Result: Should have records if shipments occurred
   ```

2. **Verify Job Queue Execution**

   ```text
   Job Queue Log Entries
   Filter: Object ID = 71552664, Parameter = INVOICE-IMPORT
   Result: Recent successful execution
   ```

3. **Review Pending Invoices**

   ```text
   Purchase Orders
   Filter: Wholesale Invoice Data Available = true, Status <> Invoiced
   Result: Orders ready to invoice
   ```

4. **Check for Duplicates**

   ```text
   Invoice Lists
   Filter: Duplicate Invoice = true, Created Date = recent
   Result: Should be zero or very few
   ```

---

#### **Weekly Health Check**

1. **Invoice Volume Analysis**
   - Count invoices imported this week
   - Compare to previous weeks
   - Verify volume matches shipment activity

2. **Validation Success Rate**
   - Count invoices posted automatically
   - Count invoices posted manually (wholesale data disabled)
   - Calculate % posted automatically
   - Investigate if < 90% automatic

3. **Duplicate Rate**
   - Count duplicate invoices
   - Calculate % of total imports
   - Investigate if > 5%

4. **Charge Allocation Review**
   - Sample posted purchase invoices
   - Verify freight and handling charges present
   - Check GL accounts are correct

---

### Regular Maintenance Tasks

#### **Daily**

- Review imported invoices
- Post purchase invoices with wholesale data
- Address any validation failures
- Clear duplicates if false positives

#### **Weekly**

- Run health check procedures
- Review automatic vs manual posting ratio
- Check charge allocation accuracy
- Archive old import logs

#### **Monthly**

- Reconcile posted invoices with wholesaler statements
- Review GL account allocations for charges
- Verify all invoices were processed
- Document any manual adjustments

#### **Quarterly**

- Review GL account setup for charges
- Update credentials if needed
- Test invoice import process
- Train staff on procedures

---

## Best Practices

### Setup Recommendations

1. **GL Account Configuration**
   - Set up all charge GL accounts before importing invoices
   - Use separate accounts for each charge type
   - Configure both Essendant and SP Richards accounts
   - Review chart of accounts with accounting team

2. **Job Queue Scheduling**
   - Run invoice import during off-peak hours
   - Consider daily imports (overnight) for most dealers
   - Increase frequency during month-end
   - Monitor job queue execution times

3. **Credential Management**
   - Store credentials securely in BC
   - Test after any password changes
   - Document credential renewal schedule
   - Keep backup contact at wholesaler

4. **Duplicate Detection**
   - Trust system duplicate detection
   - Investigate false positives
   - Don't manually unmark duplicates without verification
   - Contact wholesaler if frequent duplicates

---

### Operational Guidelines

1. **Daily Invoice Processing**
   - Review imported invoices each morning
   - Post invoices promptly (same day if possible)
   - Don't let invoices accumulate
   - Address validation failures immediately

2. **Validation Failure Handling**
   - Always investigate before posting manually
   - Document reason for manual posting
   - Contact wholesaler for cost discrepancies
   - Keep log of manual adjustments

3. **Three-Way Match Discipline**
   - Verify PO → Receipt → Invoice consistency
   - Don't post invoices before receiving goods
   - Question quantity or cost differences
   - Maintain audit trail

4. **Manual Posting**
   - Use sparingly (< 10% of invoices)
   - Document reason in posting description
   - Review manual postings weekly
   - Trend analysis to identify systemic issues

---

### Performance Optimization

1. **Import Frequency Tuning**
   - Start with daily imports
   - Increase if volume warrants
   - Balance speed vs system load
   - Consider business hours vs overnight

2. **Data Retention**
   - Archive old invoice records (> 1 year)
   - Keep summary statistics
   - Maintain audit trail
   - Balance storage vs query performance

3. **Validation Optimization**
   - Ensure PO unit costs are accurate
   - Receive goods promptly
   - Keep inventory costs current
   - Minimize cost update lag

---

### Common Pitfalls to Avoid

❌ **Don't:**

- Post invoices without verifying wholesale data
- Ignore validation failures
- Manually post without investigation
- Delete invoices marked as duplicates
- Skip daily monitoring
- Misconfigure GL accounts for charges

✅ **Do:**

- Review invoice details before posting
- Investigate all validation failures
- Document reasons for manual posting
- Monitor import success daily
- Keep GL accounts properly configured
- Reconcile with wholesaler statements monthly

---

## Field Reference

### Purchase Header Extensions

Fields added to Purchase Header table for wholesale invoicing.

| Field Name | Type | Description | Usage Notes |
| ---------- | ---- | ----------- | ----------- |
| WholesaleOrderBMU | Boolean | Indicates PO created via bumpdown | Auto-set by bumpdown |
| WholesaleInvDataAvailableBMU | Boolean | Invoice file(s) imported for this PO | Auto-set on invoice import |
| PostInvWithWholesaleDataBMU | Boolean | User wants to post with wholesale data | User can toggle on/off |
| WholesaleInvDataSourceBMU | Enum | Essendant or SPRichards | Auto-set on invoice import |

---

### Essendant Invc Header Rec Table

Main invoice header table for Essendant invoices.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Unique file transmission identifier |
| Sequence Number | Integer | Sequence within transmission |
| Invoice Date | Date | Date of invoice |
| Invoice Number | Code[7] | Essendant invoice number |
| Purchase Order Number | Text[22] | Customer's PO number |
| Ship to Name | Text[35] | Ship-to customer name |
| Ship to Address 1/2 | Text[35] | Ship-to address lines |
| Ship to City | Text[26] | Ship-to city |
| Ship to State | Text[2] | Ship-to state code |
| Ship to Zip Code | Text[15] | Ship-to postal code |
| Terms Net Due Date | Date | Payment due date |
| Duplicate Invoice | Boolean | Duplicate detection flag |

---

### Essendant Invc Detail Table

Line-level invoice details for Essendant.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Links to header |
| Sequence Number | Integer | Header sequence number |
| Line Number on Invoice | Integer | Invoice line number |
| Line Number on PO | Integer | PO line number |
| Quantity Shipped | Integer | Quantity on invoice |
| Item Number Ordered | Code[15] | Item number |
| Item Shipped Description | Text[25] | Item description |
| Unit Cost to Customer | Decimal | Unit cost |
| Line Amount | Decimal | Total line amount |

---

### Essendant Invc Summary Table

Invoice totals and charges for Essendant.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Links to header |
| Sequence Number | Integer | Header sequence number |
| Total Invoice Amount | Decimal | Total invoice |
| Discounted Amount Due | Decimal | Amount after discount |
| Freight Charge | Decimal | Freight amount |
| Drop Ship Charge | Decimal | Drop ship fee |
| Wrap & Label Charge | Decimal | Wrap/label fee |
| Small/Minimum Order Charge | Decimal | Small order fee |
| Sales Tax Charge | Decimal | Sales tax |
| Desk Top Delivery Charge | Decimal | Desk delivery fee |
| Gift Wrap Charge | Decimal | Gift wrap fee |
| Inside Delivery Charge | Decimal | Inside delivery fee |
| Stenciling Charge | Decimal | Stenciling fee |
| Shrink Wrap Charge | Decimal | Shrink wrap fee |
| Azerty Processing Fee | Decimal | Azerty processing |
| Azerty Small Order Charge | Decimal | Azerty small order |
| Additional Misc. Charge 1/2/3 | Decimal | Miscellaneous charges |

---

### SPR EZINV Invoice Header Table

Main invoice header for SP Richards.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Unique transmission identifier |
| CUSTNO | Code[9] | SP Richards customer number |
| INVCNO | Code[8] | Invoice number |
| IDATE | Date | Invoice date |
| SDATE | Date | Ship date |
| CPONO | Code[20] | Customer PO number |
| Duplicate Invoice | Boolean | Duplicate detection flag |

---

### SPR EZINV Invoice Detail 1 Table

Line-level invoice details for SP Richards.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Links to header |
| INVCNO | Code[8] | Invoice number |
| ITEMNO | Code[15] | Item number (with dashes) |
| PDESC | Code[27] | Product description |
| LFLAG | Code[1] | Line flag ('R' or 'S' = item line) |
| QORDRD | Integer | Quantity ordered |
| QSHPPD | Integer | Quantity shipped/invoiced |
| EPRICE | Decimal | Extended price (line total) |

---

### SPR EZINV Invoice Totals Table

Invoice totals and charges for SP Richards.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| SA Transmission ID | Guid | Links to header |
| INVCNO | Code[8] | Invoice number |
| ITOT03 | Decimal | Freight Allowance |
| ITOT06 | Decimal | Sales Tax |
| ITOT07 | Decimal | Handling Charge |
| ITOT08 | Decimal | Freight Charge |
| ITOT09 | Decimal | Misc. Charge |

---

### WholesaleInvtItemBuffer Table

Temporary buffer used during invoice validation and posting.

| Field Name | Type | Description |
| ---------- | ---- | ----------- |
| Item No. | Code[20] | Item number (key) |
| Wholesale Quantity | Decimal | Qty from wholesaler invoice |
| Quantity | Decimal | Qty received but not invoiced in BC |
| Wholesale Line Amount | Decimal | Amount from wholesaler invoice |
| Line Amount | Decimal | Amount in BC |
| Wholesale Unit Cost | Decimal | Unit cost from wholesaler |
| Unit Cost | Decimal | Unit cost in BC |
| Quantity Difference | Decimal | BC Qty - Wholesale Qty |
| Line Amount Difference | Decimal | BC Amount - Wholesale Amount |
| Unit Cost Difference | Decimal | BC Cost - Wholesale Cost |
| OK to Invoice | Boolean | Validation passed (true = can auto-post) |

---

## Advanced Topics

### Three-Way Match Validation Logic

The system performs sophisticated three-way matching:

**Step 1: Build Buffer**

- Collects data from wholesale invoice (what wholesaler bills)
- Collects data from purchase lines (what BC expects to invoice)
- Uses "Qty. Rcd. Not Invoiced" (what was received but not yet invoiced)

**Step 2: Calculate Differences**

```al
Quantity Difference = BC Quantity - Wholesale Quantity
Unit Cost Difference = BC Unit Cost - Wholesale Unit Cost
```

**Step 3: Validate**

```al
OK to Invoice = true IF:
  - Unit Cost Difference = 0 (exact match required)
  - Quantity Difference >= 0 (received at least what's being invoiced)
```

**Step 4: Filter**

- Removes any lines where OK to Invoice = false
- If buffer is empty after filtering → validation failed → manual posting required

**Why Unit Cost Must Match Exactly:**

- Prevents posting incorrect costs to inventory
- Forces resolution of cost discrepancies
- Maintains cost accuracy
- Audit trail requirement

**Why Quantity Can Be >=:**

- Allows partial invoicing (wholesaler invoices less than received)
- Prevents over-invoicing (wholesaler cannot invoice more than received)
- Supports split shipments

---

### Charge Allocation Algorithm

**Essendant Charges:**

System reads `Essendant Invc Summary` table and creates purchase lines:

```al
For Each Charge Type:
  IF (Charge Amount <> 0) AND (GL Account Configured) THEN
    Create Purchase Line:
      Type = G/L Account
      No. = GL Account from Setup
      Description = Charge Type Name
      Quantity = 1
      Direct Unit Cost = Charge Amount
```

**SP Richards Charges:**

System reads `SPR EZINV Invoice Totals` table (ITOT fields):

- ITOT03 = Freight Allowance
- ITOT06 = Sales Tax
- ITOT07 = Handling
- ITOT08 = Freight
- ITOT09 = Miscellaneous

**Line Numbering:**

- System finds last purchase line
- Adds 10000 to line number
- Ensures charge lines don't conflict with existing lines

---

### Partial Invoicing Scenarios

**Scenario 1: Wholesaler invoices less than received**

```text
Received: 100 units
Wholesaler invoices: 60 units

Result:
- 60 units invoiced automatically
- 40 units remain "Qty. Rcd. Not Invoiced"
- Can invoice remaining 40 later (manual or future wholesale invoice)
```

**Scenario 2: Multiple receipts, single invoice**

```text
Receipt 1: 50 units
Receipt 2: 50 units
Wholesaler invoice: 100 units

Result:
- System invoices across both receipts
- Distributes 100 units to available lines
- All received quantities invoiced
```

**Scenario 3: Wholesaler invoices more than received (VALIDATION FAILS)**

```text
Received: 80 units
Wholesaler invoice: 100 units

Result:
- Validation fails (Quantity Difference = -20, which is < 0)
- OK to Invoice = false
- Manual posting required
- User must investigate discrepancy
```

---

### Cost Matching Requirements

**Why costs must match exactly:**

1. **Inventory Valuation**
   - BC tracks inventory at cost
   - Invoice cost becomes inventory cost
   - Mismatched costs corrupt valuation

2. **Margin Accuracy**
   - Sales prices based on costs
   - Incorrect costs → incorrect margins
   - Profitability reporting affected

3. **Audit Requirements**
   - Must trace cost from PO → Receipt → Invoice
   - Cost changes require documentation
   - Audit trail depends on consistency

**When costs differ:**

- **Price increases**: Wholesaler raised prices after PO created
- **Price decreases**: Promotional pricing applied
- **Data entry errors**: Wrong cost on PO or invoice
- **Currency rounding**: Minor rounding differences

**Resolution:**

1. Contact wholesaler to clarify cost
2. If correct: Update PO unit cost, receive again if needed
3. If incorrect: Request corrected invoice from wholesaler
4. Document resolution
5. Post manually if immediate posting needed

---

### Duplicate Invoice Detection

**Detection Logic:**

**On Invoice Insert:**

```al
SPREZINVInvoiceHeader.SetRange(INVCNO, Rec.INVCNO);
IF not SPREZINVInvoiceHeader.IsEmpty() THEN
  "Duplicate Invoice" := true;
```

**Effect of Duplicate Flag:**

- `Duplicate Invoice = true` → Does NOT set wholesale flags on PO
- `Duplicate Invoice = false` → Sets `WholesaleInvDataAvailableBMU = true`

**Why Duplicates Occur:**

- Wholesaler retransmits invoice files
- System reimports historical date range
- Wholesaler uses non-unique invoice numbers (rare)
- Testing/development data

**Handling Duplicates:**

- System automatically detects and flags
- No user action needed for true duplicates
- If false positive: Unmark duplicate, system will process

---

### Event Subscribers and Integration Points

**OnBeforePostPurchaseDoc Event:**

System subscribes to BC's standard posting codeunit:

```al
[EventSubscriber(ObjectType::Codeunit, Codeunit::"Purch.-Post", 'OnBeforePostPurchaseDoc')]
```

**Purpose:**

- Intercepts posting before BC processes
- Checks if wholesale invoice data should be used
- Modifies purchase header and lines
- Commits changes
- Returns control to BC posting

**Integration Points:**

1. **Standard BC Posting** - Subscribes to OnBeforePostPurchaseDoc
2. **Purchase Header** - Extends with wholesale fields
3. **Purchases & Payables Setup** - Stores GL accounts for charges
4. **Job Queue** - Scheduled import processing
5. **Bumpdown System** - Shares credentials and file management

---

### Extensibility

Custom code can extend wholesale invoicing:

**Add Custom Charges:**

- Extend Purchases & Payables Setup with new GL account fields
- Modify invoice management codeunits to insert custom charges
- Update summary tables if needed

**Add Custom Validation:**

- Subscribe to internal events (if published)
- Add custom logic to WholesaleInvtItemBuffer filling
- Implement additional OK to Invoice checks

**Customize Invoice Import:**

- Extend file parsing logic
- Add custom field mappings
- Integrate with third-party invoice systems

---

## Appendix: Common Scenarios

### Scenario 1: Standard Automatic Invoice Posting

**Situation:** Normal wholesale order, invoice received electronically

**Process:**

1. **Morning:** Bumpdown transmits PO to Essendant
2. **Afternoon:** Essendant ships order and generates invoice
3. **Evening:** Job queue imports invoice file
4. **System Actions:**
   - Creates invoice header, detail, summary records
   - Sets `WholesaleInvDataAvailableBMU = true` on PO
   - Sets `PostInvWithWholesaleDataBMU = true` on PO
5. **Next Morning:** User posts PO (Receive and Invoice)
6. **System Actions:**
   - Validates invoice against receipt
   - Sets quantities from invoice
   - Sets costs from invoice
   - Adds freight and handling charges
   - Posts purchase invoice
7. **Result:** Invoice posted with exact wholesaler quantities, costs, and charges

---

### Scenario 2: Cost Mismatch Requires Manual Posting

**Situation:** Wholesaler invoice cost differs from PO cost

**Example:**

- PO Unit Cost: $10.00
- Receipt Unit Cost: $10.00
- Invoice Unit Cost: $9.50 (promotional discount applied)

**Process:**

1. Invoice imported and matched to PO
2. User attempts to post (Receive and Invoice)
3. **System validation:**
   - Unit Cost Difference = $10.00 - $9.50 = $0.50
   - OK to Invoice = false (cost mismatch)
   - All lines removed from buffer
   - Validation fails
4. **System behavior:**
   - Does NOT automatically set quantities
   - User must manually enter "Qty. to Invoice"
   - Standard BC posting occurs
5. **User actions:**
   - Investigates with wholesaler
   - Confirms $9.50 is correct (promo pricing)
   - Updates PO unit cost to $9.50
   - Receives again (adjusts cost)
   - Posts invoice with wholesale data (now validates)

**Alternative Resolution:**

- User unchecks "Post Invoice with Wholesale Data"
- Manually sets "Qty. to Invoice"
- Manually updates unit cost to $9.50
- Posts invoice
- Documents promotional pricing

---

### Scenario 3: Partial Shipment, Full Invoice Later

**Situation:** Order placed for 100 units, wholesaler ships in two shipments

**Week 1:**

- Shipment 1: 60 units
- ASN processed, receipt posted
- No invoice yet

**Week 2:**

- Shipment 2: 40 units
- ASN processed, receipt posted
- Invoice received for full 100 units

**Invoice Processing:**

1. Invoice imported: 100 units @ $10.00
2. PO shows: Qty. Rcd. Not Invoiced = 100 (60 + 40)
3. User posts PO (Receive and Invoice)
4. **System validation:**
   - Wholesale Quantity = 100
   - BC Quantity = 100
   - Quantity Difference = 0
   - Unit Cost Difference = 0
   - OK to Invoice = true
5. **System posts:**
   - 100 units invoiced
   - All received quantities cleared
   - Single invoice for both shipments

---

### Scenario 4: Multiple POs, Single Invoice (Not Supported)

**Situation:** Wholesaler combines multiple POs on one invoice

**Example:**

- PO-001: 50 units Item A
- PO-002: 50 units Item B
- Invoice 12345: Both POs combined

**Current Behavior:**

- Invoice import matches to PO-001 only (first PO number found)
- PO-002 not linked to invoice
- User must manually post PO-002

**Workaround:**

1. Import invoice (links to PO-001)
2. Post PO-001 with wholesale data
3. For PO-002:
   - No wholesale invoice data available
   - Post manually
   - Use same vendor invoice number
   - Document that invoice covers multiple POs

**Future Enhancement:**

- Multi-PO invoice support
- Invoice splitting across multiple POs
- Enhanced invoice-to-PO matching

---

### Scenario 5: Invoice Arrives Before Receipt

**Situation:** Wholesaler sends invoice before goods arrive

**Process:**

1. **Day 1:** Invoice imported and matched to PO
2. `WholesaleInvDataAvailableBMU = true`
3. **Day 2:** User attempts to post invoice
4. **Problem:** No quantities to invoice yet
5. **System behavior:**
   - Qty. Rcd. Not Invoiced = 0
   - WholesaleInvtItemBuffer.Quantity = 0
   - Quantity Difference = 0 - Wholesale Qty (negative)
   - OK to Invoice = false
   - Validation fails
6. **Day 3:** Goods arrive, receipt posted
7. **Day 4:** User posts invoice
8. **Result:** Now validates successfully, auto-posts

**Best Practice:**

- Always receive goods before invoicing
- Don't invoice until "Qty. Rcd. Not Invoiced" > 0
- Wait for ASN processing to complete

---

## Support and Additional Resources

### Getting Help

1. **Check This Documentation**
   - Review troubleshooting section
   - Follow diagnostic steps
   - Try suggested solutions

2. **Review System Logs**
   - Invoice list pages (check duplicate flags, import dates)
   - Job Queue Log Entries
   - Posted purchase invoices

3. **Contact BMI Support**
   - Website: https://bmiusa.freshdesk.com/support/home
   - Include: Invoice number, PO number, error messages
   - Specify: Wholesaler (Essendant or SP Richards)
   - Attach: Screenshots of invoice data, validation failures

### Related Documentation

- **BUMPDOWN.md** - Purchase order transmission to wholesalers
- **WHOLESALE_ASN.md** - Advance Ship Notice (receipt automation)
- **Wholesale Sourcing** - Vendor priority and sourcing rules
- **Credit Management** - Credit hold and payment terms

### Glossary

| Term | Definition |
| ---- | ---------- |
| **Three-Way Match** | Validation that PO, Receipt, and Invoice agree on quantities and costs |
| **ELink** | Essendant's API for invoice and ASN data |
| **EZINV** | SP Richards electronic invoicing format |
| **Qty. Rcd. Not Invoiced** | BC field showing quantity received but not yet invoiced |
| **Wholesale Invoice Data Available** | Flag indicating invoice file imported for this PO |
| **Post Invoice with Wholesale Data** | Flag controlling automatic vs manual invoice posting |
| **OK to Invoice** | Validation flag on buffer line (true = can auto-post) |
| **Unit Cost Difference** | Difference between BC cost and wholesaler cost |
| **Charge Allocation** | Process of posting freight/handling to GL accounts |
| **Duplicate Invoice** | Invoice number already exists in system |

---

## Document Revision History

| Date | Version | Changes |
| ---- | ------- | ------- |
| 2025-12-23 | 1.0 | Initial documentation - Invoice import, validation, and automated posting |

---

*For questions or issues not covered in this documentation, contact BMI Support at https://bmiusa.freshdesk.com/support/home*
