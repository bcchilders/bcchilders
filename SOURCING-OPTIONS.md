# Sourcing Options

**Document Version**: 1.0  
**Last Updated**: December 23, 2025  
**Feature Status**: Production

---

## 1. OVERVIEW

**Sourcing Options** provide rule-based overrides for sales line sourcing behavior, enabling location, customer, or item-specific configurations that supersede default settings. This feature allows businesses to implement complex sourcing strategies such as carton-based ordering, deviated vendor costing, customer-specific fulfillment methods, and wholesale account routing based on product attributes or categories.

The system evaluates Sourcing Options rules during sales line entry and automatically applies matching configurations including: forced purchasing codes (drop ship, special order, wholesale), suggested vendors, custom unit costs, wholesale account group assignments, and quantity-based splitting. Rules are prioritized hierarchically with item-level matches taking precedence over category and attribute matches, enabling precise control over fulfillment behavior without modifying item master data.

Sourcing Options integrate seamlessly with Wholesale Account Groups, Load Factor Management, and Order Fulfillment systems. When a sourcing option matches, it can override the customer or ship-to address wholesale account group assignment, inject custom costs for deviated pricing scenarios, and automatically split sales lines to match carton quantities—all transparently to the user during order entry.

---

## 2. HOW IT WORKS

### Automated Processing

**Sales Line Validation Trigger**:
1. User enters or modifies sales line: Item No., Quantity, Unit of Measure, or Location Code
2. `SourcingOptionEventMgtBMU` event subscribers intercept validation events
3. System invokes `SourcingOptionMgtBMU.SetSalesLineSourcingOptions()`
4. Sourcing options are evaluated in priority order (lowest priority number first)
5. System searches for active sourcing options matching:
   - Location Code (required match)
   - Customer No. (blank = all customers)
   - Ship-to Code (blank = all ship-to addresses)
   - Type + Type No. (Item, Item Category, or Item Attribute)
   - Unit of Measure Code (blank = all UOMs)
   - Minimum Quantity (sales line quantity must meet threshold)
6. First matching rule found wins (stops evaluation)
7. System applies matched rule to sales line:
   - Sets `WholesaleAccountGroupCodeBMU` (overrides customer/ship-to default)
   - Validates and sets `Purchasing Code` (if Forced Purchasing Code populated)
   - Sets `SuggestedVendorNoBMU` (if Suggested Vendor No. populated)
   - Sets `Unit Cost (LCY)` with load factor applied (if Unit Cost populated)
   - Splits sales line if Quantity Multiple configured and quantity not evenly divisible
   - Marks `SourcingOptionFoundBMU` = true
8. If no match found and line previously had sourcing option, resets line to item defaults

**Carton Sourcing / Line Splitting**:
- When sourcing option has Quantity Multiple > 0 and sales line quantity is not evenly divisible
- System automatically splits line: one line with multiple quantity, one line with remainder
- User receives notification: "The sales line has been split due to a Sourcing Option Setup for Carton Sourcing"
- Enables carton-based ordering where vendors require specific packaging quantities

### Manual Operations

**No direct user interaction during sales order entry**—sourcing options apply automatically.

**Configuration tasks**:
- Create/modify sourcing option rules on Sourcing Options page
- Activate rules to enable them
- Deactivate rules to suspend without deletion
- Adjust Priority to control rule precedence

### Data Flow

**Key Tables**:
- **SourcingOptionBMU** (71552627): Stores sourcing rules with matching criteria and actions
- **Sales Line** (extensions): `SourcingOptionFoundBMU`, `WholesaleAccountGroupCodeBMU`, `SuggestedVendorNoBMU` fields
- **Item, Item Category, Item Attribute**: Referenced for Type matching
- **Wholesale Account Group, Vendor, Purchasing, Location**: Master data for rule configuration

**Integration Points**:
- **Sales Line Validation**: Event subscribers on No., Quantity, Location Code, Unit of Measure Code
- **Wholesale Account Groups**: Sourcing options override default group assignments
- **Load Factor Management**: Applied to Unit Cost when sourcing option provides cost
- **Bumpdown/Order Fulfillment**: Wholesale Account Group override affects vendor selection
- **Item Vendor**: Validates vendor relationships when suggested vendor assigned

