# Periodic Invoicing

## 1. OVERVIEW

Periodic Invoicing enables customers to receive consolidated invoices at defined intervals (weekly, monthly, etc.) rather than individual invoices for each sales order. When a customer is configured for periodic invoicing, their sales orders post to customer ledger entries marked as "requiring periodic invoice" with charges accumulating against an accrued revenue G/L account. At the end of each invoicing period, a batch process generates summary invoices that consolidate all accumulated charges, creating a single customer ledger entry and clearing the accrued balance.

This feature solves the business problem of invoice overload for high-frequency customers who place multiple orders daily or weekly. Instead of processing dozens of individual invoices, customers receive a single consolidated invoice per period, reducing administrative overhead for both the customer and accounts receivable processing. The system maintains full traceability by linking each periodic invoice back to the original sales order customer ledger entries.

Periodic Invoicing integrates with standard Business Central customer ledger entries, general journal posting, payment terms, and reporting selections. It extends the sales order processing flow by intercepting invoice posting events and redirecting financial transactions to an accrued revenue account until the periodic invoice is generated. The feature also prevents posting to closed periods, ensuring that once a periodic invoice is created, no additional charges can be added to that period without first reversing the invoice.

## 2. HOW IT WORKS

**Automated Processing:**
- **Trigger**: User runs periodic invoice generation for a specific ending date
- **Step 1**: System finds all Invoicing Period Calendar entries matching the ending date
- **Step 2**: For each calendar entry, system identifies customers assigned to that Invoicing Period Type with Invoicing Type = Periodic
- **Step 3**: For each customer, system locates all customer ledger entries where PeriodicInvoiceRequiredBMU = true, PeriodicInvoiceNoBMU is blank, and Posting Date falls within period start/end dates
- **Step 4**: System calculates total amount by summing all matching customer ledger entry amounts
- **Step 5**: Creates summary customer ledger entry (invoice or credit memo based on amount sign) posting to customer account and accrued periodic invoice G/L account
- **Step 6**: Updates all original customer ledger entries with the generated periodic invoice number, linking them to the summary invoice
- **Outcome**: Single invoice per customer per period consolidating all sales activity, with original entries marked as invoiced

**Manual Operations:**
- **Generate Periodic Invoices**: From Periodic Invoices page, specify ending date and optional customer filter, system creates consolidated invoices
- **Reverse Periodic Invoice**: Select a periodic invoice from the list and run Reverse action to create reversing entry and clear periodic invoice numbers from associated entries
- **Print/Email Invoices**: Standard send/print actions available on Periodic Invoices page using custom report selection
- **Exclude Order from Periodic Billing**: On sales orders, toggle ExcludeFromPeriodicBillingBMU to override periodic invoicing and create standard invoice

**Data Flow:**
1. Customer setup (Customer table) → InvoicingPeriodTypeBMU assigned, InvoicingTypeBMU auto-set to Periodic
2. Sales order creation (Sales Header) → InvoicingTypeBMU copied from customer, payment terms/method validation enforced
3. Sales order posting → Gen. Journal Line.PeriodicInvoiceRequiredBMU set to true
4. Customer ledger entry creation → PeriodicInvoiceRequiredBMU = true, posts to accrued G/L account instead of standard revenue
5. Periodic invoice generation → Creates summary customer ledger entry, updates original entries with PeriodicInvoiceNoBMU
6. Integration: Payment Terms applied to summary invoice due date calculation, Report Selections control print/email output, Source Code Setup tracks periodic invoice transactions

## 3. SETUP

**Primary Setup Page: Sales & Receivables Setup**
- Search: "Sales & Receivables Setup"
- **Accrued Periodic Inv. Account No.** (Required): G/L account for accumulating periodic invoice charges, must allow direct posting
- **Periodic Invoicing Nos** (Required): Number series for generating periodic invoice document numbers

**Primary Setup Page: Source Code Setup**
- Search: "Source Code Setup"
- **Periodic Invoice BMU** (Required): Source code identifying periodic invoice generation transactions in G/L entries and customer ledger entries

**Setup Page: Invoicing Period Types**
- Search: "Invoicing Period Types"
- Create period type codes (e.g., "WEEKLY", "MONTHLY") with descriptions
- Used to group customers with same invoicing frequency
- Cannot delete if assigned to customers or calendar entries

