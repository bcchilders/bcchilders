# BMI SupplyAutomate - Bumpdown System

## Overview
The **Bumpdown System** is an automated purchase order transmission and fulfillment system that sources wholesale items from vendors (Essendant and SP Richards). It analyzes released sales orders, groups items by vendor priority, transmits purchase orders to wholesalers, processes acknowledgements, and handles failures through intelligent vendor priority "bumpdown" logic.

---

## Key Concepts

### What is "Bumpdown"?
When a wholesaler cannot fulfill a line item (rejects, out of stock, etc.), the system automatically retries the next vendor in the priority sequence. This cascading through vendor priorities is called "bumping down" the list until either:
- A vendor accepts the order, OR
- All vendors are exhausted (line becomes a Failed Wholesale Line)

### Transmission Modes
The system operates in three modes:
- **Production** - Live transmissions to wholesaler production endpoints
- **Test** - Transmissions to wholesaler test/sandbox endpoints
- **Internal Test** - Simulated transmissions (no external API calls)

---

## How the Bumpdown Process Works

### Automated Processing (Job Queue)

#### **Step 1: Job Queue Identifies Eligible Orders**
A scheduled job queue (`JobQueue-BumpdownBMU` with parameter `'PO'`) scans for sales orders that meet criteria:

```
Sales Orders eligible for bumpdown:
✓ Document Type = Order
✓ Status = Released
✓ BumpdownInProgressBMU = false
✓ Has at least one line with:
  - Type = Item
  - WholesaleOrderBMU = true
  - Outstanding Quantity > 0
  - Purch. Order Line No. = 0 (not yet linked to PO)
  - Special Order Purch. Line No. = 0
  - FailedWholesaleLineExistsBMU = false
```

#### **Step 2: Line Grouping by Vendor Priority**
The system groups sales lines by their vendor priority sequence:

1. For each line, retrieves `WSAccountPriorityBuffer` based on:
   - **Wholesale Account Group Code** (from sales line)
   - **WS Account Priority Group Code** (from sales line)
   - **Item Number**
   - **Drop Ship requirements** (DropShipOnlyBMU, NoDropShipAllowedBMU, Drop Shipment flag)

2. Lines with identical vendor priority lists receive the same **Bumpdown Group ID**

3. Each group has an ordered list of wholesaler accounts to attempt

**Example:**
```
Group 1: Lines 1000, 2000, 3000 → Try Essendant Account A, then SPR Account B
Group 2: Lines 4000, 5000 → Try SPR Account C, then Essendant Account D
```

#### **Step 3: Transmission to Wholesaler**
For each Bumpdown Group, attempts transmission in vendor priority order:

1. **Creates BumpdownSendReceiveDoc records** for each line:
   - Assigns unique Transmission Group ID
   - Generates new Purchase Order number
   - Sets Line Status = "Not Processed"
   - Records item, quantity, unit cost, vendor info

2. **Transmits based on Transmission Mode**:
   - **Essendant** → UNILINK API (XML-based)
   - **SP Richards** → FTP file transfer
   - **Internal Test** → Simulated acceptance/rejection

3. **Receives response from wholesaler**:
   - Sets Line Status to: Acknowledged, Partially Acknowledged, Failed, or Transmission Error
   - Records Confirmation Code and Description
   - For acknowledged lines: records Quantity Acknowledged and Unit Cost Acknowledged

#### **Step 4: Process Response**

##### **For Acknowledged Lines:**
1. Creates **Purchase Header** (if first line in group):
   - Document Type = Order
   - Vendor = wholesaler account vendor
   - Location matches sales order
   - For drop shipments: copies ship-to information
   - Sets `WholesaleOrderBMU = true`

2. Creates **Purchase Line**:
   - Quantity = Quantity Acknowledged
   - Direct Unit Cost = Unit Cost Acknowledged
   - Links to Sales Line via Special Order or Drop Shipment fields
   - Sets `WholesaleOrderBMU = true`
   - Records `BDSendReceiveDocEntryNoBMU`