---

## 3. SETUP

### Primary Setup Page

**Search**: "Sourcing Options"

**Required Fields**:
- **Location Code**: Warehouse location where rule applies (required, not blank)
- **Type**: Item, Item Category, or Item Attribute (determines matching logic)
- **Active**: Must be checked to enable rule

**Configuration Fields** (mark actions to apply):
- **Customer No.** (optional): Specific customer, blank = all customers
- **Ship-to Code** (optional): Specific ship-to address for customer, blank = all addresses
- **Type No.**: Item No., Item Category Code, or blank for "any" within type
- **Item Attribute Name / Value**: For Type = Item Attribute, select attribute and value
- **Unit of Measure Code** (optional): Specific UOM, blank = all UOMs
- **Minimum Quantity** (optional): Sales line quantity must meet or exceed, 0 = any quantity
- **Quantity Multiple** (optional): If > 0, splits lines not evenly divisible by this value (carton sourcing)
- **Forced Purchasing Code** (optional): Overrides purchasing code (drop ship, special order, wholesale)
- **Suggested Vendor No.** (optional): Sets suggested vendor on sales line
- **Wholesale Account Group Code** (optional): Overrides default wholesale account group
- **Unit Cost** (optional): Deviated cost for special pricing scenarios (requires Type = Item, UOM, Forced Purchasing Code, and Suggested Vendor or Wholesale Account Group)

**Priority Field**:
- Auto-calculated when activated based on Type and customer/ship-to specificity
- Item-level: 10-35, Attribute-level: 40-65, Category-level: 70-95
- Ship-to specific < Customer specific < Global (all customers)
- Can be manually overridden before activation

### Validation Rules

**Activation Requirements**:
- Wholesale Account Group Code must be blank if Forced Purchasing Code is non-wholesale (stock, special order)
- If Quantity Multiple > 1, Unit of Measure Code must be specified
- If Unit Cost <> 0:
  - Type must be Item (not category or attribute)
  - Unit of Measure Code required
  - Forced Purchasing Code required
  - Either Suggested Vendor No. or Wholesale Account Group Code required (at least one)

**Field Locking**:
- All configuration fields are locked when Active = true
- Must deactivate rule to modify any field except Priority

### Dependencies

**Master Data Prerequisites**:
- Locations must exist
- Customers and Ship-to Addresses must exist if specified
- Items, Item Categories, Item Attributes must exist for Type matching
- Wholesale Account Groups must be configured if referenced
- Vendors must exist if Suggested Vendor No. specified
- Purchasing Codes must exist if Forced Purchasing Code specified

**Feature Dependencies**:
- **Wholesale Account Groups**: If sourcing option assigns wholesale account group
- **Load Factor Management**: If Unit Cost specified, load factors are applied
- **Item Vendors**: System validates item-vendor relationships for suggested vendors

---

## 4. USER GUIDE

### Task 1: Create Item-Specific Sourcing Option for Drop Ship

1. Search "Sourcing Options" and open page
2. Create new record
3. Set **Location Code** = your warehouse (e.g., "MAIN")
4. Set **Customer No.** = blank (applies to all customers)
5. Set **Type** = "Item"
6. Set **Type No.** = specific item number (e.g., "ITEM-001")
7. Set **Forced Purchasing Code** = your drop ship purchasing code (e.g., "DROPSHIP")
8. Set **Suggested Vendor No.** = preferred vendor for this item (e.g., "VENDOR-100")
9. Check **Active** = true (system auto-assigns Priority = 30)
10. Close page—rule is now active

**Result**: When this item is entered on sales order at MAIN location, purchasing code and suggested vendor are automatically set.

### Task 2: Create Carton Sourcing Rule for Item Category

1. Search "Sourcing Options" and open page
2. Create new record
3. Set **Location Code** = your warehouse (e.g., "MAIN")
4. Set **Type** = "Item Category"
5. Set **Type No.** = category code (e.g., "PAPER")
6. Set **Unit of Measure Code** = "CARTON"
7. Set **Quantity Multiple** = 10 (items must be ordered in multiples of 10 cartons)
8. Set **Forced Purchasing Code** = wholesale purchasing code
9. Set **Wholesale Account Group Code** = appropriate group (e.g., "WHOLESALE-PAPER")
10. Check **Active** = true (system auto-assigns Priority = 90)

