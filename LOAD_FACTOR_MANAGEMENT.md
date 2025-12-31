# Load Factor Management

## 1. OVERVIEW

Load Factor Management provides a flexible system for automatically calculating and applying cost adjustments (loads) to vendor purchase prices when determining sales line costs in BMI SupplyAutomate. The feature solves the business problem of accurately accounting for overhead costs, freight, handling fees, and other indirect costs that must be included in the final unit cost to ensure proper profit margins.

A **Load Factor** is a percentage markup applied to the base vendor cost to arrive at a "loaded cost" that reflects the true cost of goods. For example, a 15% load factor applied to a $100 vendor cost results in a $115 loaded cost on the sales line. The system uses a sophisticated hierarchy-based matching engine that can apply different load factors based on specific combinations of Item, Customer, Vendor, Item Category, Purchasing Code, and Wholesale Account Group.

The feature integrates deeply with sales order processing, purchase order creation, and pricing calculations. Load factors are automatically applied when sales lines are created or modified, when purchase orders are generated from sales orders, and when costs change through receipt processing. This ensures that unit costs on sales orders always reflect current vendor pricing plus appropriate overhead allocations.

### Key Terminology

- **Load Factor %**: The percentage markup applied to base vendor cost (e.g., 15% = 0.15)
- **Loaded Cost**: Vendor cost × (1 + Load Factor %) = Final unit cost on sales line
- **Hierarchy Number**: System-calculated priority ranking (lower number = higher priority)
- **Load Factor Type**: Defines which fields must be populated (Item, Customer, Customer & Item, Vendor, Vendor & Item, Item Category Code, Wholesale Group Account, Wholesale Group Account & Item)

### System Integration

- **Sales Order Processing**: Automatically applies load factors when sales lines are created, calculating Unit Cost (LCY) from vendor cost
- **Purchase Order Generation**: Maintains load factor linkage when creating purchase orders from sales orders
- **Cost-Based Pricing**: Works with Extended Sales Pricing to calculate prices based on loaded costs
- **Item Master**: Reads Main Vendor No. and Main Wholesaler No. to determine which vendor cost to load
- **Purchase Price Management**: Retrieves current vendor costs from BC price lists
- **Gross Profit Calculations**: Updates gross profit percentage and dollars based on loaded cost vs selling price

## 2. HOW IT WORKS

### Load Factor Resolution Process

When a sales line is created or modified, the system follows this sequence:

1. **Trigger Events**: Sales line item validation, purchasing code change, suggested vendor change, or purchase order receipt
2. **Base Cost Retrieval**: 
   - If Suggested Vendor specified, uses that vendor's cost
   - Otherwise, uses Main Vendor No. from item master
   - Retrieves vendor cost from BC purchase price lists for the as-of date, UOM, and quantity
3. **Load Factor Lookup**: Calls `GetLoadFactorPct()` with parameters:
   - Item No., Customer No., Vendor No., Item Category Code, Purchasing Code, As-of Date, Wholesale Account Group
4. **Hierarchy-Based Matching**:
   - Filters active load factors by date range (Starting Date ≤ As-of Date ≤ Ending Date)
   - Applies filters allowing blank or matching values for all dimensions
   - Sorts by Hierarchy No. ascending (most specific first)
   - Returns first matching load factor percentage
5. **Special Purchasing Code Logic**: If match found with blank Purchasing Code but sales line has Purchasing Code, searches again for exact purchasing code match
6. **Cost Calculation**: Loaded Unit Cost = Base Vendor Cost × (1 + Load Factor Pct / 100)
7. **Sales Line Update**:
   - Sets Unit Cost (LCY) to loaded cost
   - Stores Load Factor Pct BMU for reference
   - Calculates Gross Profit Pct = ((Unit Price - Unit Cost) / Unit Price) × 100
   - Calculates Gross Profit Dollars = Unit Price - Unit Cost

### Manual Operations

Users interact with load factors through:

- **Load Factors Page**: Create, activate, and maintain load factor rules
- **Sales Order Lines**: View Load Factor Pct BMU field to see which load was applied
- **Cost Override**: Manually changing Unit Cost (LCY) clears load factor percentage (sets to 0)
- **Vendor Changes**: Changing Suggested Vendor No. recalculates loaded cost with new vendor pricing and applicable load factor
- **Purchasing Code Selection**: Changing purchasing code re-evaluates load factor matches