3. Updates **Sales Line**:
   - Links to Purchase Order and Line
   - Sets appropriate Purchasing Code
   - Updates unit cost via Load Factor calculation

4. **If Partially Acknowledged** (Qty Acknowledged < Qty Requested):
   - Original sales line updated to acknowledged quantity
   - **New sales line created** with remaining quantity
   - New line has `BumpdownCreatedEntryBMU = true`
   - New line has `SplitFromLineNoBMU` pointing to original
   - New line will retry bumpdown in next cycle

##### **For Failed Lines:**
1. **Tries next vendor** in priority sequence (the bumpdown)
2. If vendor priority list exhausted:
   - Creates **FailedWholesaleLine** record with:
     - Source document information
     - Confirmation Code (reason for failure)
     - Confirmation Description
   - Sales Line marked: `FailedWholesaleLineExistsBMU = true`
   - Line will NOT retry automatically

##### **For Transmission Errors:**
1. Sets **Sales Header** fields:
   - `BumpdownErrorBMU = true`
   - `BumpdownErrorMsgBMU` = error message
2. Creates **Error Message** record for system admin review
3. Line will retry on next job queue cycle

---

### Manual Transmission (On-Demand)

Users can manually trigger bumpdown from the Sales Order page instead of waiting for the scheduled job queue.

#### **How to Manually Transmit**

1. Open the **Sales Order** you want to transmit
2. Ensure order is **Released** status
3. Click **Actions** → **Functions** → **Transmit Order**
4. System performs same validation and transmission as automated process

#### **Manual Transmission Restrictions**

The "Transmit Order" action is **only available when**:
- No scheduled task exists for `JobQueue-BumpdownPOBMU` codeunit
- If a scheduled task is detected, user receives message:
  > "A scheduled task is set up to transmit orders. Please wait for the scheduled task to run or contact your system administrator."

**Purpose:** Prevents conflict between manual and automated transmissions

#### **Manual Transmission Use Cases**
- **Urgent orders** that can't wait for next scheduled cycle
- **Testing/troubleshooting** in development environments
- **One-time transmissions** when job queue is disabled
- **Immediate retry** after fixing failed line issues

---

## Setup and Configuration

### 1. Bumpdown Setup Page

Navigate to: **Search → "Bumpdown Setup"**

#### **General Settings**
| Field | Description |
|-------|-------------|
| **Enabled** | Master switch - must be ON for any bumpdown processing |
| **Transmission Mode** | Production, Test, or Internal Test |
| **Application (client) ID** | Azure App Registration client ID (for API authentication) |
| **Client Secret** | Azure App Registration secret (masked field) |

#### **Essendant Integration - UNILINK (PO Transmission)**
| Field | Description |
|-------|-------------|
| **Unilink User ID** | Essendant UNILINK API username |
| **Unilink Password** | Essendant UNILINK API password (masked) |
| **Transmission ID** | Auto-incrementing transmission sequence number |
| **Order Taker** | 3-character code for order taker identification |
| **Label Type Override** | Optional: override default label type |

#### **SP Richards Integration (PO Transmission)**
| Field | Description |
|-------|-------------|
| **FTP Username** | SP Richards FTP server username |
| **FTP Password** | SP Richards FTP server password (masked) |
| **File Prefix** | Prefix for outbound PO files |
| **User ID** | SP Richards user ID for transactions |
| **Label Text Fields** | Custom label text for Route, Ref. No., Attn., PO No. |

---

### 2. Wholesaler Accounts

#### **Essendant Accounts** (`EssendantAccountBMU`)
Each account represents a ship-from location or account number:
- **Account Code** - Unique identifier
- **Vendor No.** - BC vendor record to use
- **Drop Shipment** - Whether this account supports drop ship
- **Account Number** - Essendant account number
- **Ship From Code** - Essendant warehouse code
- Priority/grouping settings

