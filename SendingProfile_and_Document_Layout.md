# Business Central Document Sending Profiles & Customer Document Layouts
Enhanced User Training Manual

## Table of Contents

1. Overview
2. Document Sending Profiles
3. Assigning Profiles to Customers
4. Customer Document Layouts
5. Process Flow & Best Practices

## 1. Overview

**Purpose:** This training manual explains how to set up and use Document Sending Profiles and Customer Document Layouts in Microsoft Dynamics 365 Business Central.

**Scope:** For all staff involved in invoicing, sales operations, and customer communication.

**Definitions:**

- **Document Sending Profile** — Controls how documents are delivered (email, print, file), by the Default Sending Profile or by a customer specified profile.
- **Document Layouts** — Determine **what the document looks like**—which report format is used.
- **Customer Document Layout** — Controls what the document looks like per customer if a custom layout is required.

## 2. Document Sending Profiles

### Step 1: Access the Document Sending Profiles

Use **Search → Document Sending Profiles**.

Each customer can be set up with a preferred method of sending sales documents, so that you do not have to select a sending option every time you choose the **Post and Send** action.

On the **Document Sending Profiles** page, you set up different sending profiles that you can select from in the **Document Sending Profile** field on a customer card. You can select the **Default** check box to specify that the document sending profile is the default profile for all customers, except for customers where the **Document Sending Profile** field is filled with another sending profile.

When you choose the **Post and Send** action on a sales document, the **Post and Send Confirmation** dialog box shows the sending profile used, either the one set up for the customer or the default for all customers. In the dialog box, you can change the sending profile for the sales document.

### Step 2: Create or Edit a Sending Profile

- **Code** – Identifier such as `EMAILONLY`
- **Description** – Friendly display name
- **Delivery Method** (Email, Printer, Disk)
- **Email Subject** placeholders (`%1` = customer name, `%2` = document no.)
- Attach **PDF** or **Word** layouts

## 3. Assigning Profiles to Customers

1. Navigate to **Sales → Customers** and select a customer.
2. Choose **Navigate → Customer → Document Sending Profile**.
3. Select the default profile (e.g., `EMAILONLY`).

## 4. Customer Document Layouts

- **Access:** Customer Card → **Navigate → Customer → Document Layouts**
- **Usage** (Invoice, Order, Statement, etc.)
- **Report ID** (specifies a standard report with a custom layout or a custom report)
- Optional: **Email Attachment Layout** (specifies the report layout)
- Optional: **Email Body Layout** (specifies the email body layout)
- Optional: Override email address

## 5. Process Flow & Best Practices

### Process Flow

1. Business Central checks **Customer Document Layouts** first.
2. If none exist, default **Report Selections** are used.
3. **Sending Profile** determines delivery behavior.
4. Document is generated and sent/printed/saved.

### Best Practices

- Use `EMAILONLY` profile unless customer requires printing.
- Maintain **Report Selections** regularly.
- Train staff to differentiate **Preview** vs **Send**.
- Use custom layouts for branded or high‑value customers.
