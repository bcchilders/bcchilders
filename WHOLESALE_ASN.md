# BMI SupplyAutomate - Wholesale ASN (Advance Ship Notice)

## Overview

The **Wholesale ASN** (Advance Ship Notice) system automates the receipt and processing of shipping notifications from wholesalers (Essendant and SP Richards). When a wholesaler ships items to fulfill a purchase order, they transmit an ASN file containing shipment details. The system imports these files, processes the shipment data, automatically receives purchase orders, creates warehouse documents, and generates tracking information.

### Business Problem Solved

**Manual processing challenges:**

- Manually receiving wholesale purchase orders is time-consuming
- Tracking numbers must be manually entered for each shipment
- Package counts need manual counting and entry
- Risk of receiving incorrect quantities
- Delayed visibility into incoming shipments

**ASN automation provides:**

- **Automatic PO receipts** - Purchase orders received automatically based on ASN data
- **Warehouse integration** - Creates warehouse receipts, put-aways, picks, and shipments
- **Tracking numbers** - Automatically populates tracking information
- **Package counts** - Creates accurate carton/package records
- **Real-time visibility** - Immediate notification when shipments are sent

---

## Key Concepts

### What is an ASN?

An **Advance Ship Notice (ASN)** is an electronic document sent by the wholesaler notifying the dealer that:

- A shipment has been dispatched
- What items and quantities are in the shipment
- Which purchase order(s) the shipment fulfills
- Tracking numbers for the packages
- Number of cartons/packages

### ASN File Types

#### **Essendant ASN Files**

- **ASN Detail Files** - Line-level shipment details (items, quantities, tracking)
- **Manifest Files** - Package/carton count information
- **Format**: XML files via ELink API
- **Transmission**: Retrieved via web service

#### **SP Richards ASN Files**

- **Single ASN Files** - Special order shipments (warehouse fulfillment)
- **Multi ASN Files** - Drop ship shipments (direct to customer)
- **Format**: Fixed-width text files
- **Transmission**: Retrieved via FTP server

### Processing Stages

The ASN process has three distinct stages:

1. **Import & Translate**: Download ASN files and convert to BC format
2. **Receipt Processing**: Receive purchase orders and create warehouse documents
3. **Ship Processing**: Create picks, shipments, and package records (special orders only)
4. **Tracking**: Populate tracking numbers on sales/purchase documents

### Order Type Handling

#### **Drop Shipment Orders**

- Wholesaler ships directly to customer
- System receives the purchase order
- Does NOT create warehouse pick/shipment (already shipped to customer)
- Tracking number recorded on purchase/sales order

#### **Special Order (Warehouse) Orders**

- Wholesaler ships to dealer warehouse
- System receives the purchase order
- Creates warehouse receipt and put-away
- Creates warehouse pick and shipment for outbound to customer
- Creates package records
- Records tracking numbers

---

## How the Wholesale ASN Process Works

### Automated Processing (Job Queue)

The system processes ASN data through scheduled job queue entries that handle different stages.

#### **Stage 1: Import & Translate ASN Files**

##### **Job Queue Configuration**

- **Parameter**: `'ASN-MANIFEST-IMPORT-TRANSLATE'` or `'SPR-ASN-IMPORT-TRANSLATE'`
- **Frequency**: Every 15-30 minutes
- **Purpose**: Downloads and translates new ASN files

##### **Process Flow**

**For Essendant:**

1. **Download ASN Files**
   - `EssendantASNDataMgt.DownloadNewEssendantASNFiles()` called
   - Connects to ELink web service
   - Downloads new ASN detail files
   - Files stored in `Essendant ASN Header` and `Essendant ASN Detail` tables

2. **Download Manifest Files**
   - `EssendantASNDataMgt.DownloadNewEssendantManifestFiles()` called
   - Downloads package/carton count files
   - Files stored in `Essendant Manifest v2` table

3. **Translate ASN Details**
   - `TranslateEssendantASNFiles()` processes ASN headers
   - For each ASN file:
     - Parses item numbers, quantities, tracking numbers
     - Matches items to purchase order lines
     - Creates `EssendantASNData` records
     - Consolidates quantities if multiple ASN lines for same PO line

4. **Translate Manifest Files**
   - `TranslateEssendantManifestFiles()` processes manifest data
   - Matches Essendant Order Number to ASN Data
   - Increments `Package Quantity` field

5. **Marks as Translated**
   - Sets `Translated in BC = true` on source files
   - Prevents reprocessing same file

**For SP Richards:**

1. **Download ASN Files**
   - Connects to SP Richards FTP server
   - Downloads new ASN files (SINGLE or MULTI format)
   - Files stored in SPR ASN tables

2. **Translate Single ASN Files** (Special Orders)
   - Processes `SPR ASN Manifest Hdr` with type = 'SINGLE'
   - Reads `SPR ASN Order Summary` and `SPR ASN Order Line`
   - Matches to purchase orders using:
     - PO Number (CPONO)
     - Reference Code (REFCD1) = BDSendReceiveDocEntryNo
     - Item Number (MFRID + STCKNO)
   - Creates `EssendantASNData` records (shared table for both wholesalers)
   - Counts cartons from `SPR ASN Carton V2` table

3. **Translate Multi ASN Files** (Drop Ships)
   - Processes `SPR ASN Manifest Hdr` with type = 'MULTI'
   - Only processes lines where `IsDropShipOrder = true`
   - Extracts tracking numbers from carton data
   - Creates ASN Data records

4. **Marks as Translated**
   - Sets `Translated in BC = true`

##### **Key Table: EssendantASNData**

This is the central staging table used by both wholesalers:

| Field | Description |
| ----- | ----------- |
| Purchase Order No. | BC purchase order number |
| Purchase Order Line No. | BC purchase line number |
| Sales Order No. | Linked sales order |
| Sales Order Line No. | Linked sales line |
| Qty. to Receive | Quantity being shipped |
| Package Quantity | Number of cartons/packages |
| Wholesaler | Essendant or SPRichards |
| Tracking Number | Shipment tracking number |
| Receipt Processed | Purchase receipt completed |
| Ship Processed | Warehouse pick/ship completed |
| Tracking Numbers Processed | Tracking populated |
| Wholesale Order | True for bumpdown orders |

---

#### **Stage 2: Receipt Processing**

##### **Job Queue Configuration (Receipt)**

- **Parameter**: `'ASN-PROCESS'`
- **Frequency**: Every 15-30 minutes (after import/translate)
- **Purpose**: Process ASN data into BC documents

##### **Process Flow**

###### **Step 1: Consolidate Quantities**

- Groups ASN Data by Purchase Order and Line
- Sums `Qty. to Receive` for each PO line
- Creates temporary buffers for processing

###### **Step 2: Process Purchase Receipts** (`ProcessASNReceiptBMU`)

For each Purchase Order:

1. **Validate and Prepare**
   - Verifies purchase order and lines exist
   - Updates posting date to today
   - Releases purchase order if not already released
   - Identifies order type (Drop Ship vs Special Order)

2. **Drop Shipment Orders:**
   - Sets `Qty. to Receive` on purchase lines from ASN data
   - Posts purchase receipt directly
   - Does NOT create warehouse documents
   - Shipment already went to customer

3. **Special Order (Warehouse) Orders:**
   - Creates Warehouse Receipt from purchase order
   - Sets `Qty. to Receive` on warehouse receipt lines from ASN data
   - Posts warehouse receipt
   - Creates Put-Away documents
   - Updates put-away lines with wholesaler bin codes
   - Registers put-away automatically

4. **Update ASN Data Status**
   - Sets `Receipt Processed = true` on successful processing
   - Records error message if processing fails
   - Commits transaction

###### **Step 3: Process Warehouse Shipments** (`ProcessASNShipBMU`)

*Note: Only for Special Order lines that went to warehouse*

For each Sales Order:

1. **Create Warehouse Shipment**
   - Checks if warehouse shipment already exists
   - Creates shipment from sales order if needed
   - Sets `Wholesale Bin Code` on shipment lines

2. **Create and Register Pick**
   - Generates warehouse pick from shipment
   - Updates bin codes on pick lines
   - Automatically registers the pick

3. **Create Package Records**
   - Creates `LAX Package` records
   - Quantity = `Package Quantity` from ASN Data
   - Package Type = Essendant or SP Richards
   - Links packages to sales order

4. **Update ASN Data Status**
   - Sets `Ship Processed = true`
   - Records errors if any
   - Commits transaction

###### **Step 4: Process Tracking Numbers** (`ProcessTrackingNumbersBMU`)

For each ASN Data record where Receipt Processed = true and Ship Processed = true:

1. **Create Tracking Number Record**
   - Source: Sales Order and Line
   - Purchase Order and Line (optional)
   - Tracking Number Source = Wholesaler enum
   - Shipment Date = ASN SystemCreatedAt date
   - Tracking Number = from ASN data

2. **Update ASN Data Status**
   - Sets `Tracking Numbers Processed = true`
   - Marks complete

---

### Manual Operations

#### **View ASN Data**

Navigate to: **Search → "ASN Data"** or **"Essendant ASN Data"**

This page shows all ASN records:

- Filter by Purchase Order or Sales Order
- Check processing status flags
- View error messages
- Manually trigger reprocessing if needed

#### **Troubleshoot Failed Processing**

1. Open ASN Data page
2. Filter: `Receipt Processed = false` or `Ship Processed = false`
3. Review `Last Error Message` field
4. Fix underlying issue (PO exists, quantities valid, etc.)
5. Clear error message
6. Wait for next job queue cycle or manually process

#### **Manual Receipt** (Not Recommended)

If ASN processing fails and cannot be resolved:

1. Manually receive purchase order using standard BC process
2. Mark ASN Data record as `Receipt Processed = true`
3. Prevents automatic processing attempts

---

### Data Flow Summary

```text
┌─────────────────────────────────────────────────────────┐
│  Stage 1: Import & Translate (Every 15-30 min)         │
└─────────────────────────────────────────────────────────┘
    │
    │ Downloads files from wholesaler
    ▼
┌─────────────────────┐       ┌──────────────────────┐
│ Essendant ASN Files │       │ SP Richards ASN Files│
│ (ELink API - XML)   │       │ (FTP - Text Files)   │
└─────────────────────┘       └──────────────────────┘
    │                               │
    │ Parse & Match to PO           │
    ▼                               ▼
┌──────────────────────────────────────────────────────────┐
│           EssendantASNData (Central Staging Table)       │
│  - Purchase Order No. & Line No.                         │
│  - Sales Order No. & Line No.                            │
│  - Qty to Receive, Package Qty, Tracking Number          │
└──────────────────────────────────────────────────────────┘
    │
    │
┌─────────────────────────────────────────────────────────┐
│  Stage 2: Receipt Processing (Every 15-30 min)         │
└─────────────────────────────────────────────────────────┘
    │
    ├─────────────────────────────┬─────────────────────────┐
    ▼                             ▼                         ▼
┌──────────────┐      ┌─────────────────────┐   ┌──────────────────┐
│ Drop Ship    │      │ Special Order       │   │ Tracking Numbers │
│ - Post PO    │      │ - Whse Receipt      │   │ - Create records │
│   Receipt    │      │ - Put-Away          │   │ - Link to SO/PO  │
└──────────────┘      │ - Whse Pick         │   └──────────────────┘
                      │ - Whse Shipment     │
                      │ - Package Records   │
                      └─────────────────────┘
```

---