#### **SP Richards Accounts** (`SPRichardsAccountBMU`)
Each account represents an SP Richards ship-from location:
- **Account Code** - Unique identifier
- **Vendor No.** - BC vendor record to use
- **Drop Shipment** - Whether this account supports drop ship
- **Account Number** - SP Richards account number
- **DUNS Number** - SP Richards DUNS identifier
- Priority/grouping settings

---

### 3. Purchasing Codes

**Required:** Two purchasing codes marked `WholesaleBMU = true`

Navigate to: **Search → "Purchasing Codes"**

| Purchasing Code | Special Order | Drop Shipment | WholesaleBMU |
|----------------|---------------|---------------|--------------|
| WSDROP | ☐ No | ☑ Yes | ☑ Yes |
| WSSPECIAL | ☑ Yes | ☐ No | ☑ Yes |

The system automatically selects the appropriate code based on the wholesaler account's drop shipment setting.

---

### 4. Job Queue Setup

For **automated processing**, create job queue entry:

Navigate to: **Search → "Job Queue Entries"**

**Create new entry:**
- **Object Type to Run:** Codeunit
- **Object ID to Run:** `71552664` (JobQueue-BumpdownBMU)
- **Parameter String:** `PO`
- **Status:** Ready
- **Recurring Job:** ☑ Yes
- **No. of Minutes between Runs:** (e.g., 15 minutes)
- **Earliest Start Date/Time:** Now
- **Job Queue Category Code:** (your category)

**Recommended schedule:** Every 15-30 minutes during business hours

---

### 5. Item Setup - Vendor Priorities

Each item must be configured with vendor priority groups:

Navigate to: **Item Card → SupplyAutomate FastTab**

| Field | Description |
|-------|-------------|
| **Wholesale Account Group Code** | Groups items by wholesale sourcing strategy |
| **WS Account Priority Group Code** | Defines vendor attempt sequence for this item |
| **Drop Ship Only** | Item can ONLY be drop shipped |
| **No Drop Ship Allowed** | Item cannot be drop shipped |

**Vendor priority is determined by:**
- Item's WS Account Priority Group Code
- Customer's Wholesale Account Group Code (if item field is blank)
- Item's Drop Ship requirements
- Account's Drop Shipment capability

---

### 6. Sales Line Setup

When creating sales orders, relevant fields:

| Field | Populated By | Description |
|-------|--------------|-------------|
| **WholesaleOrderBMU** | User/Auto | Line should be wholesale sourced |
| **Wholesale Account Group Code** | Item/Auto | Overrides item's wholesale account group |
| **WS Account Priority Group Code** | Item/Auto | Overrides item's priority group |
| **Drop Shipment** | User | Line is drop ship |
| **Drop Ship Only** | Item | Cannot warehouse ship |
| **No Drop Ship Allowed** | Item | Cannot drop ship |

---

## Troubleshooting Guide

### Problem: Orders Not Transmitting Automatically

**Diagnostic Steps:**

1. **Verify Bumpdown Setup**
   - Navigate to Bumpdown Setup page
   - Confirm `Enabled = true`
   - Verify Transmission Mode is appropriate (Production/Test)
   - Check wholesaler credentials are populated

2. **Check Sales Order Eligibility**
   - Sales Order Status = **Released** (not Open)
   - At least one line has:
     - `WholesaleOrderBMU = true`
     - `Outstanding Quantity > 0`
     - `Purch. Order Line No. = 0` (not already linked)
     - `FailedWholesaleLineExistsBMU = false`

3. **Verify Job Queue**
   - Navigate to Job Queue Entries
   - Find entry with Object ID = 71552664, Parameter = 'PO'
   - Status = **Ready** (not "On Hold" or "Error")
   - Check "Last Ready State" timestamp
   - Review Job Queue Log Entries for errors

