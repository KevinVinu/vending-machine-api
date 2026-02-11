# Bug Fixes

### Slot Creation Limit Not Enforced
- Issue: Earlier, the API allowed creating unlimited slots, ignoring the configured maximum limit.
- Impact: Could exceed MAX_SLOTS and break system constraints.
- Fix: Added a check in create_slot to raise HTTPException when trying to create more than settings.MAX_SLOTS.
- Result: API now enforces slot limits and returns proper 400 Slot limit reached errors.

### Max Item Capacity Not Checked
- Issue: The API did not prevent a slot from being created with more items than MAX_ITEMS_PER_SLOT.
- Impact: Could violate business logic and allow slots to overflow.
- Fix: Added Max item capacity reached check in the slot creation flow and returned 409 Conflict.
- Result: Slots now respect maximum item capacity defined in config.py.

### add_item_to_slot Capacity Check Fix
- Issue: The earlier code had a conflicting capacity check (< settings.MAX_ITEMS_PER_SLOT) that was logically incorrect.
- Impact: Could incorrectly raise capacity errors even when adding valid item quantities.
- Fix: Removed the invalid check and now only validates against slot.capacity.
- Result: Items are correctly added without exceeding slot limits.

### bulk_add_items Capacity and Commit Logic
- Issue: bulk_add_items did not check slot capacity and committed inside the loop, allowing partial additions on errors.
- Impact: Could overflow slots and leave inconsistent database state.
- Fix: Added per-item capacity check and moved db.commit() outside the loop.
- Result: Bulk additions are safe, atomic, and respect slot capacity.

### update_item_price Timestamp Fix
- Issue: update_item_price manually reset updated_at, preventing it from reflecting actual modifications.
- Impact: Audit logs for price changes were inaccurate.
- Fix: Removed manual updated_at handling; SQLAlchemy now auto-updates timestamps.
- Result: updated_at correctly reflects item price changes.

### remove_item_quantity Negative Count Prevention
- Issue: Removing items could make current_item_count negative in edge cases.
- Impact: Negative counts could break capacity logic and cause errors in subsequent operations.
- Fix: Used max(0, slot.current_item_count - removed_quantity) to prevent negative values.
- Result: Slot counts always remain non-negative and accurate.

### bulk_remove_items Negative Count Prevention
- Issue: Bulk removal did not prevent current_item_count from going negative.
- Impact: Could corrupt slot state when all items are removed.
- Fix: Applied max(0, slot.current_item_count - item.quantity) during deletions.
- Result: Slot counts stay valid after bulk removals.

### Slot Creation Limits & Max Capacity Fix
- Issue: Earlier, slots could be created without enforcing MAX_SLOTS or MAX_ITEMS_PER_SLOT.
- Impact: Could exceed configured maximums, leading to inconsistent slot management and potential API errors.
- Fix: Added checks to raise ValueError if data.capacity > MAX_ITEMS_PER_SLOT or total slots exceed MAX_SLOTS; also ensured duplicate slot codes are rejected.
- Result: Slot creation now respects configuration limits and prevents duplicate codes, maintaining API consistency.

### Database Models Cascade & ForeignKey Fix
- Issue: Earlier, deleting a slot automatically deleted all related items due to cascade="all, delete-orphan" and ondelete="CASCADE", which could lead to unintended data loss.
- Impact: Removing a slot could erase important item data without explicit intent, affecting audit and inventory tracking.
- Fix: Updated cascade to "save-update, merge" and ondelete to "SET NULL" to preserve items when a slot is deleted.
- Result: Slot deletions no longer remove items automatically; items now retain their data for accurate inventory management.

### Database Session & Engine Configuration Fix
- Issue: Earlier, the database engine and session had legacy behaviors that caused warnings and potential unexpected session expiration.
- Impact: Could lead to inconsistent database sessions, stale data, or warning messages during API operations.
- Fix: Added future=True in create_engine to prevent legacy behavior warnings and expire_on_commit=False in sessionmaker to keep objects valid after commit.
- Result: Database sessions now behave consistently, objects remain accessible post-commit, and warnings are eliminated.

### Pydantic Field Validation Fix
- Issue: Earlier, price and cash_inserted fields allowed 0 or negative values in some schemas (ItemCreate, PurchaseRequest).
- Impact: Could result in items with invalid pricing or purchases with zero/negative cash, breaking API logic.
- Fix: Updated fields to use gt=0 (greater than zero) for critical numeric inputs while allowing non-negative prices where needed.
- Result: API now enforces proper validation on price and cash insertion, preventing invalid data entry.