## Setup and Configuration

### 1. Bumpdown Setup

ASN credentials are configured in **Bumpdown Setup** page:

Navigate to: **Search → "Bumpdown Setup"**

#### **Essendant ELink Settings** (for ASN)

| Field | Description | Required |
|-------|-------------|----------|
| **ELink User ID** | Essendant ELink API username | Yes |
| **ELink Password** | Essendant ELink API password (masked) | Yes |

*Note: ELink is different from UNILINK. ELink handles ASN, Invoice, and iCAPS data.*

#### **SP Richards Settings** (for ASN)

| Field | Description | Required |
|-------|-------------|----------|
| **FTP Username** | FTP server username for file downloads | Yes |
| **FTP Password** | FTP server password (masked) | Yes |

### 2. Job Queue Setup

Create job queue entries for ASN processing:

Navigate to: **Search → "Job Queue Entries"**

#### **Entry 1: Essendant ASN Import & Translate**

| Field | Value |
|-------|-------|
| **Object Type to Run** | Codeunit |
| **Object ID to Run** | 71552664 (JobQueue-BumpdownBMU) |
| **Parameter String** | `ASN-MANIFEST-IMPORT-TRANSLATE` |
| **Status** | Ready |
| **Recurring Job** | Yes |
| **No. of Minutes between Runs** | 15-30 |

#### **Entry 2: SP Richards ASN Import & Translate**

| Field | Value |
|-------|-------|
| **Object Type to Run** | Codeunit |
| **Object ID to Run** | 71552664 (JobQueue-BumpdownBMU) |
| **Parameter String** | `SPR-ASN-IMPORT-TRANSLATE` |
| **Status** | Ready |
| **Recurring Job** | Yes |
| **No. of Minutes between Runs** | 15-30 |

#### **Entry 3: ASN Processing**

| Field | Value |
|-------|-------|
| **Object Type to Run** | Codeunit |
| **Object ID to Run** | 71552664 (JobQueue-BumpdownBMU) |
| **Parameter String** | `ASN-PROCESS` |
| **Status** | Ready |
| **Recurring Job** | Yes |
| **No. of Minutes between Runs** | 15-30 |

**Recommended Schedule:**
- Run import/translate at :00 and :30 of each hour
- Run ASN process at :15 and :45 of each hour
- Ensures translate completes before processing begins

---

### 3. Location Setup

For warehouse locations processing special orders:

Navigate to: **Search → "Locations"**

#### **Wholesaler Bin Codes**

Each location must have wholesaler-specific bins configured:

| Setup Field | Purpose |
|-------------|---------|
| **Essendant Bin Code** | Default bin for Essendant items |
| **SP Richards Bin Code** | Default bin for SP Richards items |

**Why needed:**
- ASN put-away needs to know where to stage wholesale items
- Bins are typically near packing/shipping area
- Keeps wholesale items separate for easy picking

---

### 4. Warehouse Setup

For locations using ASN automation:

Navigate to: **Location Card → Warehouse FastTab**

| Setting | Requirement |
|---------|-------------|
| **Require Receive** | Must be enabled for special orders |
| **Require Put-away** | Must be enabled for special orders |
| **Require Shipment** | Must be enabled for special orders |
| **Require Pick** | Must be enabled for special orders |
| **Bin Mandatory** | Must be enabled |

**Bin Setup:**
- Create receiving bins
- Create wholesaler staging bins (Essendant Bin, SPR Bin)
- Create shipping bins

---

## User Guide

### Daily Operations

#### **Monitor Incoming ASN Files**

**Morning Routine:**
1. Navigate to **ASN Data** page
2. Filter: `SystemCreatedAt = TODAY`
3. Review new ASN records received
4. Check for any with error messages

#### **Verify Receipt Processing**

1. Open **ASN Data** page
2. Filter: `Receipt Processed = false`
3. Should be empty if all processing successful
4. If records exist:
   - Review `Last Error Message`
   - Check if purchase orders exist and are valid
   - Verify quantities are reasonable

#### **Verify Shipment Processing**

1. Open **ASN Data** page
2. Filter: `Ship Processed = false` AND `Receipt Processed = true`
3. Should be empty unless very recent receipts
4. Review errors if any exist

#### **Check Tracking Numbers**

1. Open **Sales Order**
2. Navigate to **Lines → Tracking Numbers** action
3. Verify tracking numbers populated from ASN
4. Or open **Tracking Numbers** page and filter by order

---

### Common Tasks

#### **View ASN History for Purchase Order**

1. Open **Purchase Order**
2. Navigate to **ASN Data** (custom action/field)
3. Or manually: Open **ASN Data** page, filter by PO number
4. Review all ASN records for this order

#### **View ASN History for Sales Order**

1. Open **Sales Order**
2. View tracking numbers populated by ASN
3. Check package records created
4. Review ASN Data filtered by SO number

#### **Reprocess Failed ASN**

If an ASN record failed processing:

1. Open **ASN Data** page
2. Find the failed record (error message populated)
3. **Fix the underlying issue:**
   - Verify purchase order exists
   - Check quantities are valid
   - Ensure warehouse location setup correct
   - Verify sales order exists (for special orders)
4. **Clear the error:**
   - Clear `Last Error Message` field
   - Set processing flags to `false`:
     - `Receipt Processed = false` (to reprocess receipt)
     - `Ship Processed = false` (to reprocess shipment)
5. **Wait for job queue** to retry processing
6. Or manually trigger by running codeunit

---

## Troubleshooting Guide

### Problem: ASN Files Not Importing

**Symptom:** No new ASN Data records being created

**Diagnostic Steps:**

1. **Check Job Queue Status**
   - Navigate to: **Job Queue Entries**
   - Find entries with Parameter = `ASN-MANIFEST-IMPORT-TRANSLATE` or `SPR-ASN-IMPORT-TRANSLATE`
   - Verify Status = **Ready** (not "On Hold" or "Error")
   - Check "Last Ready State" timestamp (should be recent)