4. **Check for Stuck "In Progress" Flag**
   - If `SalesHeader.BumpdownInProgressBMU = true`, order is locked
   - This can happen if process was interrupted
   - Reset to `false` and retry

**Common Solutions:**
- Release the sales order
- Enable bumpdown setup
- Set job queue to Ready status
- Clear BumpdownInProgressBMU flag
- Verify item has vendor priorities configured

---

### Problem: Manual Transmission Not Available

**Symptom:** "Transmit Order" action button not visible or gives error

**Diagnostic Steps:**

1. **Check for Scheduled Tasks**
   - Error message: "A scheduled task is set up to transmit orders..."
   - Means automated job queue is active
   - Manual transmission is blocked to prevent conflicts

2. **Solutions:**
   - **Option A:** Wait for automated cycle to process
   - **Option B:** Disable/delete the job queue entry temporarily
   - **Option C:** Contact system administrator to run manual transmission

---

### Problem: Transmission Errors

**Symptom:** Sales Header has `BumpdownErrorBMU = true`

**Diagnostic Steps:**

1. **Check Sales Header Error Fields**
   - Open Sales Order
   - View `BumpdownErrorMsgBMU` field for error message
   - Common errors:
     - Authentication failure
     - Network timeout
     - Invalid credentials
     - Missing setup data

2. **Review Bumpdown Send-Receive Docs**
   - Navigate to: **Search → "Bumpdown Send-Receive Docs"**
   - Filter: `Document No. = [your order number]`
   - Check `Line Status` column:
     - "Transmission Error" = failed to send
     - Review `Transmission Error Message` field

3. **Check Error Message Register**
   - Navigate to: **Search → "Error Messages"**
   - Filter: `Additional Information` contains "Bumpdown PO Failed"
   - Review call stack and error details

4. **Common Issues and Solutions**

| Issue | Solution |
|-------|----------|
| "Authentication failed" | Verify Bumpdown Setup credentials |
| "Network error" | Check firewall, network connectivity |
| "Invalid transmission mode" | Confirm Production/Test mode matches wholesaler endpoint |
| "Missing Azure OAuth" | Verify Web App client ID and secret |
| "Expired credentials" | Update passwords in Bumpdown Setup |

**Resolution Steps:**
1. Fix the underlying issue (credentials, network, etc.)
2. Reset `BumpdownErrorBMU = false` on sales header
3. Clear `BumpdownErrorMsgBMU` field
4. Retry transmission (manual or wait for job queue)

---

### Problem: Lines Failing at Wholesaler

**Symptom:** Line transmitted but wholesaler rejected

**Diagnostic Steps:**

1. **Check Failed Wholesale Lines**
   - Navigate to: **Search → "Failed Wholesale Lines"**
   - Filter: `Source Document No. = [your order number]`
   - Review `Confirmation Code` and `Confirmation Description`

2. **From Sales Order Line**
   - Open sales order
   - Select the line
   - Click **Actions → Failed Wholesale Transmission Lines**
   - Review rejection reason

3. **Common Confirmation Codes**

| Code | Description | Meaning |
|------|-------------|---------|
| `INT-ERR-01` | No valid vendors found | Item has no configured vendor priorities |
| `OOS` | Out of stock | Wholesaler has no inventory |
| `DISC` | Discontinued | Item no longer available |
| `INACTIVE` | Inactive | Item not active in wholesaler system |
| `MIN-QTY` | Minimum quantity | Order doesn't meet minimum |
| (Various) | Wholesaler-specific | Consult wholesaler documentation |

4. **Resolution Based on Rejection**

**For "No valid vendors found":**
1. Check item's WS Account Priority Group Code
2. Verify Wholesale Account Group Code setup
3. Ensure item/customer have priority groups assigned
4. Confirm vendor accounts exist for priority group