**Setup Page: Invoicing Periods**
- Search: "Invoicing Periods"
- Define individual period records (e.g., "Week 1 2025", "January 2025")
- **Code** (Required): Unique identifier for the period
- **Name** (Required): Descriptive name displayed to users
- **Invoicing Period Type Code** (Required): Links to Invoicing Period Type
- Cannot delete if used in calendar entries

**Setup Page: Invoicing Period Calendar**
- Search: "Invoicing Period Calendar"
- Define date ranges for each period
- **Invoicing Period Code** (Required): Select from Invoicing Periods
- **Starting Date** (Required): First date of period (validates < Ending Date)
- **Ending Date** (Required): Last date of period (validates > Starting Date)
- System prevents overlapping date ranges within same period type
- Periodic invoice generation searches by Ending Date, so ensure calendar covers all needed dates

**Customer Setup:**
- On Customer Card, set **Invoicing Period Type BMU** field
- System automatically sets **Invoicing Type BMU** to Periodic when period type is assigned
- Validates no open unposted periodic orders exist before allowing changes
- Validates no accrued periodic balance exists before allowing changes
- Cannot use payment method with Balancing Account No. when periodic invoicing enabled

**Dependencies:**
- G/L Account structure must include accrued revenue account
- Number Series configured for periodic invoicing documents
- Source Code defined for transaction tracking
- Report Selections configured for Periodic Invoice usage type (optional, for custom report layout)
- Customer payment terms defined (applied to consolidated invoice due dates)

## 4. USER GUIDE

**Task 1: Configure Invoicing Period Types and Calendar**
1. Open "Invoicing Period Types" page and create period type (e.g., Code: "MONTHLY", Description: "Monthly Billing")
2. Open "Invoicing Periods" page and create period records (e.g., Code: "2025-01", Name: "January 2025", Invoicing Period Type Code: "MONTHLY")
3. Open "Invoicing Period Calendar" page and define date ranges (e.g., Invoicing Period Code: "2025-01", Starting Date: 01/01/2025, Ending Date: 01/31/2025)
4. Repeat step 3 for all future periods to ensure complete calendar coverage

**Task 2: Assign Customer to Periodic Invoicing**
1. Open Customer Card for the customer
2. Set **Invoicing Period Type BMU** field to desired period type (e.g., "MONTHLY")
3. System automatically sets **Invoicing Type BMU** to Periodic
4. Verify payment terms are appropriate for consolidated invoicing (discount calculations will apply to summary invoice)
5. Ensure payment method does not have a Balancing Account No. (system enforces this validation)

**Task 3: Generate Periodic Invoices**
1. Open "Periodic Invoices" page (search: "Periodic Invoices")
2. Click Actions → Generate Periodic Invoices (not visible on page, run from search)
3. Enter **Ending Date** matching an Invoicing Period Calendar ending date
4. Optionally filter by **Customer No.** to generate for specific customers only
5. System displays count of invoices generated (one per customer with accumulated charges)
6. Generated invoices appear in Periodic Invoices list with document number, customer, amount, and date range

**Task 4: Print or Email Periodic Invoices**
1. Open "Periodic Invoices" page
2. Select one or more periodic invoice records
3. Choose Send action for document transmission workflow, Print for immediate output, or Email for direct email sending
4. System uses Report Selection for Periodic Invoice usage type to determine report layout

**Task 5: Reverse a Periodic Invoice**
1. Open "Periodic Invoices" page and locate the invoice to reverse
2. Click Actions → Reverse Periodic Invoice
3. System creates reversing customer ledger entry (credit memo if original was invoice, invoice if original was credit memo)
4. Clears PeriodicInvoiceNoBMU from all associated original customer ledger entries, making them available for re-invoicing
5. Marks original periodic invoice record as reversed (PeriodicInvoiceReversedBMU = true)

## 5. TROUBLESHOOTING

**Problem**: Cannot change Invoicing Period Type on customer
**Cause**: Customer has open unposted sales orders with InvoicingTypeBMU = Periodic, or accrued periodic balance exists in customer ledger entries
**Solution**: Either post or delete all open periodic orders, and ensure all accumulated charges have been invoiced through periodic invoice generation. To check accrued balance, search Customer Ledger Entries filtered by PeriodicInvoiceRequiredBMU = true and PeriodicInvoiceNoBMU = blank.

**Problem**: Periodic invoice generation reports "No calendar found for date"
**Cause**: No Invoicing Period Calendar entry exists with Ending Date matching the date entered in generation process
**Solution**: Open "Invoicing Period Calendar" page and verify calendar entries cover the desired ending date. Create missing calendar entry if needed, ensuring Starting Date < Ending Date and no overlap with other periods of same type.

