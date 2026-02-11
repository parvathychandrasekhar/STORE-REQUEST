# Store Request - Odoo Module

## Overview

The **Store Request** module is an Odoo addon that facilitates internal warehouse material requests and transfers. It enables employees to request products from the warehouse, track transfers, and manage inventory movements with integration to Bill of Materials (BOM) and Sales Orders.

## Key Features

- **Internal Material Requests**: Create requests for products from warehouse locations
- **BOM Integration**: Automatically populate request lines from Bill of Materials
- **Sales Order Integration**: Link requests to sales orders and auto-populate components
- **Multi-step Transfer Workflow**: Draft → Transfer → Done states
- **Partial Transfers**: Support for partial quantity fulfillment with tracking
- **Real-time Inventory Validation**: Check stock availability before creating transfers
- **Transfer Tracking**: Monitor validated and non-validated transfers
- **Lot/Serial Number Tracking**: Track which lots were transferred for each request

## How It Works

### 1. **Creating a Store Request**

When a user creates a store request, they can:
- Select an **Operation Type** (picking type) which determines source/destination locations
- Optionally link to a **Sale Order** to auto-populate components based on the ordered product's BOM
- Manually select a **Bill of Materials** to populate request lines
- Manually add products with required quantities

**Workflow:**
```
User creates request → Select picking type → Choose BOM or Sale Order → Request lines auto-populate → Submit request
```

### 2. **Bill of Materials (BOM) Preview**

The module extends the standard MRP BOM with a **BOM Preview Line** feature:
- Allows pre-defining component lists with quantities and prices
- When a BOM is selected in a request, these preview lines automatically populate the request lines
- Quantities are multiplied by the demanded quantity from the sale order

**Models:**
- `bom.preview.line`: Stores preview components for each BOM
- Each line contains: Product, Quantity, Unit of Measure, Price

### 3. **Request Line Management**

Each request contains multiple lines (`stock.request.lines`):

**Fields:**
- `product_id`: The product being requested
- `required_qty`: Total quantity needed
- `qty_to_transfer`: Quantity to transfer in current operation
- `transferred_qty`: Cumulative quantity already transferred
- `pending_qty`: Calculated as `required_qty - transferred_qty`
- `product_uom`: Unit of measure for the product

**Status Tracking:**
- `is_requirement_satisfied`: Set to True when `transferred_qty` equals `required_qty`
- `has_unvalidated_transfer`: Indicates if there are pending (unvalidated) transfers

### 4. **Transfer Workflow**

#### Step 1: Create Transfer
When a user clicks "Transfer" on a request line:
1. **Stock Availability Check**: System checks if enough stock exists in source location
2. **UOM Conversion**: Handles conversion between product UOM and stock UOM
3. **Validation**: Ensures:
   - `qty_to_transfer` is greater than 0
   - No pending unvalidated transfers exist
   - Quantity doesn't exceed pending quantity
   - Stock is available in source location

4. **Picking Creation**: Creates a `stock.picking` record with:
   - Picking type from request
   - Source and destination locations
   - Link to request line
   - Request recipient (`request_to` field)

5. **Stock Move Creation**: Creates `stock.move` with:
   - Product and quantity (converted to stock UOM)
   - Source and destination locations
   - Link to picking

6. **Automatic Actions**:
   - `picking.action_confirm()`: Confirms the picking
   - `picking.action_assign()`: Reserves the stock

#### Step 2: Validate Transfer
When a user clicks "Validate":
1. **Picking Validation**: Calls `button_validate()` on the picking
2. **Lot Tracking**: Extracts lot/serial numbers from move lines
3. **Quantity Update**: 
   - Updates `transferred_qty` += `qty_to_transfer`
   - Recalculates `pending_qty`
   - Resets `qty_to_transfer` to 0
4. **Requirement Check**: If `transferred_qty` matches `required_qty`, sets `is_requirement_satisfied = True`

### 5. **Stock Picking Extensions**

The module extends `stock.picking` with:
- `stock_request_line_id`: Link back to the request line
- `stock_request_id`: Link to the parent request
- `request_to`: The user requesting the materials
- `product_and_qty`: Computed field displaying all products and quantities in the picking

### 6. **State Management**

**Request States:**
```
draft → transfer → done
```