**For wholesaler rejection (OOS, DISC, etc.):**
1. Decide if you want to:
   - **Try different vendor:** Clear Failed Wholesale Line, assign different priority group, retry
   - **Source elsewhere:** Clear failed line, manually create PO from different vendor
   - **Cancel line:** Delete or reduce quantity on sales line
2. Delete the FailedWholesaleLine record
3. Reset `FailedWholesaleLineExistsBMU = false` on sales line
4. Retry transmission

---

### Problem: Partial Acknowledgements Creating Split Lines

**Symptom:** One sales line becomes two after transmission

**This is expected behavior when:**
- You requested 10 units
- Wholesaler acknowledged 6 units
- System creates PO for 6 units
- New sales line created for remaining 4 units

**How It Works:**
1. **Original Line (Line 10000):**
   - Quantity reduced to 6 (acknowledged amount)
   - Linked to Purchase Order
   - Purchasing Code set to WSDROP or WSSPECIAL
   - Status shows "Acknowledged"

2. **New Split Line (Line 15000):**
   - Quantity = 4 (remaining)
   - `BumpdownCreatedEntryBMU = true`
   - `SplitFromLineNoBMU = 10000`
   - Will attempt bumpdown on next cycle
   - May try different vendor in priority sequence

**Viewing Split History:**
- Check `SplitFromLineNoBMU` field to trace lineage
- Review Bumpdown Send-Receive Docs for full history
- Filter by Transmission Group ID to see related attempts

**If You Don't Want the Split:**
- Delete the new split line
- Manually reduce quantity on original sales order
- Or manually create PO for remaining quantity from different vendor

---

### Problem: Vendor Priority Not Working as Expected

**Symptom:** Order going to wrong vendor or skipping vendors

**Diagnostic Steps:**

1. **View Transmission Preview**
   - Open Sales Order line
   - Click **Actions → View Wholesale Sequence**
   - Shows vendor priority order for that specific line
   - Displays wholesaler, account code, and priority number

2. **Verify Priority Group Setup**
   - Navigate to: **WS Account Priority Groups**
   - Find the priority group assigned to item
   - Verify sequence numbers and account assignments
   - Lower Priority Number = tried first

3. **Check Item vs Customer Priority**
   - Item card has Wholesale Account Group Code
   - Customer card may have different Wholesale Account Group Code
   - Item-level setting takes precedence if populated

4. **Drop Ship Compatibility**
   - If `DropShipOnlyBMU = true`, only drop ship accounts considered
   - If `NoDropShipAllowedBMU = true`, drop ship accounts excluded
   - Sales line `Drop Shipment = true` filters to drop ship accounts
   - Account must have matching `Drop Shipment` capability

---

## Monitoring and Reporting

### Key Pages for Monitoring

#### **1. Bumpdown Send-Receive Docs**
**Search:** "Bumpdown Send-Receive Docs"

Shows every transmission attempt:
- Filter by Document No. to see order history
- Group by Transmission Group ID to see related lines
- Line Status shows current state:
  - Not Processed → Ready to transmit
  - Pending → Awaiting response
  - Acknowledged → Accepted, PO created
  - Partially Acknowledged → Partial fill, PO created
  - Failed → Rejected by wholesaler
  - Transmission Error → Technical failure
  - PO Created → Processing complete

**Use for:**
- Troubleshooting transmission issues
- Auditing order history
- Tracking bumpdown attempts
- Retransmitting failed orders (via actions)

#### **2. Failed Wholesale Lines**
**Search:** "Failed Wholesale Lines"

Shows lines that exhausted all vendor priorities:
- Source Document information
- Confirmation Code and Description
- Handled in Req. Worksheet flag
- Req. Worksheet Action

**Use for:**
- Daily review of failed sourcing
- Identifying pattern issues (item discontinued, always OOS)
- Planning alternative sourcing strategies

#### **3. Wholesale Sourcing Cue**
**Search:** "Wholesale Sourcing Role Center"