**Problem**: Cannot select payment method on customer with periodic invoicing
**Cause**: Selected payment method has a Balancing Account No. defined, which conflicts with periodic invoicing (individual orders must post to accrued account, not directly offset)
**Solution**: Either select a different payment method without balancing account, or if balancing account payment method is required, customer cannot use periodic invoicing feature.

**Problem**: Sales order shows payment terms but periodic invoice should not have payment terms on individual orders
**Cause**: System validation allows payment terms on individual orders when ExcludeFromPeriodicBillingBMU is checked; prevents payment terms only on true periodic orders
**Solution**: This is by design. Payment terms on individual periodic orders are ignored during posting. Payment terms from customer master data apply to consolidated periodic invoice due date calculation.

**Problem**: Posting error "You cannot post an invoice with a posting date of [DATE]. A periodic invoice has already been created for this period."
**Cause**: Attempting to post sales order with posting date falling within date range of an already-generated periodic invoice that has not been reversed
**Solution**: Either change posting date to fall outside the closed period, reverse the existing periodic invoice to reopen the period, or mark the sales order as ExcludeFromPeriodicBillingBMU to create standard invoice.

## 6. MONITORING

**Key Monitoring Pages:**
- **Periodic Invoices** (Page 71552663): Lists all generated periodic invoices with document number, customer, posting date, amount, and due date. Filter by posting date to review specific periods. Check for reversed invoices using PeriodicInvoiceReversedBMU filter.
- **Customer Ledger Entries**: Use filter PeriodicInvoiceRequiredBMU = true and PeriodicInvoiceNoBMU = blank to find accumulated but not yet invoiced charges. Provides accrued balance by customer.
- **G/L Entries**: Filter by Accrued Periodic Inv. Account No. to monitor total accrued revenue balance. Balance should decrease to zero as periodic invoices are generated.
- **Customer Ledger Entries** (by customer): Filter by PeriodicInvoiceBMU = true to see all periodic invoices for a customer. Use PeriodicInvoiceStartingDateBMU and PeriodicInvoiceEndingDateBMU to understand period coverage.

**Important Reports:**
- Standard Customer Ledger Entry reports filtered for periodic invoices
- G/L Account Balance reports for accrued periodic invoice account trending
- Custom periodic invoice reports via Report Selection Usage = Periodic Invoice BMU

**Health Checks:**
- **Accrued Balance Reconciliation**: Compare sum of uninvoiced customer ledger entries (PeriodicInvoiceRequiredBMU = true, PeriodicInvoiceNoBMU = blank) to G/L balance in Accrued Periodic Inv. Account No. Values should match.
- **Calendar Coverage**: Review Invoicing Period Calendar to ensure future periods are defined before customers reach period end dates. Missing calendar entries prevent invoice generation.
- **Reversed Invoice Follow-up**: Monitor for periodic invoices marked as reversed (PeriodicInvoiceReversedBMU = true). These indicate issues requiring investigation and regeneration of invoices.

## 7. BEST PRACTICES

- **Maintain Complete Calendar**: Define Invoicing Period Calendar entries at least 2-3 periods in advance to prevent generation failures. Schedule quarterly calendar maintenance task.
- **Consistent Period End Dates**: Use consistent ending dates across customers with same period type (e.g., all monthly customers end on last day of month) to simplify batch generation process.
- **Monitor Accrued Balance**: Review uninvoiced customer ledger entries weekly to identify customers approaching period end with significant accrued charges. Prevents surprise large consolidated invoices.
- **Validate Before Generation**: Before running periodic invoice generation, verify Sales & Receivables Setup fields (Accrued Periodic Inv. Account No., Periodic Invoicing Nos) and Source Code Setup (Periodic Invoice BMU) are correctly configured. Missing setup causes generation failure after processing begins.
- **Customer Communication**: Notify customers before transitioning to periodic invoicing. Provide sample consolidated invoice and explain timing of future invoices relative to order placement.
- **Payment Terms Strategy**: Review customer payment terms when enabling periodic invoicing. Due dates calculate from consolidated invoice posting date, not original order dates, potentially extending payment timing.
- **Avoid Mid-Period Changes**: Do not change customer Invoicing Period Type mid-period. Wait until all accumulated charges are invoiced before switching period types to prevent split periods and confusion.

## 8. FIELD REFERENCE