- **Draft**: Initial state, request can be edited
- **Transfer**: Request submitted, transfers can be created
- **Done**: All request lines satisfied (all `is_requirement_satisfied = True`)

**State Transitions:**
- `action_store_request()`: Moves from draft → transfer
- `action_done()`: Validates all lines are satisfied, then moves to done
- Prevents marking as done if any line has `pending_qty > 0`

### 7. **Re-Request Feature**

If a request needs to be resubmitted:
- `re_request_transfer()` sets `is_re_requested = True`
- Allows tracking of requests that required multiple attempts

### 8. **Smart Buttons & Tracking**

The request form displays:
- **Validated Transfers**: Count and access to completed pickings (`state = 'done'`)
- **Non-Validated Transfers**: Count and access to pending pickings (`state not in ['done', 'cancel']`)

Both buttons open filtered views showing only relevant pickings with custom tree view.

## Data Flow Diagram

```
Sale Order Line → BOM → BOM Preview Lines
                           ↓
                    Stock Request
                           ↓
              Stock Request Lines (Multiple)
                           ↓
           [Transfer Button] → Stock Picking + Stock Move
                           ↓
          [Validate Button] → Transfer Validated
                           ↓
                Quantities Updated → Lots Tracked
                           ↓
            All Lines Satisfied? → Mark Request Done
```

## Technical Details

### Models

1. **stock.request**
   - Main request model
   - Inherits: `mail.thread`, `mail.activity.mixin`
   - Sequence: `REQ00001` format

2. **stock.request.lines**
   - Request line items
   - One2many relationship with stock.request
   - Many2one relationship with product.product

3. **bom.preview.line**
   - BOM component preview
   - Links to mrp.bom
   - Stores component details with pricing

4. **mrp.bom** (inherited)
   - Added `preview_line_ids` field

5. **stock.picking** (inherited)
   - Added request tracking fields
   - Computed `product_and_qty` field

### Key Computed Fields

- **demanded_qty**: Auto-filled from sale order line quantity
- **pending_qty**: `required_qty - transferred_qty`
- **product_and_qty**: Formatted string of products in picking
- **validated_picking_count**: Count of done pickings
- **non_validated_picking_count**: Count of pending pickings

### Security

Access rights defined in `ir.model.access.csv`:
- All users can read/write/create/delete BOM preview lines
- All users can manage stock requests and request lines

## Installation

1. Place the module in your Odoo `custom_addons` directory
2. Update the apps list: `Settings → Apps → Update Apps List`
3. Search for "Store Request"
4. Click "Install"

## Dependencies

- `stock` - Inventory management
- `sale` - Sales module
- `mrp` - Manufacturing (optional, for BOM features)

## Usage Example

### Scenario: Requesting Materials for a Sale Order

1. **User receives a sale order** for Product X (quantity: 10 units)
2. **Creates a Store Request**:
   - Selects "Internal Transfer" as operation type
   - Selects the sale order
   - Selects the order line for Product X
   - System auto-populates request lines from Product X's BOM
3. **Request Lines Show**:
   - Component A: Required 50, Pending 50
   - Component B: Required 30, Pending 30
4. **User Transfers Component A**:
   - Sets qty_to_transfer = 50
   - Clicks "Transfer" → Picking created and reserved
   - Clicks "Validate" → Transfer completed
   - Component A: Transferred 50, Pending 0, Status: ✓ Satisfied
5. **User Transfers Component B Partially**:
   - Sets qty_to_transfer = 20 (partial)
   - Transfer and validate
   - Component B: Transferred 20, Pending 10
6. **User Completes Component B**:
   - Sets qty_to_transfer = 10
   - Transfer and validate
   - Component B: Transferred 30, Pending 0, Status: ✓ Satisfied
7. **Request Completion**:
   - User clicks "Done" button
   - System validates all lines are satisfied
   - Request marked as Done

## Validation Rules

- Cannot transfer more than required quantity
- Cannot transfer when stock is unavailable
- Cannot validate request until all lines satisfied
- Cannot create multiple unvalidated transfers for same line
- Unit of measure conversions handled automatically

## Cleanup on Delete

When a request or request line is deleted:
- All associated stock moves are unmarked
- Pickings are unmarked from request tracking
- Ensures data integrity