Dashboard showing:
- Count of Failed Wholesale Lines
- Quick access to bumpdown pages
- Links to setup pages

#### **4. Job Queue Log Entries**
**Search:** "Job Queue Log Entries"

Filter by:
- Object ID = 71552664 (JobQueue-BumpdownBMU)
- Review Status: Success, Error, In Process
- Check Start Date/Time and End Date/Time
- View error messages for failed runs

---

## Best Practices

### Setup and Configuration

1. **Start with Test Mode**
   - Set Transmission Mode = "Internal Test" during initial setup
   - Verify line grouping and logic before live transmission
   - Switch to "Test" mode to test with wholesaler sandbox
   - Move to "Production" only after thorough testing

2. **Vendor Priority Strategy**
   - Rank vendors by: reliability, pricing, shipping speed
   - Consider regional distribution for faster shipping
   - Group items with similar vendor relationships
   - Review priorities quarterly based on performance

3. **Job Queue Scheduling**
   - Run every 15-30 minutes during business hours
   - Avoid running during known wholesaler maintenance windows
   - Disable overnight if not needed (save API calls)
   - Monitor job queue log for failures

4. **Credential Management**
   - Store credentials securely (BC handles masking)
   - Rotate passwords per security policy
   - Test credentials after changes
   - Document credential renewal schedule

### Operational Best Practices

1. **Daily Monitoring**
   - Review Failed Wholesale Lines first thing each morning
   - Check Job Queue Log for errors
   - Address transmission errors promptly
   - Clear resolved failed lines to keep data clean

2. **Order Release Timing**
   - Release orders in batches at optimal times
   - Consider wholesaler cut-off times for same-day processing
   - Don't release orders that can't be fulfilled (credit holds, etc.)

3. **Manual Transmission**
   - Use sparingly for urgent orders only
   - Disable job queue if doing extensive manual testing
   - Re-enable job queue when done
   - Document why manual transmission was needed

4. **Error Resolution**
   - Address authentication errors immediately (blocks all transmission)
   - For network errors, check with IT before retrying
   - For wholesaler rejections, research item availability
   - Keep notes on recurring issues for pattern analysis

### Maintenance

1. **Regular Reviews**
   - Monthly: Review Failed Wholesale Lines trends
   - Quarterly: Audit vendor priority effectiveness
   - Annually: Review wholesaler credentials and contracts
   - As needed: Optimize job queue schedule

2. **Data Cleanup**
   - Archive old Bumpdown Send-Receive Docs (per retention policy)
   - Resolve or delete old Failed Wholesale Lines
   - Clean up Error Message records

3. **Performance Optimization**
   - Monitor job queue execution time
   - If transmission taking too long, consider:
     - More frequent smaller batches
     - Splitting by location or customer
     - Staggered transmission windows

---

## Advanced Topics

### Understanding Bumpdown Groups

Lines are grouped to minimize API calls and optimize transmission:

**Lines grouped together when they have:**
- Identical vendor priority sequence
- Same priority account codes
- Same priority order

**Example Scenario:**
```
Item A: Priority = [Essendant-East, SPR-West, Essendant-West]
Item B: Priority = [Essendant-East, SPR-West, Essendant-West]
Item C: Priority = [SPR-Central, Essendant-Central]

Result: Items A & B share Bumpdown Group 1
        Item C gets Bumpdown Group 2
```

**Benefits:**
- Single PO created for all lines in group (when acknowledged by same vendor)
- Reduced API calls
- Faster processing

### Transmission Group ID vs Bumpdown Group ID

**Bumpdown Group ID:**
- Assigned during initial line grouping
- Represents vendor priority sequence
- Persists across retry attempts
- All lines with same priorities share ID

**Transmission Group ID:**
- Assigned when creating Send-Receive Docs
- Unique per transmission attempt
- Different for each bumpdown retry
- Used to track related Send-Receive Docs