### Automated Processing

The system automatically:

- **On Sales Line Creation**: Applies load factor when item is selected
- **On Purchase Order Creation**: Recalculates sales line cost with load factor when requisition line becomes purchase order (via Make Order batch job)
- **On Purchase Invoice Receipt**: Updates sales line cost when purchase invoice line references a receipt tied to a sales order (drop shipment or special order)
- **On Cost Change**: Recalculates gross profit when unit price changes

### Data Flow

**Tables Created/Updated**:

- `LoadFactorsBMU` (71552775): Master table storing load factor rules
- `Sales Line` (extended): Fields added - LoadFactorPctBMU, GrossProfitPctBMU, GrossProfitDollarsBMU
- `Sales Invoice Line` (extended): LoadFactorPctBMU copied from sales line
- `Sales Cr.Memo Line` (extended): LoadFactorPctBMU copied from sales line

**Integration Points**:

- BC Purchase Price Management: Retrieves vendor costs using Requisition Line temporary records
- Sales Order Processing: Event subscribers on Sales Line table validation events
- Purchase Order Processing: Event subscribers on Purchase Line and Requisition Line operations
- Item Master: Reads Main Vendor No. BMU and Main Wholesaler No. BMU
- Wholesale Sourcing: Integrated with Wholesale Account Group assignments

## 3. SETUP AND CONFIGURATION

### Primary Setup Page

**Load Factors** (Search: "Load Factors")

**Type Field** (required): Select load factor type which determines field requirements

- **Item**: Applies to specific item regardless of customer/vendor
- **Customer**: Applies to all items for specific customer
- **Customer & Item**: Applies to specific customer-item combination (highest specificity)
- **Vendor**: Applies to all items from specific vendor
- **Vendor & Item**: Applies to specific vendor-item combination
- **Item Category Code**: Applies to all items in category
- **Wholesale Group Account**: Applies to wholesale account group
- **Wholesale Group Account & Item**: Applies to specific wholesale group and item

**Dimension Fields** (conditional based on Type):

- **Item No.**: Required for Item, Customer & Item, Vendor & Item, Wholesale Group Account & Item types
- **Customer No.**: Required for Customer and Customer & Item types
- **Vendor No.**: Required for Vendor and Vendor & Item types
- **Item Category Code**: Required for Item Category Code type
- **Purchasing Code**: Optional for any type; provides additional specificity
- **Wholesale Account Group**: Required for Wholesale Group Account types

**Load Factor Pct.** (required): Percentage to add to base cost (e.g., 15 for 15% markup)

**Starting Date** (optional): First date load factor applies; blank = no start restriction

**Ending Date** (optional): Last date load factor applies; blank = no end restriction

**Active** (required): Must be checked to activate. Validation enforces that all required fields for selected Type are populated before activation

**Hierarchy No.** (read-only): Auto-calculated priority

- Customer & Item: 10 (highest priority - most specific)
- Customer: 30
- Vendor: 40
- Item: 50
- Vendor & Item: 60
- Item Category Code: 70
- Wholesale Group Account: 80
- Wholesale Group Account & Item: 90 (lowest priority)

### Setup Validation Rules

- Cannot modify Type, dimension fields, or Purchasing Code while Active = true
- System clears inapplicable dimension fields automatically when Type changes
- When activating, system validates required fields are populated based on Type
- Hierarchy No. recalculates automatically when Type changes

### Dependencies

**Required Setup**:

- **Item Master**: Items must have Main Vendor No. BMU populated for default cost retrieval
- **Purchase Price Lists**: Vendor costs must be maintained in BC price lists with valid dates and UOMs
- **Vendor Master**: Vendors must be set up with purchase price agreements

**Optional Setup**:

- **Purchasing Codes**: Define special purchasing categories if needed for load differentiation
- **Item Categories**: Set up if using Item Category Code load factor type
- **Wholesale Account Groups**: Configure if using Wholesale Group Account load factor types
- **Main Wholesaler No.**: Populate on items if using wholesaler-specific loads

### Best Practices for Setup