### Invoicing Period Type Table (71552677)
| Field | Type | Description |
|-------|------|-------------|
| Code | Code[10] | Unique identifier for period type (e.g., "WEEKLY", "MONTHLY") |
| Description | Text[50] | User-friendly description of period frequency |

### Invoicing Period Table (71552678)
| Field | Type | Description |
|-------|------|-------------|
| Code | Code[20] | Unique identifier for individual period |
| Name | Text[50] | Descriptive period name (e.g., "January 2025") |
| Invoicing Period Type Code | Code[10] | Links to Invoicing Period Type |

### Invoicing Period Calendar Table (71552679)
| Field | Type | Description |
|-------|------|-------------|
| Invoicing Period Code | Code[20] | Links to Invoicing Period |
| Invoicing Period Type Code | Code[10] | Copied from Invoicing Period, defines period type |
| Starting Date | Date | First date of period, must be < Ending Date |
| Ending Date | Date | Last date of period, used for invoice generation lookup |

### Customer Table Extensions
| Field | Type | Description |
|-------|------|-------------|
| Invoicing Type BMU (71552584) | Enum (Manual/Periodic) | Auto-set based on Invoicing Period Type, controls order processing |
| Invoicing Period Type BMU (71552585) | Code[10] | Assigns customer to periodic invoicing schedule, links to Invoicing Period Type |

### Customer Ledger Entry Extensions
| Field | Type | Description |
|-------|------|-------------|
| Periodic Invoice Required BMU (71552575) | Boolean | True when entry originates from periodic sales order, indicates needs consolidation |
| Periodic Invoice No. BMU (71552576) | Code[20] | Document number of consolidated periodic invoice, blank until invoiced |
| Periodic Invoice BMU (71552577) | Boolean | True when entry is the consolidated periodic invoice itself |
| Periodic Invoice Reversed BMU (71552578) | Boolean | True when periodic invoice has been reversed, prevents reuse |
| Periodic Invoice Starting Date BMU (71552579) | Date | Start of period covered by consolidated invoice |
| Periodic Invoice Ending Date BMU (71552580) | Date | End of period covered by consolidated invoice |
| Periodic Invoice Pmt BMU (71552583) | Boolean | True for payment entries related to periodic invoices |

### Sales Header Extensions
| Field | Type | Description |
|-------|------|-------------|
| Invoicing Type BMU (71552597) | Enum (Manual/Periodic) | Copied from customer, controls posting behavior |
| Exclude From Periodic Billing BMU (71552639) | Boolean | Override to create standard invoice even when customer uses periodic invoicing |

### Sales & Receivables Setup Extensions
| Field | Type | Description |
|-------|------|-------------|
| Accrued Periodic Inv. Account No. BMU (71552575) | Code[20] | G/L account for accumulating periodic charges, must allow direct posting |
| Periodic Invoicing Nos BMU (71552576) | Code[20] | Number series for generating periodic invoice document numbers |

### Source Code Setup Extensions
| Field | Type | Description |
|-------|------|-------------|
| Periodic Invoice BMU (71552575) | Code[10] | Source code for periodic invoice generation transactions |

## 9. ADVANCED TOPICS

**Complex Scenario: Mid-Period Customer Assignment**
When assigning a customer to periodic invoicing mid-period, existing open sales orders retain original InvoicingTypeBMU = Manual unless manually changed. New sales orders will automatically use Periodic. Consider timing of transition to align with natural period boundaries to avoid partial periods. If mid-period transition is unavoidable, first periodic invoice will only include orders placed after assignment date.

**Period Date Validation and Overlap Prevention**
InvoicingPeriodCalendarBMU.CheckDates() procedure enforces Starting Date < Ending Date and prevents overlapping periods within same period type. Validation uses SystemId to exclude current record when checking overlaps. This ensures clean period boundaries but requires careful calendar planning when defining consecutive periods.

**Accrued Revenue Account Flow**
When sales order posts with InvoicingTypeBMU = Periodic and ExcludeFromPeriodicBillingBMU = false, Gen. Journal Line.PeriodicInvoiceRequiredBMU = true flows to Customer Ledger Entry. SalesHeader.ValidateInvoicingType() sets Bal. Account Type = G/L Account and Bal. Account No. = AccruedPeriodicInvAccountNoBMU, redirecting revenue posting from standard accounts. Periodic invoice generation reverses accrued account and creates final customer invoice.