### Load Factor and Unit Cost Updates

After successful transmission:
1. Purchase Line created with wholesaler's acknowledged unit cost
2. `LoadFactorMgtBMU.UpdateSalesLineUnitCostFromPO()` called
3. Calculates loaded cost = (PO unit cost × load factor) + freight allocation
4. Updates Sales Line unit cost
5. May update sales line profit margin

**Load Factor** accounts for:
- Freight/shipping costs
- Handling fees
- Location-specific overhead

### Handling In-Progress Lines

When transmission is interrupted (system crash, timeout):
- Lines may have Status = "Acknowledged" but no PO created
- Next transmission attempt detects this via `HandleInProgressLines()`
- System creates PO for acknowledged quantity
- No duplicate transmission occurs

**This handles edge cases like:**
- Response received but PO creation failed
- User closes session during processing
- Task scheduler interruption

---

## Field Reference

### Sales Header Fields

| Field | Type | Description |
|-------|------|-------------|
| `BumpdownInProgressBMU` | Boolean | Currently being processed (prevents concurrent runs) |
| `BumpdownErrorBMU` | Boolean | Error occurred during last transmission |
| `BumpdownErrorMsgBMU` | Text[250] | Error message from last failed transmission |

### Sales Line Fields

| Field | Type | Description |
|-------|------|-------------|
| `WholesaleOrderBMU` | Boolean | Line should be wholesale sourced |
| `WholesaleAccountGroupCodeBMU` | Code[20] | Overrides customer's wholesale account group |
| `WSAccountPriorityGroupCodeBMU` | Code[20] | Defines vendor attempt sequence |
| `DropShipOnlyBMU` | Boolean | Can only be drop shipped |
| `NoDropShipAllowedBMU` | Boolean | Cannot be drop shipped |
| `LastBumpdownStatusBMU` | Enum | Last transmission status (from Send-Receive Doc) |
| `LastBumpdownEntryNoBMU` | Integer | Links to most recent BumpdownSendReceiveDoc |
| `FailedWholesaleLineExistsBMU` | Boolean | Has unresolved failed wholesale line |
| `BumpdownCreatedEntryBMU` | Boolean | Created by split line logic |
| `SplitFromLineNoBMU` | Integer | Original line number if split occurred |

### BumpdownSendReceiveDoc Fields

| Field | Type | Description |
|-------|------|-------------|
| `Entry No.` | Integer | Primary key, auto-increment |
| `Document Type` | Enum | Sales document type |
| `Document No.` | Code[20] | Sales order number |
| `Document Line No.` | Integer | Sales line number |
| `Transmission Group ID` | Integer | Groups related transmission attempts |
| `Wholesaler` | Enum | Essendant or SPRichards |
| `Account Code` | Code[20] | Wholesaler account code |
| `Vendor No.` | Code[20] | BC vendor record |
| `Purchase Order No.` | Code[20] | Generated PO number |
| `Line Status` | Enum | Current processing status |
| `Item No.` | Code[20] | Item being sourced |
| `Quantity` | Decimal | Requested quantity |
| `Quantity Acknowledged` | Decimal | Wholesaler's accepted quantity |
| `Unit Cost` | Decimal | Requested unit cost |
| `Unit Cost Acknowledged` | Decimal | Wholesaler's unit cost |
| `Confirmation Code` | Code[10] | Wholesaler's response code |
| `Confirmation Description` | Text[50] | Reason for failure/acknowledgement |
| `Drop Shipment` | Boolean | Is this a drop ship order |
| `Transmission Error Message` | Text[250] | Technical error details |

### FailedWholesaleLine Fields

| Field | Type | Description |
|-------|------|-------------|
| `Source Document Type` | Enum | Sales document type |
| `Source Document No.` | Code[20] | Sales order number |
| `Source Document Line No.` | Integer | Sales line number |
| `Confirmation Code` | Code[30] | Final failure code |
| `Confirmation Description` | Text[50] | Reason all vendors failed |
| `Handled in Req. Worksheet` | Boolean | User has addressed this failure |
| `Req. Worksheet Action` | Enum | Action taken in requisition worksheet |