**Result**: Paper category items ordered in cartons are automatically split if quantity not multiple of 10, and wholesale account group is assigned.

### Task 3: Create Deviated Cost Rule for Special Customer

1. Search "Sourcing Options" and open page
2. Create new record
3. Set **Location Code** = your warehouse (e.g., "MAIN")
4. Set **Customer No.** = special customer (e.g., "CUST-500")
5. Set **Type** = "Item"
6. Set **Type No.** = specific item (e.g., "ITEM-999")
7. Set **Unit of Measure Code** = "EACH"
8. Set **Forced Purchasing Code** = special order code
9. Set **Suggested Vendor No.** = contracted vendor
10. Set **Unit Cost** = negotiated cost (e.g., 15.00)
11. Check **Active** = true (system auto-assigns Priority = 20)

**Result**: When this customer orders this item, cost is set to negotiated price with load factor applied automatically.

### Task 4: Override Wholesale Account Group for Specific Location

1. Search "Sourcing Options" and open page
2. Create new record
3. Set **Location Code** = specific location (e.g., "EASTCOAST")
4. Set **Type** = "Item Category"
5. Set **Type No.** = blank (applies to all items in any category)
6. Set **Wholesale Account Group Code** = location-specific group (e.g., "EAST-ACCOUNTS")
7. Check **Active** = true (system auto-assigns Priority = 95)

**Result**: All items at EASTCOAST location use EAST-ACCOUNTS wholesale group regardless of customer default.

### Task 5: Temporarily Disable a Sourcing Option

1. Search "Sourcing Options" and open page
2. Find the rule to suspend
3. Uncheck **Active** checkbox
4. Close page

**Result**: Rule is preserved but no longer applied to sales lines. Can be reactivated anytime without re-entering configuration.

---

## 5. TROUBLESHOOTING

### Problem: Sourcing option not applying to sales line

**Causes**: Rule not active, Location Code mismatch, Customer/Ship-to filter too restrictive, Quantity below Minimum Quantity, UOM mismatch, Type matching failure, higher priority rule matched first

**Solution**: Open Sourcing Options page. Verify Active = true. Check Location Code matches sales line location exactly. If Customer No. or Ship-to Code populated, verify sales line matches. If Minimum Quantity > 0, ensure sales line quantity meets threshold. If Unit of Measure Code specified, confirm sales line UOM matches. Review Priority—lower priority rules execute first. Check if another rule matched before this one (look for SourcingOptionFoundBMU = true on sales line).

### Problem: Cannot activate sourcing option—validation error

**Causes**: Missing required fields for Unit Cost configuration, Wholesale Account Group with non-wholesale Purchasing Code, Quantity Multiple without Unit of Measure Code, Unit Cost with wrong Type

**Solution**: If using Unit Cost: verify Type = Item, Unit of Measure Code populated, Forced Purchasing Code populated, and either Suggested Vendor No. or Wholesale Account Group Code populated. If Wholesale Account Group Code populated, ensure Forced Purchasing Code is wholesale type (drop ship or wrap/label). If Quantity Multiple > 1, populate Unit of Measure Code. If Unit Cost specified with Type = Item Category or Item Attribute, change Type to Item or clear Unit Cost.

### Problem: Sales line splits unexpectedly

**Cause**: Sourcing option with Quantity Multiple configured, sales line quantity not evenly divisible by multiple

**Solution**: This is expected behavior for carton sourcing rules. User receives notification "The sales line has been split due to a Sourcing Option Setup for Carton Sourcing." One line has quantity rounded down to multiple, remainder on second line. To prevent splitting, either order in exact multiples of Quantity Multiple, or deactivate the sourcing option if splitting not desired.

### Problem: Wrong cost applied to sales line

**Causes**: Sourcing option Unit Cost not accounting for load factor, Load Factor not configured correctly, Unit Cost outdated

