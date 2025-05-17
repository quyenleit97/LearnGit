# Detailed Documentation: updateInfoCustomer SOAP Web Service

## 1. Overview

The `updateInfoCustomer` function is a SOAP web service endpoint provided in the MDealerEndpoint class of the mDealer application. It allows dealers to update existing customer information associated with a mobile phone number (ISDN). This web service is responsible for validating, updating, and storing modified customer information including personal details and identification documents.

## 2. Request Parameters

The `UpdateInfoCustomerRequest` class contains the following fields:

| Parameter | Type | Description | Required |
|-----------|------|-------------|---------|
| isdn | String | Mobile phone number | Required |
| dealerIsdn | String | Dealer's mobile phone number | Required |
| idType | String | Type of identification document | Required |
| name | String | Customer's full name | Required |
| idNo | String | Identification document number | Required |
| birthDay | String | Customer's date of birth (in DD/MM/YYYY format) | Required |
| expireDate | String | ID document expiration date | Optional |
| address | String | Customer's address | Required |
| province | String | Customer's province | Required |
| sex | String | Customer's gender | Required |
| isDetect | String | Flag indicating if OCR detection should be used | Required |
| lstImage | List<ImageIn> | List of customer's identification images (must include at least 2 images) | Required |
| action | String | Action code for the transaction | Required |
| transactionId | String | Unique transaction identifier | Optional |

## 3. Service Flow and Logic

The service performs the following steps:

### 3.1. Input Validation

**Step 1: Basic Parameter Validation**
1. Validate that ISDN is provided (Error: error.empty.isdn)
2. Validate dealer's phone number is provided (Error: error.empty.account)
3. Validate customer name is provided (Error: error.empty.name)

**Step 2: ID Document Validation**
1. Validate ID document type is provided (Error: error.empty.id.type)
2. Ensure ID type is not "Others" (Error: error.unallow.id.type)
3. Validate ID number is provided (Error: error.empty.id.no)

**Step 3: Personal Information Validation**
1. Validate date of birth is provided (Error: error.empty.dob)
2. Ensure date of birth is in valid format (DD/MM/YYYY) (Error: error.invalid.dob)
3. Validate gender is provided (Error: error.empty.sex)
4. Validate address is provided (Error: error.empty.address)

**Step 4: Document Validation**
1. Ensure at least 2 identification images are provided (Error: error.empty.image)

For each missing or invalid field, the system returns an appropriate error message with the corresponding error code.

### 3.2. Business Logic Validation

**Step 5: Format and Normalize Data**
1. Format the subscriber phone number (ISDN) to standardized format
   - Removes any non-numeric characters and applies standard phone number format
   - Ensures ISDN follows the required format for the system

2. Format the dealer phone number to standardized format 
   - Similar standardization for dealer phone number
   - Error occurs if format is invalid after standardization

**Step 6: ISDN Verification**
1. Check if the ISDN is valid and active by querying the subscriber database (Error: error.isdn.invalid)
   - Error occurs if query returns empty results (ISDN not found)
   - ACT_STATUS = 1 means the subscription is active
   ```sql
   SELECT * FROM SUB_MB
   WHERE (ISDN = ? AND STATUS = 2)
   ```

**Step 7: Package Validation**
1. Check if the subscription package is in the restricted list (Error: error.package.invalid)
   - Error occurs if the product code associated with the ISDN is in the whitelist
   - The system compares the ISDN's product code against a predefined list of restricted products

2. Verify the subscription is active in the system (Error: error.isdn.invalid.check)
   ```sql
   -- Check if subscription has active status
   -- ACT_STATUS = '01' means active
   SELECT ACT_STATUS FROM SUB_MB WHERE ISDN = ?
   ```

**Step 8: Dealer Account Validation**
1. Validate dealer account exists and is active (Error: error.not.account.agent)
   ```sql
   SELECT * FROM ACCOUNT_AGENT 
   WHERE ACCOUNT = ? AND STATUS = 1
   ```

2. Validate owner (staff/shop) based on account type
   - For staff owners (Error: error.account.agent.invalid)
   ```sql
   SELECT * FROM STAFF 
   WHERE STAFF_ID = ? AND STATUS = 1
   ```
   - For shop owners (Error: error.account.agent.not.exist)
   ```sql
   SELECT * FROM SHOP 
   WHERE SHOP_ID = ? AND STATUS = 1
   ```

3. Check ID number registration limit (Error: error_unallow_register_info)
   ```sql
   -- Count existing subscriptions with this ID number to enforce limits
   SELECT COUNT(*) FROM SUB_MB WHERE ID_NO = ?
   ```

