### **1.4. Active Databases/Triggering**

#### **b) Set deregistration date when copy condition is ‘deteriorated’**  

**Design**  
- **Objective**: Automatically set `DEREGISTERED` to SYSDATE when `CONDITION` is updated to 'D'.  
- **Tables Involved**: `copies`.  
- **Trigger Logic**:  
  - Before updating `CONDITION`, check if the new value is 'D' and the old value is not 'D'.  
  - Set `DEREGISTERED` to the current date only when the condition transitions to 'D'.  

**Implementation**  
```sql
CREATE OR REPLACE TRIGGER set_deregistration_date
BEFORE UPDATE OF CONDITION ON copies
FOR EACH ROW
BEGIN
    IF :NEW.CONDITION = 'D' AND (:OLD.CONDITION != 'D' OR :OLD.CONDITION IS NULL) THEN
        :NEW.DEREGISTERED := SYSDATE;
    END IF;
END;
/
```

**Testing**  
```SQL
-- Setup: Insert a Copy with Condition 'G'
INSERT INTO editions (ISBN, TITLE, AUTHOR, LANGUAGE, NATIONAL_LIB_ID) 
VALUES ('ISBN_TEST', 'Test Book', 'Test Author', 'Spanish', 'NL_TEST');

INSERT INTO copies (SIGNATURE, ISBN, CONDITION) 
VALUES ('STEST', 'ISBN_TEST', 'G');

-- Test 1: Update to 'D' (Set DEREGISTERED)
UPDATE copies SET CONDITION = 'D' WHERE SIGNATURE = 'STEST';
SELECT SIGNATURE, CONDITION, DEREGISTERED FROM copies WHERE SIGNATURE = 'STEST'; -- DEREGISTERED should be SYSDATE

-- Test 2: Update to 'W' (No Change)
UPDATE copies SET CONDITION = 'W' WHERE SIGNATURE = 'STEST';
SELECT DEREGISTERED FROM copies WHERE SIGNATURE = 'STEST'; -- DEREGISTERED remains unchanged

-- Test 3: Update Back to 'D' (Update DEREGISTERED Again)
UPDATE copies SET CONDITION = 'D' WHERE SIGNATURE = 'STEST';
SELECT DEREGISTERED FROM copies WHERE SIGNATURE = 'STEST'; -- DEREGISTERED updates to new SYSDATE

```

---

**Notes**:  
- **Trigger b** updates `DEREGISTERED` every time `CONDITION` is set to 'D'. If business rules require it to be set only once, modify the trigger to check if `DEREGISTERED` is already populated.  
- Error codes (`-20008`, `-20009`) are arbitrary but should be unique for debugging.
  
  