**Solutions**: Sourcing option Unit Cost is BASE cost before load factor. System automatically applies load factor percentage based on item, customer, vendor, and wholesale account group. If cost seems incorrect, verify: 1) Load Factor setup for this item/customer/vendor/account group combination is correct (Search "Load Factors"). 2) Unit Cost field in sourcing option reflects current vendor base cost (update if vendor pricing changed). 3) Load Factor percentage still accurate for overhead allocation. System calculates: Unit Cost (LCY) = Unit Cost × (1 + LoadFactorPct / 100).

### Problem: Sourcing option with high priority not executing

**Cause**: Lower priority number = higher priority (priority 10 executes before priority 50); misunderstanding of priority order

**Solution**: Review Priority field values. System evaluates sourcing options sorted by Priority ascending (1, 2, 3... 10, 20, 30...). Lower numbers execute first and stop evaluation when match found. Item-level rules default to 10-35, Attribute-level 40-65, Category-level 70-95. If item-level rule priority 30 not executing, check for another rule with priority 1-29 matching first. To force rule execution first, set Priority = 1.

---

## 6. MONITORING

### Key Monitoring Pages

**Sourcing Options**:
- Search: "Sourcing Options"
- Review Active checkbox—only active rules apply
- Sort by Priority to understand execution order
- Filter by Location Code to see location-specific rules
- Filter by Customer No. to review customer-specific configurations

**Sales Order / Sales Line**:
- `SourcingOptionFoundBMU` field indicates if rule matched
- `WholesaleAccountGroupCodeBMU` field shows applied group (may be from sourcing option override)
- `SuggestedVendorNoBMU` field shows suggested vendor (may be from sourcing option)
- `Purchasing Code` field shows forced code if applied
- `Unit Cost (LCY)` field shows cost with load factor if sourcing option provided cost

### Important Reports

**No dedicated sourcing options reports**—use standard Sales Order reports and filter by:
- Purchasing Code to identify orders affected by forced purchasing codes
- Suggested Vendor to identify orders using sourcing option vendor assignments
- Review split lines (consecutive line numbers, same item) to identify carton sourcing splits

### Health Checks

**Weekly**:
- Review Active sourcing options for outdated rules (old items, discontinued vendors)
- Verify Unit Cost fields reflect current vendor pricing
- Check for rules with no recent usage—consider archiving

**Monthly**:
- Audit Priority assignments for logical hierarchy
- Review customer-specific rules for customers no longer active
- Validate Wholesale Account Group assignments still align with business strategy

**Quarterly**:
- Test sample orders to confirm sourcing options apply correctly
- Review split line frequency—excessive splits may indicate carton size mismatches
- Consolidate redundant rules (multiple rules achieving same outcome)

---

## 7. BEST PRACTICES

**Rule Design**:
- Use descriptive combinations of Location + Customer + Item to make rules self-documenting
- Minimize number of active rules—prefer master data configuration over sourcing options where possible
- Apply sourcing options for exceptions and overrides, not standard behavior
- Blank Customer No. and Ship-to Code makes rules apply globally at location—use sparingly to avoid unintended consequences

**Priority Management**:
- Accept default priority auto-calculation during activation—follows proven hierarchy
- Only override priority when business requires specific exception to standard precedence
- Document non-standard priority assignments in comments or external documentation
- Lower priority number = higher precedence: Item (10-35) > Attribute (40-65) > Category (70-95)

**Cost Management (Unit Cost field)**:
- Keep Unit Cost current—outdated costs create incorrect margins
- Remember Unit Cost is BASE cost—system applies load factor automatically
- Use Unit Cost for deviated pricing scenarios only (special contracts, promotional pricing)
- Validate load factor configuration when using Unit Cost to ensure total cost accuracy

**Carton Sourcing (Quantity Multiple)**:
- Only use Quantity Multiple when vendor requires exact packaging quantities
- Set Quantity Multiple to actual carton/case pack size from vendor
- Educate users that line splits are expected behavior for carton sourcing
- Consider item UOM configuration instead of sourcing options for standard carton items

**Maintenance**:
- Deactivate rules instead of deleting them—preserves configuration for future reactivation
- Regularly audit active rules for business relevance
- Remove rules for discontinued items, closed customers, or obsolete vendors
- Test sourcing option changes in test environment before activating in production