### 3.3. Processing

**Step 9: OCR Detection Handling**
1. Extract action code for updating customer information from the request
   - The system processes the action code to determine the appropriate update type

2. If OCR detection is disabled (`isDetect != "1"`):
   - Forward update request to Customer Care system
   - The system sends the customer update request with all relevant information to the Customer Care system 
   - Information sent includes: ISDN, name, ID type, ID number, gender, birth date, address, province, images, action code, dealer ISDN, customer ID, and registration flag

**Step 10: Disable Previous Customer Images**
1. Mark previous customer images as inactive to avoid duplication
   ```sql
   UPDATE CUST_IMAGE 
   SET STATUS = 0
   WHERE ISDN = ? AND DEALER_CODE = ?
   ```

**Step 11: Create New Customer Image Record**
1. Generate sequence IDs for new customer images
   ```sql
   -- Getting sequence numbers from CM_PRE database
   SELECT ACCOUNT_IMAGE_SEQ.NEXTVAL FROM DUAL
   ```

2. Create new customer image record to store updated information
   ```sql
   INSERT INTO CUST_IMAGE (
       ID, ID_NO, NAME, CUST_ID, AGENT_CODE, DEALER_CODE, 
       SERIAL, ISDN, OWNER_TYPE, STATUS, CREATE_DATE, CREATE_USER, SOURCE_TYPE
   ) VALUES (
       ?, ?, ?, ?, ?, ?, ?, ?, ?, 1, SYSDATE, ?, ?
   )
   ```

**Step 12: Upload and Store Identification Images**
1. For each image in the request:
   - Generate unique sequence ID for image details
   ```sql
   SELECT CUST_IMAGE_DETAIL_SEQ.NEXTVAL FROM DUAL
   ```

   - Create unique image filename with timestamp
   - The system generates a unique filename using a combination of the dealer code, current timestamp, and sequence ID

   - Upload image to FTP server
   - The system uploads the image file to an FTP server using the appropriate file transfer protocol

   - Verify upload success (Error: error.upload.image)
   - If the upload fails, the system returns an error

   - Store image metadata in database
   ```sql
   INSERT INTO CUST_IMAGE_DETAIL (
       ID, IMAGE_TYPE, FILENAME, DEALER_CODE, FTP_IP, 
       FTP_PATH, CUST_IMAGE_ID, STATUS, CREATE_DATE
   ) VALUES (
       ?, ?, ?, ?, ?, ?, ?, 1, SYSDATE
   )
   ```

**Step 13: Generate Action Audit Record**
1. Create audit sequence number for tracking updates
   ```sql
   SELECT ACTION_AUDIT_SEQ.NEXTVAL FROM DUAL
   ```

2. Prepare description for audit record
   - The system generates a description string that includes dealer information for auditing purposes

**Step 14: Check Existing Customer Images**
1. Verify if customer images exist for the subscription
   ```sql
   SELECT COUNT(*) FROM CUST_IMAGE
   WHERE STATUS = 1 AND (SERIAL = ? OR ISDN = ?)
   ```

**Step 15-16: Customer Data Management**
1. If customer images were previously uploaded:
   - Retrieve active subscriptions for the customer ID
   ```sql
   SELECT * FROM SUB_MB 
   WHERE CUST_ID = ? AND STATUS = 2 AND PRODUCT_CODE NOT LIKE 'T2%'
   ```

   - Get current customer information
   ```sql
   SELECT * FROM CUSTOMER
   WHERE CUST_ID = ?
   ```

   - If customer exists with active subscriptions:
     - Update existing customer record with new information
     ```sql
     UPDATE CUSTOMER
     SET NAME = ?, SEX = ?, PROVINCE = ?, ADDRESS = ?,
         ID_TYPE = ?, ID_NO = ?, BIRTH_DATE = TO_DATE(?, 'DD/MM/YYYY'),
         ID_EXPIRE_DATE = TO_DATE(?, 'DD/MM/YYYY'), UPDATED_TIME = SYSDATE
     WHERE CUST_ID = ?
     ```

     - Create detailed audit records for each changed field
     ```sql
     INSERT INTO ACTION_DETAIL
     (ACTION_AUDIT_ID, OLD_VALUE, NEW_VALUE, REASON_ID, FIELD_NAME)
     VALUES (?, ?, ?, ?, ?)
     ```

     - Store original customer ID for reference
     - The system keeps track of the original customer ID value for audit purposes

   - If no active customer record found:
     - Create new customer record
     ```sql
     INSERT INTO CUSTOMER (
         CUST_ID, BUS_TYPE, NAME, BIRTH_DATE, SEX, ID_NO, ID_TYPE,
         ID_ISSUE_DATE, ID_EXPIRE_DATE, PROVINCE, STATUS, ADDED_DATE,
         UPDATED_TIME, NOTES, CORRECT_CUS, ADDRESS
     ) VALUES (
         CUSTOMER_SEQ.NEXTVAL, 1, ?, TO_DATE(?, 'DD/MM/YYYY'), ?,
         ?, ?, SYSDATE, TO_DATE(?, 'DD/MM/YYYY'), ?, 1, SYSDATE,
         SYSDATE, 'UPDATE BY MDEALER', 1, ?
     )
     ```