1. **Start Broad, Then Specialize**: Create general load factors first (Vendor, Customer), then add specific exceptions (Customer & Item)
2. **Use Date Ranges for Seasonal Loads**: Set Starting/Ending Dates for temporary load factor changes
3. **Monitor Hierarchy Conflicts**: Review Hierarchy No. to understand which load will apply when multiple matches exist
4. **Test Before Activation**: Set up load factor with Active = false, verify fields, then activate
5. **Document Purchasing Code Usage**: Maintain clear definitions for custom purchasing codes used in load factor matching

## 4. USER GUIDE

### Creating a General Vendor Load Factor

1. Search for "Load Factors" and open the page
2. Select **New** to create a new record
3. Set **Type** = Vendor
4. Enter **Vendor No.** (system clears other dimension fields automatically)
5. Enter **Load Factor Pct.** (e.g., 12.5 for 12.5% overhead)
6. Leave **Starting Date** and **Ending Date** blank for permanent load
7. Optionally enter **Purchasing Code** for additional specificity
8. Check **Active** = true (system validates setup)
9. System sets **Hierarchy No.** = 40
10. Click **OK** to save

**Result**: All items purchased from this vendor will have 12.5% load applied to cost on sales orders.

### Creating a Customer-Specific Item Load Factor

1. Open Load Factors page and select **New**
2. Set **Type** = Customer & Item
3. Enter **Customer No.**
4. Enter **Item No.**
5. Enter **Load Factor Pct.** (e.g., 20 for premium handling)
6. Set date range if temporary, or leave blank
7. Check **Active** = true
8. System sets **Hierarchy No.** = 10 (highest priority)

**Result**: When this customer orders this item, 20% load applies instead of any broader loads.

### Creating a Time-Limited Load Factor

1. Open Load Factors page and select **New**
2. Choose appropriate **Type** (e.g., Item Category Code for seasonal category)
3. Fill required fields based on type
4. Enter **Load Factor Pct.**
5. Set **Starting Date** = first day load applies (e.g., 12/01/2025 for winter season)
6. Set **Ending Date** = last day load applies (e.g., 02/28/2026)
7. Check **Active** = true

**Result**: Load applies only for sales orders dated within the specified range.

### Viewing Load Factor on Sales Orders

1. Open sales order and navigate to lines
2. Select a line with Type = Item
3. Look for **Load Factor Pct BMU** field in line details
4. Value shows the percentage applied (0 = no load factor matched)
5. **Unit Cost (LCY)** shows the final loaded cost
6. Compare to vendor cost to verify load application

**Calculation Check**: If vendor cost = $100 and Load Factor Pct = 15, Unit Cost (LCY) should equal $115.

### Overriding Load Factor Cost

1. On sales order line, manually edit **Unit Cost (LCY)** field
2. Enter desired cost
3. System automatically sets **Load Factor Pct BMU** = 0
4. Gross profit percentages recalculate based on manual cost
5. Cost remains fixed until line is modified or vendor changes

**Note**: Manual cost override prevents automatic load factor recalculation.

### Troubleshooting Load Factor Application

1. If unexpected load factor applies:
   - Open Load Factors page
   - Filter by **Active** = true
   - Set filters matching sales line conditions (Item, Customer, Vendor, etc.)
   - Sort by **Hierarchy No.** ascending
   - First record shown is the one that will match
2. If no load factor applies (Load Factor Pct BMU = 0):
   - Verify at least one active load factor exists for vendor or item
   - Check date range on load factors covers order date
   - Confirm dimension fields match or are blank on load factor records
   - Review purchasing code matching requirements

## 5. TROUBLESHOOTING GUIDE

### Problem: Load factor not applying (Load Factor Pct BMU shows 0)

**Diagnostic Steps**:

1. Check if any active load factors exist in system
2. Review sales line fields: Item No., Sell-to Customer No., Vendor (Main Vendor or Suggested), Item Category Code, Purchasing Code, Wholesale Account Group
3. Compare sales line Document Date to load factor Starting/Ending Date ranges
4. Verify item has Main Vendor No. BMU populated

**Common Causes**:

- No active load factors configured in system
- All load factors have date ranges that don't include order date
- Load factor dimension fields don't match sales line values (e.g., load factor has Customer No. but sales line customer doesn't match)
- Item missing Main Vendor No. BMU so vendor cost cannot be retrieved