**Critical Pitfalls to Avoid**:
- **Overlapping rules with same priority**: System uses first match found, which may be unpredictable—ensure priority differentiation
- **Global rules (blank customer) with aggressive configurations**: Can affect all orders at location unintentionally—prefer customer-specific rules
- **Mixing Suggested Vendor and Wholesale Account Group**: Both can be specified but may create sourcing conflicts—use one or the other per rule
- **Outdated Unit Costs**: Stale cost data leads to incorrect pricing and margin analysis—review quarterly minimum
- **Forgetting Unit of Measure dependencies**: Quantity Multiple and Unit Cost require Unit of Measure Code—activation will fail if missing

---

## 8. FIELD REFERENCE

### SourcingOptionBMU Table (71552627)

| Field | Type | Description |
|-------|------|-------------|
| Location Code | Code[10] | Required warehouse location where rule applies (PK component) |
| Customer No. | Code[20] | Optional specific customer; blank = all customers (PK component) |
| Ship-to Code | Code[10] | Optional specific ship-to address for customer; blank = all ship-to addresses (PK component) |
| Type | Enum | Matching type: Item, Item Category, Item Attribute (PK component) |
| Type No. | Code[20] | Item No., Item Category Code, or blank; blank = all within type (PK component) |
| Type ID | Integer | Item Attribute ID for Type = Item Attribute (PK component) |
| Type Value ID | Integer | Item Attribute Value ID for Type = Item Attribute (PK component) |
| Unit Of Measure Code | Code[10] | Optional UOM filter; blank = all UOMs (PK component) |
| Description | Text[250] | Auto-populated from Item, Item Category, or Item Attribute Value |
| Minimum Quantity | Decimal | Sales line quantity must meet or exceed this threshold; 0 = any quantity |
| Quantity Multiple | Decimal | If > 0, splits lines not evenly divisible (carton sourcing) |
| Forced Purchasing Code | Code[10] | Purchasing code to apply to sales line (overrides default) |
| Suggested Vendor No. | Code[20] | Vendor to suggest on sales line |
| Wholesale Account Group Code | Code[10] | Overrides customer/ship-to default wholesale account group |
| Unit Cost | Decimal | Base cost before load factor; applied to Unit Cost (LCY) with load factor |
| Active | Boolean | Rule must be active to apply; locks other fields when true |
| Priority | Integer | Execution order; lower number = higher priority; auto-calculated on activation |
| Item Attribute Name | Text[250] | Display name of Item Attribute for Type = Item Attribute |
| Item Attribute Value | Text[250] | Display value of Item Attribute Value for Type = Item Attribute |

**Primary Key**: Location Code, Customer No., Ship-to Code, Type, Type No., Type ID, Type Value ID, Unit Of Measure Code

### Sales Line Table Extensions

| Field | Type | Description |
|-------|------|-------------|
| SourcingOptionFoundBMU | Boolean | Indicates sourcing option matched and applied to this line |
| WholesaleAccountGroupCodeBMU | Code[10] | May be assigned by sourcing option (overrides customer/ship-to default) |
| SuggestedVendorNoBMU | Code[20] | May be assigned by sourcing option |

### SourcingOptionTypeBMU Enum (71552602)

| Value | Caption | Description |
|-------|---------|-------------|
| 0 | Item | Matches specific item number or all items (Type No. blank) |
| 1 | Item Attribute | Matches items with specific attribute value |
| 2 | Item Category | Matches items in category or all categories (Type No. blank) |

---

## 9. ADVANCED TOPICS

### Priority Auto-Calculation Algorithm

When **Active** is checked, system calculates Priority based on Type and customer/ship-to specificity:

**Item-level (highest specificity)**:
- Ship-to specific: Priority = 10
- Customer specific: Priority = 20
- Global (all customers): Priority = 30

**Item Attribute-level (medium specificity)**:
- Ship-to specific: Priority = 40
- Customer specific: Priority = 50
- Global (all customers): Priority = 60