2. **Check Job Queue Log**
   - Navigate to: **Job Queue Log Entries**
   - Filter by Object ID = 71552664
   - Look for errors in recent runs
   - Review error messages

3. **Verify Credentials**
   - Open **Bumpdown Setup**
   - **For Essendant:** Check ELink User ID and Password
   - **For SP Richards:** Check FTP Username and Password
   - Test connection if possible

4. **Check Wholesaler Files**
   - **Essendant:** Verify ELink API is accessible
   - **SP Richards:** Verify FTP server is accessible
   - Confirm files are being generated by wholesaler

**Common Causes:**

| Issue | Solution |
|-------|----------|
| Expired credentials | Update passwords in Bumpdown Setup |
| Job queue on hold | Set Status = Ready |
| Network/firewall issue | Contact IT to verify connectivity |
| Wholesaler system down | Contact wholesaler support |
| No shipments to process | Normal - wait for shipments |

---

### Problem: ASN Data Created But Receipt Not Processing

**Symptom:** ASN Data records exist with `Receipt Processed = false`

**Diagnostic Steps:**

1. **Check ASN Data Error Message**
   - Open **ASN Data** page
   - Filter: `Receipt Processed = false`
   - Review `Last Error Message` field
   - Common errors:
     - "Purchase order does not exist"
     - "Purchase line not found"
     - "Quantity exceeds outstanding"

2. **Verify Purchase Order Exists**
   - Note the `Purchase Order No.` from ASN Data
   - Search for the purchase order
   - Verify it hasn't been deleted or fully received

3. **Check Purchase Line**
   - Open the purchase order
   - Find the line matching `Purchase Order Line No.` from ASN
   - Verify `Outstanding Quantity > 0`
   - Check line hasn't been manually received

4. **Validate Quantities**
   - ASN `Qty. to Receive` must be ≤ `Outstanding Quantity`
   - If greater, wholesaler sent more than ordered
   - May need manual intervention

5. **Check Order Status**
   - Purchase Order Status must allow receiving
   - Should be Released or Open status
   - Not already fully received

**Solutions:**

| Problem | Resolution |
|---------|------------|
| PO deleted/archived | Cannot process - mark ASN as processed manually |
| PO already received | Mark ASN `Receipt Processed = true` |
| Line already received | Mark ASN `Receipt Processed = true` |
| Quantity mismatch | Adjust ASN qty or receive difference manually |
| PO not released | Release the purchase order |

**Manual Workaround:**
1. Receive purchase order manually
2. Open ASN Data record
3. Set `Receipt Processed = true`
4. Clear error message
5. Allow ship processing to continue

---

### Problem: Receipt Processed But Ship Not Processing

**Symptom:** Special order ASN with `Receipt Processed = true` but `Ship Processed = false`

**Diagnostic Steps:**

1. **Verify It's a Special Order**
   - Drop ship orders don't need ship processing
   - Check purchase line: `Special Order = true` vs `Drop Shipment = true`
   - If drop shipment, this is normal - ship processing skipped

2. **Check Sales Order Exists**
   - ASN Data should have `Sales Order No.`
   - Verify sales order still exists
   - Check not already shipped

3. **Check Error Message**
   - Review `Last Error Message` in ASN Data
   - Common issues:
     - "Sales order not found"
     - "Warehouse shipment error"
     - "Cannot create pick"

4. **Verify Warehouse Setup**
   - Location must have:
     - Require Receive = true
     - Require Shipment = true
     - Require Pick = true
   - Wholesaler bin codes configured

5. **Check Bin Availability**
   - Items must be in correct bins after put-away
   - Run **Adjust Bin Content** if needed
   - Verify bin has sufficient quantity

**Solutions:**

| Problem | Resolution |
|---------|------------|
| Drop shipment | Normal - no ship processing needed |
| SO deleted | Cannot process - mark Ship Processed = true |
| SO already shipped | Mark Ship Processed = true |
| Warehouse setup incomplete | Configure location warehouse settings |
| Bin content issues | Adjust bin content, rerun processing |
| Quantity not in bins | Register put-away first |

---

### Problem: Tracking Numbers Not Populating

**Symptom:** ASN processed but tracking numbers not visible

**Diagnostic Steps:**

1. **Check ASN Data Status**
   - Open **ASN Data** page
   - Verify `Tracking Numbers Processed = true`
   - If false, check error message

2. **Verify Tracking Number Exists**
   - Check `Tracking Number` field in ASN Data
   - May be blank if wholesaler didn't provide
   - Essendant AACT (W&L) shipments won't have tracking

3. **Check Tracking Numbers Table**
   - Navigate to: **Search → "Tracking Numbers"**
   - Filter by Sales Order or Purchase Order
   - Verify records were created

4. **Sales Order Visibility**
   - Open **Sales Order**
   - Navigate to **Lines → Tracking Numbers** action
   - Should display tracking number list

**Common Causes:**

| Issue | Solution |
|-------|----------|
| Wholesaler didn't send tracking | Normal for some carriers (W&L) |
| Processing failed | Review error message, reprocess |
| Carrier Code = AACT | Expected - AACT doesn't have tracking |
| Wrong source type | Verify Tracking Number Source = correct wholesaler |

---

### Problem: Duplicate ASN Processing

**Symptom:** Same shipment processed multiple times

**Diagnostic Steps:**

1. **Check "Translated in BC" Flag**
   - Navigate to **Essendant ASN Header** or **SPR ASN Manifest Hdr**
   - Filter by recent dates
   - Verify `Translated in BC = true` after processing

2. **Check for Duplicate Files**
   - Wholesaler may have sent same file multiple times
   - Check SA Transmission ID for duplicates