**Solutions**:

1. Create appropriate load factor with matching dimensions and activate
2. Adjust date ranges on existing load factors to include current date, or create new load factor with correct dates
3. Review dimension matching: load factors must have blank OR matching values for all dimensions
4. Populate Main Vendor No. BMU on item master, then re-validate sales line

**Prevention**: Establish default load factors with broad matching (Type = Vendor or blank dimensions) to ensure minimum load applies

### Problem: Wrong load factor applying (unexpected percentage)

**Diagnostic Steps**:

1. Note the Load Factor Pct BMU value showing on sales line
2. Open Load Factors page and filter Active = true
3. Apply filters matching sales line: Item No., Customer No., Vendor No., Purchasing Code
4. Sort by Hierarchy No. ascending
5. First record shown is the matching load factor

**Common Causes**:

- Multiple load factors configured with different hierarchy levels and specific dimensions
- Customer & Item load factor (Hierarchy 10) overrides broader Customer load factor (Hierarchy 30)
- Purchasing Code mismatch causing fallback to less specific load factor
- Date-specific load factor active when general load factor expected

**Solutions**:

1. If wrong hierarchy: Adjust dimension specificity by changing Type or adding/removing dimension values
2. If purchasing code issue: Either add purchasing code to higher-priority load factor or remove from lower-priority load factor
3. If date issue: Adjust Starting/Ending Dates or inactivate temporary load factor
4. Create more specific load factor with correct percentage if customer-item-vendor combination needs unique treatment

**Prevention**: Document load factor hierarchy strategy and review new load factors against existing before activation

### Problem: Load factor changes not reflecting on existing sales orders

**Diagnostic Steps**:

1. Check when load factor was modified or created (compared to sales order date)
2. Review sales line Load Factor Pct BMU - shows percentage at time line was created
3. Test by creating new sales line for same item/customer - verify new load applies

**Common Causes**:

- Load factors apply at time of sales line creation/modification, not retroactively
- Existing sales lines retain original load factor percentage
- Unit cost is locked on posted documents (invoices, credit memos)

**Solutions**:

1. For unposted orders: Delete and re-add sales lines to pick up new load factor
2. Or manually adjust Unit Cost (LCY) on existing lines (sets Load Factor Pct = 0)
3. For future orders: New sales lines will automatically use updated load factor

**Prevention**: Plan load factor changes in advance; communicate to users when cost adjustments take effect

### Problem: Purchase order creation not updating sales line cost correctly

**Diagnostic Steps**:

1. Verify sales line has Load Factor Pct BMU populated before making purchase order
2. Check if purchase order line was created via Make Order from requisition worksheet
3. Review purchase line Direct Unit Cost vs sales line Unit Cost (LCY)
4. Verify load factor still active and within date range

**Common Causes**:

- Sales line created before load factor was activated
- Purchase order not created through proper requisition workflow (manual entry doesn't trigger update)
- Vendor changed between sales line creation and purchase order generation
- Load factor expired (Ending Date passed)

**Solutions**:

1. Run requisition worksheet using Get Sales Orders or Make Order to properly link PO to SO
2. Ensure Make Order batch job used to create purchase orders from requisition lines
3. Verify load factors active and date ranges cover both sales order date and purchase order date
4. Manually adjust sales line Unit Cost (LCY) if automatic update fails

**Prevention**: Use standard requisition worksheet workflow for all special order/drop shipment scenarios

### Problem: Gross profit percentage incorrect after load factor application

**Diagnostic Steps**:

1. Verify Unit Price, Unit Cost (LCY), and Load Factor Pct BMU on sales line
2. Calculate expected: Gross Profit % = ((Unit Price - Unit Cost) / Unit Price) × 100
3. Check Gross Profit Pct BMU field matches calculation
4. Verify vendor cost used as base (before load factor)

**Common Causes**:

- Unit Price not set or = 0 (causes division by zero, results in 0% gross profit)
- Load factor incorrectly entered (e.g., 15 entered as 1500 instead of 15)
- Vendor cost incorrect in purchase price list
- Manual Unit Cost override applied without updating price

**Solutions**:

1. If Unit Price = 0: Enter correct selling price; gross profit recalculates automatically
2. If load factor too high: Review load factor percentage (15% should be entered as 15, not 0.15 or 1500)
3. If vendor cost wrong: Update purchase price list, then re-validate sales line item
4. If manual cost: Verify intended cost and adjust Unit Price to achieve target margin

**Prevention**: Validate unit prices entered; establish minimum margin policies in Ceiling/Floor Setup; review load factor percentages before activation

## 6. MONITORING AND REPORTING

### Key Monitoring Pages

**Load Factors Page**:

- View all configured load factors
- Filter by Active = true to see currently applicable loads
- Sort by Hierarchy No. to understand priority order
- Check date ranges to identify expired or future-dated loads

**Sales Order Lines**:

- **Load Factor Pct BMU** field shows applied percentage
- **Gross Profit Pct BMU** shows margin with loaded cost
- **Gross Profit Dollars BMU** shows dollar margin with loaded cost
- **Unit Cost (LCY)** shows final loaded cost

**Sales Line Pricing Info Page** (via More Options on sales line):

- Shows detailed pricing breakdown
- Main Vendor Cost and Main Wholesaler Cost for comparison
- Load Factor Pct for visibility into cost calculation

### Health Check Procedures

**Weekly Load Factor Review**:

1. Open Load Factors page
2. Filter Active = true
3. Review records with Ending Date within next 30 days
4. Identify expiring loads and decide on renewal/replacement
5. Check for duplicate or conflicting load factors

**Monthly Margin Analysis**:

1. Run sales reports with Gross Profit Pct BMU and Gross Profit Dollars BMU
2. Compare actual margins to target margins
3. Identify items/customers with below-target margins
4. Review load factors for affected items to ensure appropriate overhead allocation
5. Adjust load percentages if cost structures have changed

**Quarterly Load Factor Audit**:

1. Review all load factors (active and inactive)
2. Inactivate obsolete load factors
3. Verify date ranges align with business seasons/contracts
4. Test hierarchy priority by creating sample sales lines
5. Document load factor strategy and update as needed

### Regular Maintenance Tasks

**Daily**:

- Monitor sales order entry for lines with 0% load factor
- Investigate unexpected Unit Cost (LCY) values on new sales lines

**Weekly**:

- Review load factors expiring in next 2 weeks
- Update or extend date ranges as needed
- Create new load factors for upcoming promotions or seasonal changes

**Monthly**:

- Analyze gross profit by customer, item, vendor
- Adjust load factors based on actual costs vs planned overhead
- Archive inactive load factors older than 1 year

**Quarterly**:

- Conduct comprehensive load factor effectiveness review
- Interview sales and operations teams on margin performance
- Adjust load factor strategy based on business changes
- Update documentation and training materials

## 7. BEST PRACTICES

### Setup Recommendations

- **Establish Hierarchy Strategy**: Document clear rules for which types of load factors to use at each specificity level (e.g., default vendor load, exception customer loads, special item loads)
- **Use Consistent Percentages**: Standardize load factors by vendor class or item category to simplify maintenance (e.g., all domestic vendors = 12%, all imported = 18%)
- **Date Temporary Loads**: Always set Ending Date for promotional or seasonal load factors to prevent them from applying indefinitely
- **Test in Sandbox**: Create and test new load factors in test environment before deploying to production
- **Name Purchasing Codes Descriptively**: Use clear names like "DROP-SHIP", "SPECIAL-ORDER", "FREIGHT-INCL" so load factor purpose is obvious

### Operational Guidelines

- **Review Before Activation**: Always verify required fields, percentage value, and date range before checking Active = true
- **One Load Factor Change at a Time**: Activate new load factors individually and test with sample sales lines before proceeding to next
- **Communicate Cost Changes**: Notify sales team when load factors change to help explain margin shifts to customers
- **Monitor Manual Overrides**: Track when users manually change Unit Cost (LCY) to identify items needing special load factors
- **Coordinate with Purchasing**: Align load factor updates with vendor cost changes to maintain accurate margins

### Performance Optimization Tips

- **Limit Active Records**: Inactivate expired load factors promptly to reduce search time during load factor resolution
- **Use Date Ranges Judiciously**: Fewer date-bound load factors = faster matching algorithm
- **Avoid Over-Specification**: Don't create customer-item load factors unless truly needed; broader load factors perform better
- **Index Considerations**: System uses Hierarchy No. as secondary sort key; keep active load factors under 1,000 records for optimal performance

### Common Pitfalls to Avoid

- **Percentage Entry Errors**: Entering 0.15 instead of 15 results in 0.15% load (nearly zero) instead of 15% load
- **Forgetting to Activate**: Creating load factor record without checking Active = true means it never applies
- **Overlapping Date Ranges**: Multiple load factors with different percentages but same dimensions and overlapping dates cause confusion
- **Missing Main Vendor**: If item lacks Main Vendor No. BMU, load factor cannot apply because base cost cannot be determined
- **Manual Cost Override Persistence**: Once Unit Cost manually changed, load factor won't reapply automatically even if vendor cost changes
- **Purchasing Code Mismatches**: Load factor with purchasing code won't match sales lines without that code; creates gaps in load application
- **Ignoring Hierarchy**: Creating multiple load factors without understanding hierarchy priority leads to unexpected results

## 8. FIELD REFERENCE

### LoadFactorsBMU Table (71552775)

| Field Name | Type | Description | Usage Notes |
|------------|------|-------------|-------------|
| Type | Enum (LoadFactorTypeBMU) | Defines which dimension fields are required | Determines field requirements and hierarchy number |
| Item No. | Code[20] | Specific item for load factor | Required for Item, Customer & Item, Vendor & Item, Wholesale Group Account & Item types |
| Customer No. | Code[20] | Specific customer for load factor | Required for Customer, Customer & Item types |
| Vendor No. | Code[20] | Specific vendor for load factor | Required for Vendor, Vendor & Item types |
| Item Category Code | Code[20] | Item category for load factor | Required for Item Category Code type |
| Purchasing Code | Code[10] | Purchasing type classification | Optional for any type; adds specificity |
| Wholesale Account Group | Code[20] | Wholesale account grouping | Required for Wholesale Group Account types |
| Hierarchy No. | Integer | Auto-calculated priority (lower = higher priority) | Read-only; ranges from 10 (Customer & Item) to 90 (Wholesale Group Account & Item) |
| Load Factor Pct. | Decimal | Percentage markup to apply to vendor cost | Required; enter as whole number (e.g., 15 for 15%, not 0.15) |
| Starting Date | Date | First date load factor is valid | Optional; blank = no start restriction |
| Ending Date | Date | Last date load factor is valid | Optional; blank = no end restriction |
| Active | Boolean | Whether load factor is currently in use | Must be true for load factor to apply; validation enforces field requirements |

### LoadFactorTypeBMU Enum (71552579)

| Value | Caption | Hierarchy No. | Required Fields |
|-------|---------|---------------|-----------------|
| Item | Item | 50 | Item No. |
| Customer | Customer | 30 | Customer No. |
| CustomerItem | Customer & Item | 10 | Item No., Customer No. |
| Vendor | Vendor | 40 | Vendor No. |
| VendorItem | Vendor & Item | 60 | Item No., Vendor No. |
| Item Category Code | Item Category Code | 70 | Item Category Code |
| Wholesale Group Account | Wholesale Group Account | 80 | Wholesale Account Group |
| Wholesale Group AccountItem | Wholesale Group Account & Item | 90 | Wholesale Account Group, Item No. |

### Sales Line Extensions (71552593)

| Field Name | Type | Description | Usage Notes |
|------------|------|-------------|-------------|
| LoadFactorPctBMU | Decimal | Load factor percentage applied when line created | Shows actual percentage used; 0 = no load factor or manual override |
| GrossProfitPctBMU | Decimal | Calculated gross profit percentage | ((Unit Price - Unit Cost) / Unit Price) × 100; uses loaded cost |
| GrossProfitDollarsBMU | Decimal | Calculated gross profit dollars | Unit Price - Unit Cost; dollar margin after load factor |

## 9. ADVANCED TOPICS

### Hierarchy Priority Logic

When multiple load factors could match a sales line, the system uses **Hierarchy No.** to determine priority:

**Example Scenario**: Item ABC ordered by Customer 123 from Vendor 999

**Active Load Factors**:
- Type = Customer & Item, Customer 123, Item ABC, 20% (Hierarchy 10)
- Type = Customer, Customer 123, 15% (Hierarchy 30)
- Type = Vendor, Vendor 999, 12% (Hierarchy 40)
- Type = Item, Item ABC, 10% (Hierarchy 50)

**Result**: 20% load applies because Customer & Item (Hierarchy 10) has highest priority

### Purchasing Code Special Logic

When load factor matches with blank Purchasing Code but sales line has Purchasing Code populated:
1. System finds first match (e.g., Vendor load with blank Purchasing Code)
2. Before returning percentage, searches again for exact match including Purchasing Code
3. If exact match found, uses that percentage instead
4. If no exact match, returns original match percentage

**Example**:
- Load Factor A: Vendor 999, Purchasing Code = blank, 12%
- Load Factor B: Vendor 999, Purchasing Code = "DROP-SHIP", 15%
- Sales Line: Vendor 999, Purchasing Code = "DROP-SHIP"
- **Result**: 15% applies (Load Factor B) because exact purchasing code match takes precedence

### Integration with Wholesale Sourcing

Load factors integrate with Wholesale Account Group field on sales lines:
- Wholesale Account Group Code BMU field added to sales line
- Load factors can match on Wholesale Account Group dimension
- Enables different load percentages for different wholesaler channels
- Two types available: Wholesale Group Account (group-level) and Wholesale Group Account & Item (group-item combination)

### Event Subscribers and Extensibility

**OnAfterSetLoadFactorFilterss Event**: Allows customization of load factor filtering logic
- Fired after standard filters applied but before FindFirst
- Can add additional filters or modify existing filters
- Use case: Add custom dimension fields for load factor matching

**OnAfterSetLoadFactor2Filterss Event**: Allows customization of purchasing code fallback logic
- Fired when searching for exact purchasing code match
- Can modify search criteria for special purchasing code handling
- Use case: Implement custom purchasing code hierarchies

**OnBeforeLoadActivate Event**: Fires before activation validation
- Can add custom validation rules
- Can prevent activation based on custom business rules

**OnAfterLoadActivate Event**: Fires after successful activation
- Can perform post-activation processing
- Use case: Log load factor changes, send notifications

### Cost Source Determination

Load Factor Management determines which vendor cost to load through this logic:
1. If sales line has SuggestedVendorNoBMU populated, uses that vendor's cost
2. Otherwise, uses item's MainVendorNoBMU
3. Retrieves cost via temporary Requisition Line (mimics BC purchase price lookup)
4. Uses Order Date, Unit of Measure Code, and Quantity to find best price
5. Returns Direct Unit Cost which becomes base for load factor application

### Uncatalogued Item Handling

For uncatalogued items (items not in vendor catalogs):
- UnCataloguedItemBMU flag prevents vendor cost lookup
- Uses manually entered Unit Cost (LCY) on sales line as base cost
- Load factor still applies to manual cost if applicable load factor exists
- Allows load factors to work with custom-priced items

### Gross Profit Real-Time Calculation

System maintains real-time gross profit metrics through event subscribers:
- **Unit Price Change**: Recalculates both percentage and dollar profit
- **Unit Cost Change**: Recalculates both percentage and dollar profit
- **Manual Cost Override**: Clears load factor percentage, recalculates profit
- **Purchasing Code Change**: Re-evaluates load factor, updates cost and profit
- **Suggested Vendor Change**: Re-retrieves cost, reapplies load factor, updates profit

## 10. APPENDIX

### Common Scenarios and Solutions

**Scenario 1: Standard Wholesale Operations**

*Requirement*: Apply 15% overhead to all items from primary wholesaler, 18% for secondary wholesaler

*Solution*:
1. Create load factor: Type = Vendor, Vendor No. = PRIMARY-WHOLE, Load Factor Pct = 15
2. Create load factor: Type = Vendor, Vendor No. = SECONDARY-WHOLE, Load Factor Pct = 18
3. Activate both load factors
4. Set items' Main Vendor No. BMU to appropriate wholesaler

**Scenario 2: High-Value Customer Reduction**

*Requirement*: Reduce overhead to 10% for specific customer to remain competitive

*Solution*:
1. Create load factor: Type = Customer, Customer No. = BIGCUST001, Load Factor Pct = 10
2. Activate load factor
3. Customer load (Hierarchy 30) overrides vendor load (Hierarchy 40)

**Scenario 3: Seasonal Freight Increase**

*Requirement*: Add 5% winter freight surcharge November through February

*Solution*:
1. Create load factor: Type = Item Category Code, Item Category Code = BULKY, Load Factor Pct = 23
2. Set Starting Date = 11/01/2025
3. Set Ending Date = 02/28/2026
4. Activate load factor
5. When March arrives, load factor automatically stops applying

**Scenario 4: Drop Ship Direct Vendor**

*Requirement*: Different load for drop shipments (no warehouse handling)

*Solution*:
1. Create purchasing code "DROP-SHIP"
2. Create load factor: Type = Vendor, Vendor No. = VENDOR001, Purchasing Code = DROP-SHIP, Load Factor Pct = 8
3. Create default load factor: Type = Vendor, Vendor No. = VENDOR001, Purchasing Code = blank, Load Factor Pct = 15
4. When sales line has Purchasing Code = DROP-SHIP, 8% applies
5. All other orders from VENDOR001 get 15%

**Scenario 5: Special Item Handling**

*Requirement*: One item requires refrigerated freight, needs 25% load

*Solution*:
1. Create load factor: Type = Item, Item No. = SPECIALITEM, Load Factor Pct = 25
2. Activate load factor
3. Item load (Hierarchy 50) overrides customer/vendor loads for this specific item

### Quick Reference Guide

**Load Factor Type Selection**:
- Need item-specific load → Item
- Need customer-specific load → Customer
- Need both → Customer & Item
- Need vendor-specific load → Vendor
- Need vendor-item combination → Vendor & Item
- Need category-wide load → Item Category Code
- Need wholesale channel load → Wholesale Group Account

**Troubleshooting Checklist**:
- [ ] Load factor Active = true?
- [ ] Date range includes order date?
- [ ] Required fields populated for Type?
- [ ] Item has Main Vendor No.?
- [ ] Vendor has purchase price list?
- [ ] Purchasing code matches if specified?

**Priority Order (Highest to Lowest)**:
1. Customer & Item (10)
2. Customer (30)
3. Vendor (40)
4. Item (50)
5. Vendor & Item (60)
6. Item Category Code (70)
7. Wholesale Group Account (80)
8. Wholesale Group Account & Item (90)

### Glossary of Terms

**Load Factor**: Percentage markup applied to vendor cost to arrive at loaded cost including overhead

**Loaded Cost**: Vendor cost plus load factor percentage; final Unit Cost (LCY) on sales line

**Hierarchy Number**: System-calculated priority ranking; lower numbers have higher priority in matching

**Base Cost**: Vendor purchase cost before load factor applied

**Gross Profit Pct**: ((Selling Price - Loaded Cost) / Selling Price) × 100

**Main Vendor No.**: Primary vendor for item, used as default for cost retrieval

**Suggested Vendor No.**: Sales line override for vendor; takes precedence over Main Vendor

**Purchasing Code**: Classification of purchase type (e.g., drop ship, special order, stock)

**Wholesale Account Group**: Grouping of wholesale channels for load factor differentiation

**Active**: Load factor status; only active load factors considered in matching algorithm

**Starting Date / Ending Date**: Date range for load factor validity; blank = no restriction

**Manual Cost Override**: User-entered Unit Cost (LCY); clears load factor percentage

### Related Documentation

- **Cost-Based Pricing**: Works with load factors to calculate selling prices from loaded costs
- **Extended Sales Pricing**: Uses loaded costs in margin calculations and pricing rules
- **Wholesale Sourcing**: Provides Wholesale Account Group dimension for load factor matching
- **Purchase Price Management**: Standard BC feature providing vendor cost data
- **Requisition Worksheet**: Standard BC feature for creating purchase orders from sales orders

### Support Contact Information

For questions about Load Factor setup or troubleshooting, contact:
- **Application Support**: [Support contact details]
- **System Administrator**: [Admin contact details]
- **Documentation**: See BMI SupplyAutomate documentation library

---

## Document Information

- Feature: Load Factor Management
- Created: December 23, 2025
- Version: 1.0
- Status: Complete
- Revision History:
  - v1.0 (12/23/2025): Initial comprehensive documentation