**Item Category-level (lowest specificity)**:
- Ship-to specific: Priority = 70
- Customer specific: Priority = 80
- Global (all customers): Priority = 90

**If Type No. is blank** (matches all within type): Priority += 5

**User confirmation**: If Priority changes from previous value, system prompts: "Do you want to update the Priority from [old] to [new] to maintain alignment with default priorities?" User can decline to preserve custom priority.

### Matching Hierarchy and Evaluation Flow

**Search Order** (by Priority ascending):
1. System filters active sourcing options by Location Code (exact match required)
2. Filters by Customer No.: matching value or blank
3. Filters by Ship-to Code: matching value or blank
4. Within filtered set, searches by Type in this order:
   - **Item**: Matches Type No. = Item No. or Type No. = blank, filters by UOM and Minimum Quantity
   - **Item Attribute**: Matches items with Item Attribute Value assignment, filters by UOM and Minimum Quantity
   - **Item Category**: Matches items in Item Category, filters by UOM and Minimum Quantity
5. **First match wins**—evaluation stops immediately upon match
6. No match and SourcingOptionFoundBMU was previously true: resets line to item defaults

**Key Insight**: Lower priority item-specific rules execute before higher priority category-wide rules, enabling precise exception handling.

### Carton Sourcing / Line Splitting Logic

Implemented in `SourcingOptionMgtBMU.SplitSalesLine()`:

**Trigger Condition**: `(Quantity mod OrderMultiple) <> 0`

**Split Calculation**:
- `OrigLineQuantity = Quantity - (Quantity mod OrderMultiple)` (rounds down to multiple)
- `NewLineQuantity = Quantity mod OrderMultiple` (remainder)

**Example**: Quantity = 27, Quantity Multiple = 10
- Original line: Quantity = 20 (2 cartons)
- New line: Quantity = 7 (loose items)

**Split Action**: Calls `OrderFulfillmentMgtBMU.SplitSalesLine()` which creates duplicate sales line with NewLineQuantity, adjusts original line to OrigLineQuantity, preserves all other field values.

**User Notification**: "The sales line has been split due to a Sourcing Option Setup for Carton Sourcing."

### Unit Cost with Load Factor Integration

When sourcing option provides Unit Cost:

1. System retrieves Load Factor percentage: `LoadFactorPct = LoadFactorMgtBMU.GetLoadFactorPct(ItemNo, CustomerNo, VendorNo, ItemCategoryCode, PurchasingCode, DocumentDate, WholesaleAccountGroupCode)`
2. Calculates loaded cost: `Unit Cost (LCY) = UnitCost × (1 + (LoadFactorPct / 100))`
3. Updates gross profit fields: `GrossProfitPctBMU`, `GrossProfitDollarsBMU`
4. Marks `DoNotUpdatePurchPriceBMU = true` to prevent standard BC cost update logic from overwriting

**Load Factor Lookup**: Based on item, customer, vendor (from suggested vendor or main vendor), purchasing code, and wholesale account group. See [Load Factor Management](LOAD_FACTOR_MANAGEMENT.md) for hierarchy details.

### Item Attribute Matching

For Type = Item Attribute:
- System queries `Item Attribute Value Mapping` table linking items to attribute values
- Matches items where `Table ID = Database::Item` and `Item Attribute Value ID = sourcing option Type Value ID`
- Enables pattern-based sourcing without maintaining item-specific rules
- Example: All items with Attribute "Brand" = "HP" use specific wholesale account group

### Event Subscriber Integration

`SourcingOptionEventMgtBMU` subscribes to multiple sales line events:
- `OnValidateNoOnAfterUpdateUnitPrice`: Applies sourcing options after item number changes
- `OnAfterValidateEvent 'Quantity'`: Reapplies if quantity changes (may change Minimum Quantity matching)
- `OnAfterValidateLocationCode`: Reapplies if location changes (changes Location Code filter)
- `OnValidateUnitOfMeasureCodeOnAfterGetItemData`: Reapplies if UOM changes (changes UOM filter)
- `OnBeforeInsertSalesOrderLine` (Sales-Quote to Order): Applies sourcing options when converting quote to order

### Extensibility Points

