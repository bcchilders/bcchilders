# Wholesale Sourcing

**Document Version**: 1.0  
**Last Updated**: December 23, 2025  
**Feature Status**: Production

---

## 1. OVERVIEW

The **Wholesale Account Priority Groups** system controls the sequence in which the system attempts to source items from different wholesale vendors during automated order fulfillment. This feature works in conjunction with **Wholesale Account Groups** to determine both *which* wholesale accounts are available for a customer and *in what order* those accounts should be attempted for each item.

The system uses a two-dimensional matching approach: **Wholesale Account Groups** (customer/ship-to centric) define the available wholesale accounts based on customer requirements and business rules, while **Wholesale Account Priority Groups** (item-centric) define the preferred vendor sequence for each item based on product availability, pricing, or strategic sourcing decisions. When a sales order is processed through the Bumpdown system, these two configurations intersect to create a prioritized list of wholesale vendors to attempt for each sales line.

This dual-layer approach enables sophisticated sourcing strategies where different customers can access different sets of wholesale accounts (via Wholesale Account Groups), while different items follow different vendor preference sequences (via Wholesale Account Priority Groups). The system automatically handles the intersection logic, filtering wholesale accounts based on drop-ship requirements, transmission time windows, and item-vendor relationships to ensure efficient and accurate order fulfillment.

---

## 2. HOW IT WORKS

### Automated Processing

**Bumpdown Sourcing Process**:
1. Sales order line enters bumpdown processing (wholesale order flagged, no existing purchase order)
2. System retrieves **Wholesale Account Group Code** from sales line (inherited from Ship-to Address, Customer, or Sourcing Options)
3. System retrieves **WS Account Priority Group Code** from sales line (inherited from Item master)
4. **WholesaleSourcingMgtBMU.GetWSAccountPriorityBuffer()** builds prioritized account list by:
   - Finding all accounts in the Wholesale Account Group
   - Matching each account to the WS Account Priority Group to get priority numbers
   - Filtering based on drop shipment requirements
   - Filtering based on transmission time windows (current time must be between start/end times)
   - Validating Item Vendor records exist for each account
   - Sorting by priority (lowest number = highest priority)
5. System attempts purchase order transmission to accounts in priority sequence
6. On success, sales line is sourced and process stops
7. On failure, system "bumps down" to next priority account and retries
8. Process continues until line is sourced or all accounts are exhausted

### Manual Operations

Users can override sourcing behavior by:
- **Changing WS Account Priority Group on Item Card**: Alters default vendor sequence for all future orders of that item
- **Configuring Sourcing Options**: Overrides Wholesale Account Group at location/customer/item level
- **Manual Vendor Selection**: Bypassing automatic sourcing by directly setting vendor on sales line

### Data Flow

**Key Tables**:
- **Item** (71552585 extension): Stores default WSAccountPriorityGroupCodeBMU
- **Customer** (71552590 extension): Stores default WholesaleAccountGroupCodeBMU
- **Ship-to Address** (71552601 extension): Stores Ship-to specific WholesaleAccountGroupCodeBMU (overrides customer default)
- **Sales Line** (extensions): Inherits both codes, used during bumpdown processing
- **WSAccountPriorityGroupBMU** (71552641): Header table for priority group definitions
- **WSAccountPriorityGroupLineBMU** (71552642): Priority assignments for wholesaler accounts (Primary Key: Group Code, Wholesaler, Account Code)
- **WholesaleAccountGroupBMU** (71552639): Header table for account group definitions
- **WholesaleAccountGroupLineBMU** (71552640): Account group membership (Primary Key: Group Code, Wholesaler, Account Code)
- **WSAccountPriorityBufferBMU** (71552643): Temporary buffer storing prioritized accounts for bumpdown processing

**Integration Points**:
- **Bumpdown Management**: Primary consumer of priority group logic
- **Sourcing Options**: Can override Wholesale Account Group assignments
- **Load Factor Management**: Uses Wholesale Account Group for cost calculations
- **Item Vendor Records**: Required for account validation during sourcing

---

## 3. SETUP AND CONFIGURATION

### Wholesale Account Group Setup