3. **Review Job Queue Timing**
   - Import and process jobs running too close together
   - May process before translation complete

**Solutions:**
- Verify translation sets `Translated in BC = true`
- Check job queue scheduling
- Review wholesaler file transmission logs
- Delete duplicate ASN Data records if created

---

### Problem: Package Counts Incorrect

**Symptom:** Wrong number of packages created for sales order

**Diagnostic Steps:**

1. **Check ASN Data Package Quantity**
   - Open **ASN Data** page
   - Filter by Sales Order
   - Sum `Package Quantity` field
   - Should match physical cartons

2. **Verify Manifest File Processed**
   - **Essendant:** Check `Essendant Manifest v2` table
   - **SP Richards:** Check `SPR ASN Carton v2` table
   - Count records for order number
   - Verify `Translated in BC = true`

3. **Check LAX Package Records**
   - Navigate to **Search → "Packages"**
   - Filter by Source ID = Sales Order No.
   - Count records
   - Verify Package Type set correctly

4. **Review Manifest Timing**
   - Manifest files may arrive after ASN files
   - System increments package count when manifest processed
   - May need to wait for manifest arrival

**Solutions:**

| Problem | Resolution |
|---------|------------|
| Manifest not received yet | Wait for manifest file, will update |
| Manifest file import failed | Check job queue, credentials |
| Package Quantity not summing | Review multiple ASN lines for order |
| LAX Packages not created | Review Ship Processing errors |

---

## Monitoring and Reporting

### Key Pages

#### **1. ASN Data Page**

**Search:** "ASN Data" or "Essendant ASN Data"

**Primary monitoring page showing:**
- All ASN records (both wholesalers)
- Processing status flags
- Error messages
- Purchase/Sales order links
- Quantities and package counts

**Key Filters:**
- `SystemCreatedAt = TODAY` - Today's ASN receipts
- `Receipt Processed = false` - Pending receipt
- `Ship Processed = false` - Pending shipment
- `Last Error Message <> ''` - Failed records
- `Wholesaler = Essendant/SPRichards` - By wholesaler
- `Wholesale Order = true` - Bumpdown orders only

**Use for:**
- Daily monitoring
- Troubleshooting failures
- Auditing processing
- Verifying quantities

---

#### **2. Essendant ASN Header**

**Search:** "Essendant ASN Header"

**Shows:**
- Raw ASN files received from Essendant
- SA Transmission ID (unique identifier)
- PO Number, Essendant Order Number
- Carrier Code
- Translated in BC flag

**Use for:**
- Verifying file receipt from Essendant
- Troubleshooting import issues
- Reviewing translation status
- Checking carrier information

---

#### **3. SPR ASN Manifest Hdr**

**Search:** "SPR ASN Manifest" or similar

**Shows:**
- Raw ASN files received from SP Richards
- ASN File Type (SINGLE vs MULTI)
- SA Transmission ID
- DCS Order Number
- Translated in BC flag

**Use for:**
- Verifying file receipt from SPR
- Distinguishing drop ship vs special order
- Troubleshooting import issues
- Checking translation status

---

#### **4. Job Queue Log Entries**

**Search:** "Job Queue Log Entries"

**Filter by:**
- Object ID = 71552664 (JobQueue-BumpdownBMU)
- Status = Error (to find failures)

**Shows:**
- Execution history
- Success/failure status
- Error messages
- Start/end times

**Use for:**
- Diagnosing job queue failures
- Verifying scheduled execution
- Performance monitoring
- Error pattern analysis

---

#### **5. Tracking Numbers Page**

**Search:** "Tracking Numbers"

**Shows:**
- All tracking numbers in system
- Source documents (SO/PO)
- Tracking Number Source (Essendant, SPRichards, Manual, etc.)
- Shipment date

**Use for:**
- Verifying tracking populated from ASN
- Customer service inquiries
- Shipment research
- Carrier tracking

---

### Health Check Procedures

#### **Daily Health Check**

Run each morning:

1. **Check for Failed ASN Processing**
   ```
   ASN Data page
   Filter: Last Error Message <> ''
   Result: Should be empty or minimal
   ```

2. **Verify Files Being Imported**
   ```
   ASN Data page
   Filter: SystemCreatedAt = TODAY
   Result: Should have records if shipments occurred
   ```

3. **Check Job Queue Execution**
   ```
   Job Queue Log Entries
   Filter: Object ID = 71552664, Start Date/Time >= YESTERDAY
   Result: Multiple successful executions
   ```

4. **Review Unprocessed Records**
   ```
   ASN Data page
   Filter: Receipt Processed = false OR Ship Processed = false
   Result: Only very recent records (< 1 hour old)
   ```

---

#### **Weekly Health Check**

Run weekly for trends:

1. **ASN Volume Analysis**
   - Count ASN Data records created this week
   - Compare to previous weeks
   - Identify unusual spikes or drops

2. **Error Rate Analysis**
   - Count records with error messages
   - Calculate error rate percentage
   - Investigate if > 5%

3. **Processing Time Analysis**
   - Review job queue execution times
   - Check if increasing (performance issue)
   - Optimize if needed

4. **Tracking Number Coverage**
   - Count ASN records with tracking numbers
   - Compare to total ASN records
   - Verify expected percentage (exclude AACT)

---

### Regular Maintenance Tasks

#### **Daily**
- Review failed ASN processing records
- Address error messages
- Verify tracking numbers populated

#### **Weekly**
- Run health check procedures
- Review error trends
- Check job queue performance
- Clear old processed ASN Data (if desired)

#### **Monthly**
- Review ASN volume trends
- Audit processing accuracy
- Test credentials (if near expiration)
- Review wholesaler service levels

#### **Quarterly**
- Archive old ASN Data records
- Review and optimize job queue schedule
- Update documentation
- Train staff on procedures