---

## Support and Additional Resources

### Getting Help

1. **Check This Documentation First**
   - Review relevant troubleshooting section
   - Verify setup requirements met
   - Follow diagnostic steps

2. **Review System Logs**
   - Bumpdown Send-Receive Docs
   - Failed Wholesale Lines
   - Job Queue Log Entries
   - Error Messages

3. **Contact BMI Support**
   - Website: https://bmiusa.freshdesk.com/support/home
   - Include: Sales Order number, error messages, screenshots
   - Specify: Wholesaler involved, transmission timestamp

### Related Documentation

- **Vendor Priority Setup** - Configuring WS Account Priority Groups
- **Item Setup Guide** - Wholesale sourcing fields
- **Azure OAuth Setup** - BC-Azure OAUTH.docx (BMI provided)
- **Wholesaler Integration Guides** - Essendant/SPR specific procedures

---

## Appendix: Common Scenarios

### Scenario 1: Rush Order Needs Immediate Transmission

**Situation:** Customer calls with urgent order, can't wait for job queue

**Steps:**
1. Create and release sales order as normal
2. Verify order meets bumpdown eligibility
3. Check if job queue is active (may need admin help)
4. Use **Actions → Transmit Order**
5. Monitor Bumpdown Send-Receive Docs for status
6. If acknowledged, purchase order created immediately

---

### Scenario 2: Vendor Out of Stock, Need to Try Next Vendor

**Situation:** First vendor rejected with "OOS", want to try second vendor

**Automatic Handling:**
- System automatically bumps to next vendor in priority list
- No user action needed
- Process repeats until vendor accepts or list exhausted

**If all vendors fail:**
1. Failed Wholesale Line created
2. Review alternatives in Requisition Worksheet
3. Or manually create PO from alternate vendor
4. Delete Failed Wholesale Line record when resolved

---

### Scenario 3: Wrong Vendor Priority, Need to Change

**Situation:** Item going to wrong vendor, need to adjust priorities

**Steps:**
1. Go to **WS Account Priority Groups**
2. Find priority group assigned to item
3. Adjust priority numbers or account assignments
4. If line already transmitted and failed:
   - Delete FailedWholesaleLine record
   - Reset `FailedWholesaleLineExistsBMU = false`
5. Retry transmission (manual or wait for job queue)

---

### Scenario 4: Testing New Item Setup Before Live Orders

**Situation:** New item, want to verify bumpdown works correctly

**Steps:**
1. Ensure Bumpdown Setup: `Transmission Mode = Internal Test`
2. Create test sales order with new item
3. Release order
4. Use **Actions → Transmit Order**
5. Review Bumpdown Send-Receive Docs
6. Verify vendor sequence correct
7. Check confirmation codes (Internal Test simulates responses)
8. Delete test order when satisfied

---

### Scenario 5: Transmission Failed Due to Expired Credentials

**Situation:** All transmissions failing with authentication error

**Steps:**
1. Open **Bumpdown Setup** page
2. Update expired credentials:
   - For Essendant: Unilink Password field
   - For SP Richards: FTP Password field
3. Save changes
4. Find all orders with `BumpdownErrorBMU = true`
5. Reset error flags:
   - Set `BumpdownErrorBMU = false`
   - Clear `BumpdownErrorMsgBMU`
6. Orders will retry on next job queue cycle
7. Or use manual transmission for immediate retry

---

## Document Revision History

| Date | Version | Changes |
|------|---------|---------|
| 2025-12-23 | 1.0 | Initial documentation - Purchase Order transmission and bumpdown logic |

---

*For questions or issues not covered in this documentation, contact BMI Support at https://bmiusa.freshdesk.com/support/home*