**Search**: "Wholesale Account Groups"

**Steps**:
1. Open **Wholesale Account Group List** page
2. Create new group with unique **Code** (10 characters)
3. Enter descriptive **Description** (e.g., "Drop Ship Only", "East Coast Customers")
4. Add **Lines** for each wholesale account in the group:
   - **Wholesaler**: Select Essendant or SP Richards
   - **Account Code**: Select specific account (must exist in Essendant Accounts or SP Richards Accounts)
5. Each line automatically inherits account properties: Vendor No., Drop Shipment flag, Transmission Start/End Time

**Required Fields**:
- Code (primary key)
- Description
- Lines: Wholesaler, Account Code

**Dependencies**:
- Essendant Accounts and SP Richards Accounts must be configured first
- Vendor records must exist and be linked to wholesale accounts

### Wholesale Account Priority Group Setup

**Search**: "Wholesale Account Priority Groups"

**Steps**:
1. Open **Wholesale Account Priority Group List** page
2. Create new priority group with unique **Code** (10 characters)
3. Enter descriptive **Description** (e.g., "HP Products First", "Essendant Priority")
4. Add **Lines** defining priority sequence:
   - **Wholesaler**: Select Essendant or SP Richards
   - **Account Code**: Select specific account
   - **Priority**: Enter priority number (1 = highest priority, 2 = second priority, etc.)
5. Priority numbers must be unique within each priority group
6. Lower numbers are attempted first during bumpdown

**Required Fields**:
- Code (primary key)
- Description
- Lines: Wholesaler, Account Code, Priority (minimum value: 1)

**Priority Assignment Rules**:
- Priority 1 is always attempted first
- Duplicate priorities within same group are not allowed
- Gaps in priority sequence are allowed (e.g., 1, 3, 5)
- Only accounts included in lines will be attempted during sourcing

### Assignment to Master Data

**Item Card** (OfficeProducts FastTab):
- **Wholesale Account Priority Group Code**: Defines default vendor sequence for this item
- Inherited to sales lines when item is selected
- Search: "Items" → Select item → OfficeProducts tab

**Customer Card** (OfficeProducts FastTab):
- **Wholesale Account Group Code**: Defines available wholesale accounts for this customer
- Inherited to sales lines unless overridden by Ship-to Address
- Search: "Customers" → Select customer → OfficeProducts tab

**Ship-to Address** (OfficeProducts FastTab):
- **Wholesale Account Group Code**: Overrides customer's default for this ship-to location
- Takes precedence over customer-level assignment
- Search: "Ship-to Addresses" → Select address → OfficeProducts tab

**Sourcing Options** (Advanced):
- **Wholesale Account Group Code**: Can override at location/customer/item/attribute level
- Highest priority override (trumps Ship-to and Customer assignments)
- Search: "Sourcing Options"

---

## 4. USER GUIDE

### Task 1: Create a New Wholesale Account Group

1. Search for "Wholesale Account Groups"
2. Select **New** to create a new group
3. Enter **Code** (e.g., "DROPSHIP") and **Description** (e.g., "Drop Shipment Only Accounts")
4. Navigate to **Lines** section
5. For each account to include:
   - Select **Wholesaler** (Essendant or SP Richards)
   - Select **Account Code** from dropdown
6. Save the record

### Task 2: Create a Wholesale Account Priority Group for an Item Category

1. Search for "Wholesale Account Priority Groups"
2. Select **New** to create a new priority group
3. Enter **Code** (e.g., "HPPAPER") and **Description** (e.g., "HP Paper Priority Sequence")
4. Navigate to **Lines** section
5. Add accounts in priority order:
   - Line 1: Wholesaler = "Essendant", Account = "ESS-001", Priority = 1
   - Line 2: Wholesaler = "SP Richards", Account = "SPR-001", Priority = 2
   - Line 3: Wholesaler = "Essendant", Account = "ESS-002", Priority = 3
6. Save the record

### Task 3: Assign Priority Group to Items