---

## Best Practices

### Setup Recommendations

1. **Job Queue Scheduling**
   - Import/translate jobs at :00 and :30
   - Processing job at :15 and :45
   - Ensures translation completes first
   - Adjust frequency based on volume

2. **Bin Configuration**
   - Create dedicated wholesaler bins
   - Locate near shipping area
   - Use clear naming (ESSENDANT-STAGE, SPR-STAGE)
   - Ensure adequate capacity

3. **Credential Management**
   - Store credentials securely in BC
   - Document credential renewal schedule
   - Test after any password changes
   - Keep backup contact at wholesaler

4. **Error Notifications**
   - Configure job queue to send error emails
   - Assign responsibility for monitoring
   - Set up alerts for critical failures
   - Document escalation procedures

---

### Operational Guidelines

1. **Daily Monitoring**
   - Assign staff member to check ASN processing daily
   - Review error messages first thing
   - Address failures within 1 hour
   - Keep log of recurring issues

2. **Exception Handling**
   - Document resolution steps for common errors
   - Escalate unknown errors promptly
   - Don't manually process without understanding why ASN failed
   - Update procedures based on learnings

3. **Manual Intervention**
   - Only manually receive POs when ASN truly can't process
   - Always mark ASN Data as processed after manual work
   - Document reason for manual intervention
   - Review trends to prevent future manual work

4. **Communication**
   - Notify warehouse when ASN processing fails
   - Inform customer service if tracking delayed
   - Coordinate with wholesalers on file format changes
   - Share processing statistics with management

---

### Performance Optimization

1. **Job Queue Tuning**
   - Monitor execution times
   - If slow, consider:
     - Increasing server resources
     - Splitting processing by wholesaler
     - Running during off-peak hours
   - Don't run too frequently (creates overhead)

2. **Data Retention**
   - Archive old ASN Data records (> 90 days)
   - Keep summary statistics
   - Maintain year-over-year for analysis
   - Balance storage vs query performance

3. **Warehouse Efficiency**
   - Optimize bin locations
   - Train warehouse on ASN-driven workflows
   - Use bin rankings for put-away efficiency
   - Monitor put-away/pick times

---

### Common Pitfalls to Avoid

❌ **Don't:**
- Process same ASN file multiple times
- Skip checking error messages
- Manually receive without marking ASN processed
- Ignore failed job queue executions
- Delete purchase orders with pending ASN
- Change warehouse setup during processing

✅ **Do:**
- Monitor daily for errors
- Clear errors and reprocess
- Keep job queues running
- Test after credential changes
- Document unusual situations
- Train backup staff

---

## Field Reference

### EssendantASNData Table

Central staging table for both wholesalers.

| Field Name | Type | Description | Usage Notes |
|------------|------|-------------|-------------|
| Entry No. | Integer | Primary key | Auto-increment |
| Qty. to Receive | Decimal | Quantity shipped by wholesaler | Matched to PO outstanding qty |
| Package Quantity | Decimal | Number of cartons/packages | Incremented by manifest files |
| Purchase Order No. | Code[20] | BC purchase order number | Must exist |
| Purchase Order Line No. | Integer | BC purchase line number | Must exist |
| Sales Order No. | Code[20] | Linked sales order | For special orders and drop ships |
| Sales Order Line No. | Integer | Linked sales line | For special orders and drop ships |
| Receipt Processed | Boolean | Purchase receipt completed | Set by ProcessASNReceipt |
| Ship Processed | Boolean | Warehouse pick/ship completed | Set by ProcessASNShip, special orders only |
| Tracking Numbers Processed | Boolean | Tracking populated | Set by ProcessTrackingNumbers |
| Last Error Message | Text[250] | Error from last processing attempt | Clear to retry |
| Wholesaler | Enum | Essendant or SPRichards | Identifies source |
| Wholesale Order | Boolean | Created via bumpdown | True for automated orders |
| Essendant Order Number | Code[7] | Essendant's internal order number | Used for manifest matching |
| SPRichards Order Number | Code[20] | SPR's DCS order number | Used for tracking lookup |
| Tracking Number | Text[100] | Shipment tracking number | Blank for AACT carrier |

---

### Essendant ASN Header Table

Raw ASN files from Essendant.

| Field Name | Type | Description |
|------------|------|-------------|
| SA Transmission ID | Guid | Unique file identifier |
| PO Number | Code[22] | Purchase order number |
| Essendant Order Number | Code[7] | Essendant's order reference |
| Carrier Code | Code[4] | Shipping carrier (AACT = W&L) |
| Translated in BC | Boolean | File has been processed |

---

### Essendant ASN Detail Table

Line-level shipment details from Essendant.

| Field Name | Type | Description |
|------------|------|-------------|
| SA Transmission ID | Guid | Links to header |
| Item Number v2 | Code[15] | Item shipped (with dashes/spaces) |
| Shipped Qty | Decimal | Quantity shipped |
| Tracking Number | Text[100] | Package tracking number |

---

### SPR ASN Manifest Hdr Table

Raw ASN files from SP Richards.

| Field Name | Type | Description |
|------------|------|-------------|
| SA Transmission ID | Guid | Unique file identifier |
| ASN FILE TYPE | Code[10] | 'SINGLE' or 'MULTI' |
| Translated in BC | Boolean | File has been processed |

---

### SPR ASN Order Summary Table

Order-level data from SP Richards ASN.

| Field Name | Type | Description |
|------------|------|-------------|
| DCSONO | Code[20] | SPR order number |
| CPONO | Text[50] | Customer (dealer) PO number |
| DREFN | Text[50] | Dealer reference (sales order no.) |

---

### SPR ASN Order Line Table

Line-level data from SP Richards ASN.