**Extend SourcingOptionTypeBMU enum**: Add new matching types (e.g., "Customer Price Group", "Item Discount Group")

**Subscribe to events**: Inject custom logic before/after sourcing option application (if events published by SourcingOptionMgtBMU)

**Custom Priority Logic**: Modify `SetPriority()` procedure in SourcingOptionBMU table to implement alternative priority schemes

**Additional Actions**: Extend SourcingOptionBMU table with new fields and enhance `SetSalesLineSourcingOptions()` to apply additional sales line modifications

---

## 10. APPENDIX

### Quick Reference: Decision Tree

```
Sales Line Entry/Modification
    ↓
Location Code + Type + Filters Match?
    ↓ Yes (first match by Priority)
Apply Sourcing Option Actions:
  - Set Wholesale Account Group Code (if specified)
  - Set Purchasing Code (if Forced Purchasing Code specified)
  - Set Suggested Vendor (if Suggested Vendor No. specified)
  - Set Unit Cost with Load Factor (if Unit Cost specified)
  - Split Line (if Quantity Multiple specified and not evenly divisible)
  - Mark SourcingOptionFoundBMU = true
    ↓
    ↓ No Match
Previously had Sourcing Option? (SourcingOptionFoundBMU = true)
    ↓ Yes
Reset to Item Defaults (re-validate "No.")
    ↓ No
Continue with Standard Logic
```

### Priority Hierarchy Quick Reference

| Type | Specificity | Default Priority |
|------|-------------|------------------|
| Item | Ship-to | 10 |
| Item | Customer | 20 |
| Item | Global | 30 |
| Item | Ship-to, Type No. blank | 15 |
| Item | Customer, Type No. blank | 25 |
| Item | Global, Type No. blank | 35 |
| Item Attribute | Ship-to | 40 |
| Item Attribute | Customer | 50 |
| Item Attribute | Global | 60 |
| Item Attribute | Ship-to, Type No. blank | 45 |
| Item Attribute | Customer, Type No. blank | 55 |
| Item Attribute | Global, Type No. blank | 65 |
| Item Category | Ship-to | 70 |
| Item Category | Customer | 80 |
| Item Category | Global | 90 |
| Item Category | Ship-to, Type No. blank | 75 |
| Item Category | Customer, Type No. blank | 85 |
| Item Category | Global, Type No. blank | 95 |

### Glossary

**Active**: Rule state allowing execution; locks configuration fields when true

**Carton Sourcing**: Automatic line splitting to enforce packaging quantity requirements via Quantity Multiple

**Deviated Cost**: Custom unit cost for specific item/customer/vendor scenarios that differs from standard vendor pricing

**Forced Purchasing Code**: Overrides default purchasing code assignment on sales line

**Minimum Quantity**: Sales line quantity threshold that must be met for rule to match

**Priority**: Execution order; lower number = higher precedence; auto-calculated based on Type and specificity

**Quantity Multiple**: Packaging quantity; splits lines if sales line quantity not evenly divisible

**Sourcing Option Found**: Boolean flag on sales line indicating sourcing option matched and applied

**Suggested Vendor**: Vendor recommendation for sales line without forcing specific vendor selection

**Type**: Matching dimension—Item (specific item), Item Category (product category), Item Attribute (tagged items)

**Unit Cost**: Base vendor cost before load factor; system applies load factor automatically when specified

**Wholesale Account Group Code**: Customer-specific wholesale account grouping; sourcing options can override default assignment

---

## Revision History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2025-12-23 | Documentation Team | Initial comprehensive documentation |

---

## Related Documentation

- [Wholesale Sourcing](WHOLESALE-SOURCING.md) - Wholesale Account Priority Groups and Account Groups
- [Load Factor Management](LOAD_FACTOR_MANAGEMENT.md) - Cost adjustment system integrated with sourcing options
- [Bumpdown](BUMPDOWN.md) - Order fulfillment process using sourcing option configurations
- BC Standard: Item Attributes, Sales Line Validation, Purchasing Codes

---

**Support Contact**: BMI Support Team  
**Feature Owner**: Order Fulfillment / Sourcing Operations  
**System**: BMI SupplyAutomate for Microsoft Dynamics 365 Business Central