1. Search for "Items"
2. Select item(s) to configure
3. Navigate to **OfficeProducts** FastTab
4. Set **Wholesale Account Priority Group Code** field to desired group (e.g., "HPPAPER")
5. Save the item
6. Repeat for all items requiring this priority sequence (consider using Item Templates for bulk assignment)

### Task 4: Assign Wholesale Account Group to Customer

1. Search for "Customers"
2. Select customer to configure
3. Navigate to **OfficeProducts** FastTab
4. Set **Wholesale Account Group Code** field (e.g., "DROPSHIP")
5. Save the customer
6. All sales orders for this customer will inherit this Wholesale Account Group

### Task 5: Override Wholesale Account Group for Specific Ship-to Address

1. Search for "Ship-to Addresses"
2. Select ship-to address to configure
3. Navigate to **OfficeProducts** FastTab
4. Set **Wholesale Account Group Code** field (e.g., "EASTCOAST")
5. Save the record
6. Sales orders shipping to this address will use this Wholesale Account Group instead of customer default

---

## 5. TROUBLESHOOTING

### Problem: Sales lines are not being sourced during bumpdown

**Causes**:
- No intersection between Wholesale Account Group and WS Account Priority Group (accounts don't match)
- Item Vendor records don't exist for any accounts in the intersection
- Transmission time window restrictions preventing all accounts from being eligible
- Drop shipment requirements not aligned between customer settings and account configuration

**Solutions**:
1. Verify Item has WS Account Priority Group Code assigned
2. Verify Customer/Ship-to has Wholesale Account Group Code assigned
3. Check that at least one account exists in BOTH groups (same Wholesaler + Account Code)
4. Confirm Item Vendor records exist for vendor numbers linked to intersecting accounts (Search "Item Vendors")
5. Check current time is within Transmission Start/End Time for accounts (see Essendant/SP Richards Account cards)
6. Verify drop shipment flags align: if customer requires drop ship, ensure accounts in Wholesale Account Group have "Drop Shipment" = true

### Problem: Wrong vendor is being selected first

**Causes**:
- Priority numbers not configured correctly in WS Account Priority Group
- Multiple Priority Groups exist with similar codes causing confusion
- Sourcing Options overriding expected behavior at higher precedence

**Solutions**:
1. Open the WS Account Priority Group assigned to the item
2. Verify Priority field values: Priority 1 should be the preferred vendor, Priority 2 the second choice, etc.
3. Lower priority numbers are attempted first—ensure numbering reflects intended sequence
4. Check for duplicate Priority values within the group (system should prevent but verify)
5. Review Sourcing Options setup (search "Sourcing Options") for overrides that may change Wholesale Account Group assignment

### Problem: Duplicate priority error when adding priority group line

**Cause**: Attempting to assign same Priority number to multiple accounts within same priority group

**Solution**: Each Priority number must be unique within a WS Account Priority Group. Assign sequential or distinct priority values (e.g., 1, 2, 3 or 10, 20, 30) to each account line.

### Problem: Customer orders fail sourcing but other customers succeed with same items

**Causes**:
- Customer or Ship-to Address has different Wholesale Account Group Code that doesn't intersect with item's priority group
- Customer-specific Sourcing Options overriding expected behavior
- Drop shipment or transmission time restrictions specific to customer's assigned accounts

**Solutions**:
1. Compare Wholesale Account Group Code on failing customer vs. working customer
2. Open both Wholesale Account Groups and verify account membership differences
3. Ensure failing customer's Wholesale Account Group contains accounts that also exist in item's WS Account Priority Group
4. Review Sourcing Options filtered by customer number to identify overrides

### Problem: Sales line shows "Failed Wholesale Line Exists" error

**Causes**:
- All accounts in priority sequence were attempted but none successfully accepted the order
- System exhausted priority list without successful transmission
- Prior transmission attempts failed for valid reasons (out of stock, account issues, transmission errors)

**Solutions**:
1. Search for "Failed Wholesale Lines" to view detailed failure reasons for each account attempt
2. Review failure messages to determine root cause (out of stock, invalid item, transmission error, etc.)
3. If temporary issue (e.g., transmission timeout), clear failed wholesale line flags and re-run bumpdown
4. If account-specific issue, consider adjusting WS Account Priority Group to remove problematic account or change priority order
5. If item not available from any account, consider manual vendor selection or special order processing

---

## 6. MONITORING AND REPORTING

### Key Monitoring Pages

**Wholesale Sourcing Role Center**:
- Search: "Wholesale Sourcing Role Center"
- Displays failed wholesale lines cue for immediate attention
- Provides access to all sourcing-related pages and reports

**Failed Wholesale Lines**:
- Search: "Failed Wholesale Lines"
- Shows all sales lines that failed sourcing through all priority attempts
- Displays failure reasons, attempted accounts, and error messages
- Key fields: Sales Order No., Line No., Item No., Customer, Wholesaler attempted, Failure Reason
- Use to diagnose systemic sourcing issues

**Sales Line FactBox** (on Sales Order):
- Displays Wholesale Account Group Code and WS Account Priority Group Code for selected line
- Shows "Sourcing Option Found" indicator
- Provides quick visibility into sourcing configuration for troubleshooting

### Important Reports

**Bumpdown Processing Log**:
- Review system log entries for detailed bumpdown processing history
- Filter by date range, customer, or item to analyze sourcing patterns
- Identify frequently failing accounts or items

**Item Vendor Analysis**:
- Use standard BC "Item Vendors" report to verify vendor relationships
- Cross-reference with Wholesale Account Group membership to ensure Item Vendor records exist

### Health Checks

**Daily**:
- Review Failed Wholesale Lines count on Wholesale Sourcing Role Center
- Address lines with "No Valid Vendors" status
- Investigate repeated failures for specific items or customers

**Weekly**:
- Audit Wholesale Account Priority Group assignments for new items
- Verify Wholesale Account Group assignments for new customers
- Review Sourcing Options for conflicting or outdated rules

**Monthly**:
- Analyze sourcing success rates by Wholesale Account Group
- Review priority group effectiveness—are first-priority accounts consistently succeeding?
- Validate transmission time windows align with business hours and wholesaler schedules

---

## 7. BEST PRACTICES

**Setup and Configuration**:
- Use descriptive codes for both Wholesale Account Groups and WS Account Priority Groups (e.g., "DROPSHIP", "HPPAPER") rather than cryptic abbreviations
- Maintain consistent priority numbering schemes (e.g., increments of 10: 10, 20, 30) to allow easy insertion of intermediate priorities later
- Document business rationale for priority sequences in the Description field (e.g., "Best pricing", "Fastest delivery")

**Account Group Strategy**:
- Create Wholesale Account Groups based on customer requirements: drop-ship capabilities, geographic regions, service levels, or credit terms
- Limit Wholesale Account Group size—fewer accounts mean faster processing and clearer troubleshooting
- Review and consolidate redundant account groups quarterly to reduce complexity

**Priority Group Strategy**:
- Align WS Account Priority Groups with product categories or supplier relationships—avoid overly granular item-specific groups
- Set Priority 1 to the vendor with best combination of availability, pricing, and delivery speed
- Consider multiple priority sequences for different item characteristics (e.g., "DROPSHIP-PRI", "STOCK-PRI")

**Master Data Management**:
- Use Item Templates to assign WS Account Priority Group Code during item creation—ensures consistent configuration
- Assign Wholesale Account Group Code at Customer level as default, override at Ship-to Address level only when necessary
- Maintain Item Vendor records for all active vendor relationships—missing records prevent sourcing

**Operational Excellence**:
- Monitor Failed Wholesale Lines daily and address root causes promptly
- Set transmission time windows generously to avoid unnecessary filtering—only restrict when wholesaler systems have known downtime
- Test priority sequences with small order batches before rolling out to all items in a category

**Critical Pitfalls to Avoid**:
- **Empty Intersection**: Never assign Wholesale Account Group and WS Account Priority Group combinations that have no accounts in common—sales lines will always fail
- **Missing Item Vendors**: Always create Item Vendor records for vendors in your priority groups—system requires these for validation
- **Duplicate Priorities**: System prevents duplicate priority values within a group, but avoid renumbering mid-operation—can cause inconsistent sourcing
- **Overly Complex Sourcing Options**: Excessive Sourcing Option rules can override expected behavior—keep override rules minimal and well-documented
- **Ignoring Transmission Windows**: Setting unrealistic transmission time windows (e.g., midnight to 1 AM) effectively disables accounts—align with actual wholesaler availability

---

## 8. FIELD REFERENCE

### WSAccountPriorityGroupBMU Table (71552641)

| Field | Type | Description |
|-------|------|-------------|
| Code | Code[10] | Primary key, unique identifier for priority group |
| Description | Text[50] | Human-readable description of group purpose |

### WSAccountPriorityGroupLineBMU Table (71552642)

| Field | Type | Description |
|-------|------|-------------|
| WS Account Priority Group Code | Code[10] | Foreign key to priority group header |
| Wholesaler | Enum (Essendant, SPRichards) | Wholesaler type for this account |
| Account Code | Code[20] | Specific wholesaler account code (FK to EssendantAccountBMU or SPRichardsAccountBMU) |
| Priority | Integer | Priority sequence number (1 = highest, must be unique within group, minimum value 1) |

### WholesaleAccountGroupBMU Table (71552639)

| Field | Type | Description |
|-------|------|-------------|
| Code | Code[10] | Primary key, unique identifier for account group |
| Description | Text[50] | Human-readable description of group purpose |
| Priority | Integer | **OBSOLETE** (Pending removal, tag 1.45) - no longer used for precedence determination |

### WholesaleAccountGroupLineBMU Table (71552640)

| Field | Type | Description |
|-------|------|-------------|
| Wholesale Account Group Code | Code[10] | Foreign key to account group header |
| Wholesaler | Enum (Essendant, SPRichards) | Wholesaler type for this account |
| Account Code | Code[20] | Specific wholesaler account code (FK to EssendantAccountBMU or SPRichardsAccountBMU) |

**Key Methods**:
- `DropShipment()`: Returns drop shipment flag from underlying account record
- `TransmissionStartTime()`: Returns transmission start time from underlying account record
- `TransmissionEndTime()`: Returns transmission end time from underlying account record
- `VendorNo()`: Returns BC vendor number linked to wholesaler account

### WSAccountPriorityBufferBMU Table (71552643 - Temporary)

| Field | Type | Description |
|-------|------|-------------|
| Wholesaler | Enum | Wholesaler type |
| Account Code | Code[20] | Wholesaler account code |
| Priority | Integer | Priority number from WS Account Priority Group |
| Drop Shipment | Boolean | Drop shipment flag from account |
| Vendor No. | Code[20] | BC vendor number for this account |
| Transmission Start Time | Time | Earliest time for transmission |
| Transmission End Time | Time | Latest time for transmission |
| Transmit | Boolean | Flag indicating account passed validation and should be attempted |
| Item No. | Code[20] | Item being sourced (for reference) |
| Item Line No. | Integer | Sales line number being sourced |
| Failed | Boolean | Flag indicating sourcing attempt failed for this account |

### Item Table Extension (71552585)

| Field | Type | Description |
|-------|------|-------------|
| WSAccountPriorityGroupCodeBMU | Code[10] | Default priority group for vendor sequencing when item is ordered |

### Customer Table Extension (71552590)

| Field | Type | Description |
|-------|------|-------------|
| WholesaleAccountGroupCodeBMU | Code[10] | Default account group defining available wholesale accounts for customer |

### Ship-to Address Table Extension (71552601)

| Field | Type | Description |
|-------|------|-------------|
| WholesaleAccountGroupCodeBMU | Code[10] | Account group override for this ship-to address (takes precedence over customer default) |

### Sales Line Table Extensions (71552602, WholesaleAccountGroupCodeBMU)

| Field | Type | Description |
|-------|------|-------------|
| WSAccountPriorityGroupCodeBMU | Code[10] | Inherited from Item, defines vendor priority sequence for this line |
| WholesaleAccountGroupCodeBMU | Code[10] | Inherited from Ship-to Address, Customer, or Sourcing Options, defines available accounts for this line |

### SourcingOptionBMU Table (71552627)

| Field | Type | Description |
|-------|------|-------------|
| Wholesale Account Group Code | Code[10] | Account group override when sourcing option conditions are met (highest precedence override) |

---

## 9. ADVANCED TOPICS

### Priority Buffer Generation Algorithm

The `WholesaleSourcingMgtBMU.GetWSAccountPriorityBuffer()` procedure performs sophisticated intersection logic:

1. **Account Retrieval**: Iterates through all WholesaleAccountGroupLineBMU records matching the Wholesale Account Group Code
2. **Priority Matching**: For each account, looks up corresponding WSAccountPriorityGroupLineBMU record matching both Wholesaler and Account Code
3. **Buffer Population**: Creates WSAccountPriorityBufferBMU entries with priority numbers, account details, and properties from wholesaler accounts
4. **Drop Shipment Filtering**: If sales line requires drop shipment (DropShipOnly or DropShipment flags), filters buffer to Drop Shipment = true accounts only
5. **Transmission Window Filtering**: Filters buffer to accounts where current time is between Transmission Start Time and Transmission End Time
6. **Item Vendor Validation**: For each remaining account, validates Item Vendor record exists for Item No. + Vendor No. combination
7. **Final Buffer**: Returns buffer with Transmit = true flag set only for accounts passing all filters, sorted by Priority

**Key Decision Points**:
- No priority match → account excluded from buffer (silent skip)
- Priority match but failed filters → account excluded from buffer
- Empty final buffer → "No Valid Vendors" condition, bumpdown will fail

### Sourcing Options Precedence Logic

Wholesale Account Group Code resolution follows strict precedence hierarchy in `SourcingOptionMgtBMU.GetSourcingOptionValues()`:

1. **Sourcing Options**: If active sourcing option matches (location, customer, ship-to, type, type no., UOM, quantity), uses its Wholesale Account Group Code (highest precedence)
2. **Ship-to Address**: If no sourcing option match, uses WholesaleAccountGroupCodeBMU from Ship-to Address (if populated)
3. **Customer**: If Ship-to Address not populated, uses WholesaleAccountGroupCodeBMU from Customer (lowest precedence)
4. **Blank/Empty**: If none of above populated, Wholesale Account Group Code remains blank and sourcing may fail

**Sourcing Options Matching Hierarchy**:
- Item-level rules: Highest specificity, matches specific Item No. + UOM + Quantity threshold
- Item Category-level rules: Medium specificity, matches Item Category Code + UOM + Quantity threshold
- Item Attribute-level rules: Pattern matching, matches items with specific attribute values

Sourcing Options also support Customer No. and Ship-to Code filters, creating complex multi-dimensional matching logic. Priority field on Sourcing Option determines which rule wins when multiple rules match.

### Bumpdown Group Assembly

The `BumpdownMgt.GroupSalesLines()` procedure creates bumpdown groups to batch sales lines with identical priority buffer sequences:

1. For each sales line, generates priority buffer using Wholesale Account Group Code + WS Account Priority Group Code
2. Compares priority buffer to existing bumpdown groups
3. If identical buffer exists (same accounts, same priority sequence), assigns line to existing group
4. If unique buffer, creates new bumpdown group ID and assigns line
5. Result: Lines with identical sourcing sequences are transmitted together, maximizing transmission efficiency

**Benefits**:
- Reduces API calls to wholesalers (batch transmission)
- Ensures consistent processing order within groups
- Enables parallel transmission to different wholesaler accounts for different groups

### Event Subscriber Integration

`OrderFulfillmentEventMgt.SalesLineOnBeforeValidateWSAccountPriorityGroupCodeBMU()` validates WS Account Priority Group existence when code is changed on sales line. This event subscriber prevents orphaned references and provides immediate feedback if user enters invalid code.

**Extensibility Points**:
- Extend WholesalerBMU enum to add new wholesalers (e.g., third-party distributors)
- Subscribe to `OnBeforeGetWSAccountPriorityBuffer` event (if published) to inject custom filtering logic
- Extend SourcingOptionBMU table to add custom sourcing dimensions
- Create custom reports on WSAccountPriorityGroupLineBMU to analyze vendor usage patterns

### Complex Scenario: Multi-Location Sourcing with Regional Restrictions

**Scenario**: Company has three locations (East, West, Central) and wants East Coast customers to use East Coast wholesale accounts, West Coast customers to use West Coast accounts:

**Solution**:
1. Create Wholesale Account Groups: "EAST-ACCTS", "WEST-ACCTS", "CENTRAL-ACCTS"
2. Populate each with region-appropriate wholesaler accounts
3. Assign Wholesale Account Group Code to customers based on Ship-to Address region
4. Create WS Account Priority Groups by product category (e.g., "PAPER-PRI", "TONER-PRI")
5. Assign WS Account Priority Group Code to items based on category
6. System automatically intersects customer's regional accounts with item's priority sequence

**Alternative**: Use Sourcing Options with Location Code filter to override Wholesale Account Group per location, enabling centralized item configuration with location-specific account assignment.

---

## 10. APPENDIX

### Quick Reference: Resolution Flow

```
Sales Order Created
    ↓
Sales Line populated with Item No.
    ↓
Item.WSAccountPriorityGroupCodeBMU → Sales Line.WSAccountPriorityGroupCodeBMU
    ↓
Sourcing Option Match? → Yes → Use Sourcing Option's Wholesale Account Group Code
    ↓ No
Ship-to Address.WholesaleAccountGroupCodeBMU populated? → Yes → Use Ship-to Address code
    ↓ No
Customer.WholesaleAccountGroupCodeBMU → Sales Line.WholesaleAccountGroupCodeBMU
    ↓
Bumpdown Processing Initiated
    ↓
GetWSAccountPriorityBuffer(WholesaleAccountGroupCodeBMU, WSAccountPriorityGroupCodeBMU)
    ↓
Build Priority Buffer (intersection + filtering)
    ↓
Attempt Priority 1 Account → Success? → Done
    ↓ Failure
Attempt Priority 2 Account → Success? → Done
    ↓ Failure
Continue through priority sequence...
    ↓ All Failed
Create Failed Wholesale Line
```

### Glossary

**Account Code**: Unique identifier for a specific wholesaler account (e.g., "ESS-001" for Essendant account #1)

**Bumpdown**: Process of attempting to source sales lines from wholesalers in priority sequence, "bumping down" to next vendor on failure

**Drop Shipment**: Fulfillment method where wholesaler ships directly to customer; wholesale account property determining eligibility

**Intersection**: Set of accounts that exist in both Wholesale Account Group and WS Account Priority Group—only intersecting accounts are attempted during sourcing

**Item Vendor**: BC standard relationship record linking an item to a vendor with purchasing details; required for sourcing validation

**Priority Buffer**: Temporary in-memory table (WSAccountPriorityBufferBMU) containing sorted, filtered accounts to attempt for a sales line

**Sourcing Options**: Advanced rules-based system that can override Wholesale Account Group assignment based on location, customer, item attributes, and quantity thresholds

**Transmission Window**: Time range (Start Time to End Time) when wholesaler account is available for order transmission; used for filtering eligible accounts

**Wholesaler**: Supported wholesale distributor types in the system: Essendant or SP Richards

**Wholesale Account Group**: Customer/Ship-to centric grouping defining which wholesale accounts are available for a customer or location

**WS Account Priority Group**: Item-centric grouping defining the preferred vendor sequence for sourcing items

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-23 | Documentation Team | Initial comprehensive documentation |

---

## Related Documentation

- [Bumpdown System](BUMPDOWN.md) - Automated wholesale order sourcing process
- [Load Factor Management](LOAD_FACTOR_MANAGEMENT.md) - Cost markup system using Wholesale Account Group dimension
- [Wholesale ASN](WHOLESALE_ASN.md) - Advance ship notice processing from wholesalers
- [Wholesale Invoicing](WHOLESALE_INVOICING.md) - Invoice receipt and processing from wholesalers

---

**Support Contact**: BMI Support Team  
**Feature Owner**: Wholesale Operations  
**System**: BMI SupplyAutomate for Microsoft Dynamics 365 Business Central