| Field Name | Type | Description |
|------------|------|-------------|
| DCSONO | Code[20] | SPR order number |
| REFCD1 | Text[30] | Reference code = BDSendReceiveDocEntryNo |
| MFRID | Code[10] | Manufacturer ID (item prefix) |
| STCKNO | Code[10] | Stock number (item suffix) |
| QSHIPPD | Decimal | Quantity shipped |

---

### Purchase Line Extensions

Fields added to Purchase Line for ASN processing.

| Field Name | Type | Description |
|------------|------|-------------|
| WholesaleOrderBMU | Boolean | Created via bumpdown |
| BDSendReceiveDocEntryNoBMU | Integer | Links to bumpdown transmission |
| WholesalerForASNProcessingBMU | Enum | Essendant or SP Richards |

---

### Sales Line Extensions

Fields added to Sales Line for ASN processing.

| Field Name | Type | Description |
|------------|------|-------------|
| WholesalerForASNProcessingBMU | Enum | Wholesaler for bin assignment |

---

## Advanced Topics

### ASN Data Consolidation Logic

When multiple ASN detail lines exist for the same purchase order line:

**Essendant Example:**
```
ASN Detail 1: Item 12345, Qty 5, Tracking ABC123
ASN Detail 2: Item 12345, Qty 3, Tracking ABC123
PO Line 10000: Item 12345, Outstanding Qty 8

Result:
EssendantASNData Entry 1: Qty to Receive = 8, Tracking = ABC123
(Consolidates 5 + 3 = 8)
```

**Why consolidation matters:**
- Reduces processing overhead
- Single receipt transaction per PO line
- Simplifies error handling
- Matches BC receiving paradigm

**Algorithm:**
1. Group ASN details by PO Number + PO Line Number + Item Number
2. Sum shipped quantities
3. Match against PO line outstanding quantity
4. Create single ASN Data record
5. If quantity exceeds outstanding, may create multiple records for multiple PO lines

---

### Partial Shipment Handling

When wholesaler ships less than ordered:

**Scenario:**
```
Purchase Order Line: Qty 100, Outstanding Qty 100
ASN Data: Qty to Receive 60

Receipt Processing:
- Receives 60 units
- PO Line Outstanding Qty becomes 40
- PO line remains open for future ASN

Next ASN:
- Qty to Receive 40
- Receives remaining 40 units
- PO line fully received
```

**Benefits:**
- Accurate receiving of partial shipments
- No manual quantity adjustments needed
- Purchase order tracking remains accurate
- Can ship special orders as received

---

### Drop Ship vs Special Order Processing Differences

| Aspect | Drop Shipment | Special Order |
|--------|---------------|---------------|
| **Ship To** | Customer address | Warehouse address |
| **Receipt Processing** | Post PO Receipt | Whse Receipt + Put-Away |
| **Ship Processing** | Skip (already at customer) | Whse Pick + Shipment |
| **Package Records** | Not created | Created from ASN |
| **Pick Required** | No | Yes |
| **Bin Assignment** | N/A | Wholesaler-specific bin |
| **Sales Ship** | Auto-linked | After pick registration |

**Identification:**
- Purchase Line `Drop Shipment = true` → Drop ship flow
- Purchase Line `Special Order = true` → Warehouse flow

---

### Tracking Number Source Enum

System supports multiple tracking number sources:

| Source | Description |
|--------|-------------|
| Manual | User entered manually |
| Essendant | From Essendant ASN |
| SPRichards | From SP Richards ASN |
| Carrier | From carrier API integration |
| Manifest | From delivery manifest |

**Why important:**
- Identifies data origin
- Audit trail
- Determines update rules
- Reporting accuracy

---

### Error Recovery and Retry Logic

ASN processing uses try/catch patterns with status flags:

**Receipt Processing:**
```
Try:
  ProcessPurchaseReceipt()
  Set Receipt Processed = true
  Clear Last Error Message
Catch:
  Set Receipt Processed = false
  Record Last Error Message
  Commit (saves error for review)
```

**Retry Mechanism:**
- Job queue runs on schedule
- Reprocesses all records with `Receipt Processed = false`
- Errors are idempotent (safe to retry)
- Manual fix + retry is standard workflow

**Best Practice:**
- Don't delete failed ASN Data records
- Fix underlying issue first
- Clear error message to trigger retry
- Monitor retry success rate

---

### Performance Considerations

**Large Volume Scenarios:**

For dealers processing 100+ ASN files per day:

1. **Batch Processing**
   - Process in smaller batches
   - Commit after each order
   - Prevents long-running transactions

2. **Parallel Processing**
   - Consider splitting by wholesaler
   - Separate job queues for Essendant/SPR
   - Reduce contention

3. **Database Optimization**
   - Index on Purchase Order No. + Line No.
   - Index on Receipt Processed flag
   - Archive old ASN Data regularly

4. **Monitoring**
   - Track processing times
   - Alert on increasing durations
   - Review slow queries

---

### Integration with Bumpdown System

ASN processing is tightly integrated with bumpdown:

**Flow:**
```
1. Bumpdown transmits PO to wholesaler
   - Creates BumpdownSendReceiveDoc
   - Records BDSendReceiveDocEntryNo on PO line

2. Wholesaler ships order
   - Generates ASN file
   - Includes dealer PO number

3. ASN Import matches to PO
   - Uses PO Number from ASN
   - For SPR: Uses REFCD1 = BDSendReceiveDocEntryNo
   - Ensures correct line matching

4. ASN processes receipt
   - Completes bumpdown cycle
   - Links SO to received goods
   - Enables shipment to customer
```

**Why BDSendReceiveDocEntryNo is Critical:**
- SP Richards ASN may have multiple PO lines with same item
- REFCD1 field contains BDSendReceiveDocEntryNo
- Provides exact line matching
- Prevents receiving to wrong line

---

### Extensibility Points