2. If no previous customer images found:
   - Create new customer record
   ```sql
   INSERT INTO CUSTOMER (
       CUST_ID, BUS_TYPE, NAME, BIRTH_DATE, SEX, ID_NO, ID_TYPE,
       ID_ISSUE_DATE, ID_EXPIRE_DATE, PROVINCE, STATUS, ADDED_DATE,
       UPDATED_TIME, NOTES, CORRECT_CUS, ADDRESS
   ) VALUES (
       CUSTOMER_SEQ.NEXTVAL, 1, ?, TO_DATE(?, 'DD/MM/YYYY'), ?,
       ?, ?, SYSDATE, TO_DATE(?, 'DD/MM/YYYY'), ?, 1, SYSDATE,
       SYSDATE, 'UPDATE BY MDEALER', 1, ?
   )
   ```

**Step 17: Update Subscriber Information**
1. Store new customer ID
   - The system keeps track of the newly assigned customer ID

2. Update subscriber record with updated customer information
   ```sql
   UPDATE SUB_MB
   SET CUST_ID = ?, NAME = ?, BIRTH_DATE = TO_DATE(?, 'DD/MM/YYYY'),
       SEX = ?, PROVINCE = ?, ADDRESS = ?
   WHERE SUB_ID = ?
   ```

**Step 18: Log Customer ID Change**
1. Create audit record for customer ID change
   ```sql
   INSERT INTO ACTION_DETAIL
   (ACTION_AUDIT_ID, OLD_VALUE, NEW_VALUE, REASON_ID, FIELD_NAME)
   VALUES (?, ?, ?, ?, 'CUS_ID')
   ```

**Step 19: Complete Update Records**
1. Log subscription update history
   ```sql
   INSERT INTO SUB_UPDATE_INFOR
   (SUB_ID, ISDN, UPDATE_TIME, OLD_VALUE, NEW_VALUE, ID_NO_OLD, ID_NO_NEW, SYS_USER_NAME)
   VALUES (?, ?, SYSDATE, ?, ?, ?, ?, ?)
   ```

2. Create comprehensive action audit record
   ```sql
   INSERT INTO ACTION_AUDIT
   (ACTION_AUDIT_ID, OWNER_ID, ACTION_CODE, DEALER_CODE, REASON_ID,
    DESCRIPTION, IMAGE_ID, CUST_ID, STATUS, CREATE_DATE)
   VALUES (?, ?, ?, ?, ?, ?, ?, ?, 1, SYSDATE)
   ```

### 3.4. Transaction Management

**Transaction Annotation**

The service uses Spring's `@Transactional` annotation to ensure all database operations are performed atomically:

```java
@Transactional(transactionManager = "CMPreJpaTransactionManager")
public UpdateInfoCustomerResponse updateInfoCustomer(UpdateInfoCustomerRequest request, 
                                 List<UpdateInfoCustomerResponse> infoCustomerResponsesError) throws Exception
```

**Transaction Properties**

- **Isolation**: Default isolation level (READ_COMMITTED)
- **Propagation**: REQUIRED (Uses existing transaction if available, otherwise creates a new one)
- **Rollback**: Automatic rollback on any Exception
- **Timeout**: Default timeout settings

**Transaction Boundaries**

1. Transaction begins when entering the `updateInfoCustomer` method
2. Multiple database operations occur within the transaction scope:
   - Reading subscriber data
   - Disabling old customer images
   - Creating new image records
   - Updating customer information
   - Creating audit records
3. Transaction commits automatically upon successful method completion
4. If any exception occurs, the transaction automatically rolls back all changes

This approach ensures that if any error occurs during the update process, all changes will be rolled back to prevent data inconsistency. All operations within the method either complete successfully together or are rolled back entirely to maintain data integrity.

## 4. Example Usage