**Event Subscriber Integration Points**
- **OnAfterSetFieldsBilltoCustomer**: Copies InvoicingTypeBMU from customer to sales header when bill-to customer assigned
- **OnBeforeValidateEvent Payment Terms Code/Payment Method Code**: Validates no payment terms or balancing account payment methods on periodic orders
- **OnAfterCopyGenJnlLineFromSalesHeader**: Flows PeriodicInvoiceRequiredBMU, DepartmentCodeBMU, ShipToCodeBMU to Gen. Journal Line
- **OnAfterInitCustLedgEntry**: Flows periodic invoice fields from Gen. Journal Line to Customer Ledger Entry
- **OnCheckAndUpdateOnAfterSetPostingFlags**: Calls CheckOpenInvoicingPeriod() to prevent posting to closed periods
- **Report Selection/Email Scenario events**: Maps Periodic Invoice usage type to report selection and email scenarios

**Dimension Handling**
GetCustomerDimensionSetID() procedure retrieves default dimensions from customer and creates dimension set for periodic invoice. Dimensions from original sales orders are not consolidated; periodic invoice uses customer-level dimensions only. If order-specific dimension tracking is required, consider excluding those orders from periodic billing.

**Reversing Periodic Invoices Impact**
ReversePeriodicInvoice() procedure creates offsetting customer ledger entry, clears PeriodicInvoiceNoBMU from associated entries, and marks original as reversed. Accrued account balance increases back to pre-invoice level. Associated entries become available for re-invoicing in next periodic invoice generation. Apply-to document logic allows closing reversed invoice if still open. Consider cash application implications before reversing paid invoices.

## 10. APPENDIX

**Quick Reference: Periodic Invoice Generation Process**
1. Verify calendar entry exists for desired ending date
2. Open Periodic Invoices page → Search "Generate Periodic Invoices"
3. Enter Ending Date = calendar entry ending date
4. Optional: Filter by Customer No. for selective generation
5. System generates invoices and displays count message
6. Review generated invoices on Periodic Invoices page
7. Print/email invoices using Send, Print, or Email actions

**Glossary:**
- **Accrued Revenue**: Financial accounting concept where revenue is recognized when earned but not yet invoiced; periodic invoicing uses accrued account to temporarily hold charges
- **Period Type**: Category of invoicing frequency (e.g., Weekly, Monthly) used to group customers with same billing cycle
- **Period Calendar**: Date range definitions for each specific invoicing period, provides mapping from dates to period codes
- **Consolidated Invoice**: Single invoice combining multiple sales orders within a period, created by periodic invoice generation process
- **Exclude From Periodic Billing**: Order-level override allowing creation of standard immediate invoice even when customer configured for periodic invoicing

**Common Scenarios:**

*Scenario 1: Customer places 20 orders in January*
- Each order posts creating customer ledger entry with PeriodicInvoiceRequiredBMU = true
- All 20 entries post to accrued periodic invoice G/L account
- At January 31, generate periodic invoice for ending date 01/31/2025
- System creates single consolidated invoice totaling all 20 orders
- All 20 original entries updated with periodic invoice document number
- Customer receives one invoice instead of 20

*Scenario 2: Customer needs immediate invoice for one urgent order*
- Open sales order and set ExcludeFromPeriodicBillingBMU = true
- Order posts with PeriodicInvoiceRequiredBMU = false
- Creates standard customer ledger entry posting to normal revenue accounts
- Customer receives immediate invoice for this order
- Remaining periodic orders continue accumulating for period-end consolidation

*Scenario 3: Mistake in periodic invoice requires correction*
- Identify incorrect periodic invoice on Periodic Invoices page
- Run Reverse Periodic Invoice action
- System creates offsetting entry and clears periodic invoice number from original entries
- Original entries become available for correction and re-invoicing
- Regenerate periodic invoice after corrections applied to source orders

**Related Documentation:**
- Customer Ledger Entries (standard BC documentation)
- General Journal Posting (standard BC documentation)
- Report Selections Setup (standard BC documentation)
- Payment Terms and Due Date Calculation (standard BC documentation)

**Support Information:**
- Feature Code: PeriodicInvoicingMgtBMU (Codeunit 71552630)
- Event Handling: PeriodicInvoicingEventMgtBMU (Codeunit 71552628)
- Primary Tables: InvoicingPeriodTypeBMU (71552677), InvoicingPeriodCalendarBMU (71552679)
- User Interface: PeriodicInvoicesBMU (Page 71552663)

---

**Document Information:**
- Feature: Periodic Invoicing
- Version: 1.0
- Last Updated: December 23, 2025
- Document Type: Technical Reference (Concise)