System includes integration events for customization:

#### **ProcessASNShip Codeunit**

```al
OnBeforeCreateAndRegisterPick(TempSalesHeaderToShip, IsHandled)
- Override pick creation logic
- Custom bin selection
- Alternate pick registration

OnCreateAndRegisterPickAfterCreatePick(TempSalesHeaderToShip, IsHandled)
- Post-pick creation logic
- Additional pick line updates
- Integration with third-party WMS

OnBeforeCreateCartonCounts(SalesOrderNo, ItemNo, QtyToShip, CartonQuantity, IsHandled)
- Custom package record creation
- Alternate package systems
- Integration with shipping software
```

**Use Cases:**
- Custom warehouse management system integration
- Third-party picking systems
- Advanced cartonization logic
- Shipping software integration

---

## Appendix: Common Scenarios

### Scenario 1: Same-Day Special Order Fulfillment

**Situation:** Customer places order for in-stock item at wholesaler, needs same-day delivery

**Process:**
1. **Morning:** Order transmitted via bumpdown
2. **Midday:** Wholesaler ships to dealer warehouse
3. **Afternoon:** 
   - ASN received and imported
   - Purchase receipt posted automatically
   - Warehouse put-away registered
   - Pick created automatically
   - Pick registered automatically
   - Ready for shipment to customer

**Timeline:**
- Order transmit: 9:00 AM
- Wholesaler ship: 11:00 AM
- ASN import: 1:00 PM (next job queue cycle)
- Receipt processing: 1:15 PM (next process cycle)
- Ready to ship: 1:30 PM

**Result:** Goods received, put away, picked, and ready for customer shipment in ~4 hours

---

### Scenario 2: Drop Ship Order with Tracking

**Situation:** Customer orders item for direct delivery from wholesaler

**Process:**
1. Sales order created with drop ship
2. Bumpdown transmits PO to wholesaler
3. Wholesaler ships directly to customer
4. ASN file received with tracking number
5. System:
   - Receives purchase order
   - Does NOT create warehouse documents
   - Populates tracking number on SO/PO
6. Customer service can provide tracking immediately

**No Warehouse Work:**
- No receipt to process
- No put-away
- No pick
- No shipment
- Just tracking information

---

### Scenario 3: Partial Shipment Over Multiple Days

**Situation:** Ordered 100 units, wholesaler ships in two shipments

**Day 1:**
- ASN received: Qty 60
- System receives 60 units
- Creates pick for 60 units
- 6 packages created
- PO still open for remaining 40

**Day 2:**
- ASN received: Qty 40
- System receives 40 units
- Creates pick for 40 units
- 4 packages created
- PO now fully received

**Total:** 10 packages, 100 units, two shipments

---

### Scenario 4: ASN Arrives Before PO Created

**Situation:** Timing issue where ASN imports before bumpdown completes

**Problem:**
- ASN file downloaded
- Translation attempts to match PO
- PO doesn't exist yet
- Error: "Purchase order does not exist"

**Resolution:**
1. Wait for bumpdown to complete
2. Purchase order created
3. Next job queue cycle retries ASN
4. Successfully matches and processes

**Prevention:**
- Ensure adequate delay between bumpdown and ASN processing
- Typical: Bumpdown runs :00, ASN processing runs :30

---

### Scenario 5: Multiple Items on Single PO, Different Shipments

**Situation:** PO has 3 items, each ships separately

**ASN 1 (Day 1):**
- Item A: Qty 50, Tracking ABC123

**ASN 2 (Day 2):**
- Item B: Qty 25, Tracking DEF456

**ASN 3 (Day 3):**
- Item C: Qty 10, Tracking GHI789

**Processing:**
- Three separate receipt transactions
- Three separate put-aways
- Three separate picks
- Three separate package sets
- All link to same sales order
- Three tracking numbers on SO

**Result:** Accurate receiving and shipment for split deliveries

---

## Support and Additional Resources

### Getting Help

1. **Check This Documentation**
   - Review troubleshooting section for your issue
   - Follow diagnostic steps
   - Try suggested solutions

2. **Review System Logs**
   - ASN Data page (error messages)
   - Job Queue Log Entries
   - Essendant/SPR raw file tables

3. **Contact BMI Support**
   - Website: https://bmiusa.freshdesk.com/support/home
   - Include: ASN Data Entry No., error messages, PO/SO numbers
   - Specify: Wholesaler, date/time of failure
   - Attach: Screenshots of error messages

### Related Documentation

- **BUMPDOWN.md** - Purchase order transmission system
- **Bumpdown Setup Guide** - Configuration details
- **Wholesale Sourcing** - Vendor priority and sourcing rules
- **Delivery Manifest** - Outbound shipment tracking
- **Warehouse Management** - BC warehouse processes

### Glossary

| Term | Definition |
|------|------------|
| **ASN** | Advance Ship Notice - electronic shipment notification |
| **ELink** | Essendant's API for ASN, invoice, and price data |
| **UNILINK** | Essendant's API for purchase order transmission |
| **DCS Order Number** | SP Richards internal order identifier |
| **REFCD1** | SP Richards reference code = BDSendReceiveDocEntryNo |
| **AACT** | Carrier code for W&L (will-call) shipments |
| **Manifest** | Package/carton count file |
| **Special Order** | Wholesaler ships to dealer warehouse |
| **Drop Shipment** | Wholesaler ships directly to customer |
| **Put-Away** | Warehouse process of moving received goods to bins |
| **Pick** | Warehouse process of retrieving goods for shipment |

---

## Document Revision History

| Date | Version | Changes |
|------|---------|---------|
| 2025-12-23 | 1.0 | Initial documentation - ASN import, translation, and processing |

---

*For questions or issues not covered in this documentation, contact BMI Support at https://bmiusa.freshdesk.com/support/home*
