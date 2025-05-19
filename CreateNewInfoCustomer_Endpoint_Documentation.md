# Detailed Documentation: createNewInfoCustomer SOAP Web Service

## 1. Overview

The `createNewInfoCustomer` function is a SOAP web service endpoint provided in the MDealerEndpoint class of the mDealer application. It allows dealers to register new customer information associated with a mobile phone number (ISDN) or SIM card serial. This web service is responsible for collecting, validating, and storing customer information including personal details and identification documents.

## 2. Function Definition

```java
@PayloadRoot(namespace = NAMESPACE_URI, localPart = "createNewInfoCustomer")
@ResponsePayload
public CreateNewInfoCustomerResponse createNewInfoCustomer(@RequestPayload CreateNewInfoCustomerRequest request) throws Exception
```

### 2.1 Annotations

- **@PayloadRoot**: Identifies the SOAP message this method will handle
  - **namespace**: The XML namespace URI (defined as a constant `NAMESPACE_URI` in the class)
  - **localPart**: The name of the root element of the SOAP request "createNewInfoCustomer"
- **@ResponsePayload**: Indicates the return value will be mapped to the response payload
- **@RequestPayload**: Indicates the parameter will be mapped from the request payload

## 3. Request Parameters

The `CreateNewInfoCustomerRequest` class contains the following fields:

| Parameter | Type | Description | Required |
|-----------|------|-------------|---------|
| serial | String | SIM card serial number | Optional (Either serial or isdn must be provided) |
| isdn | String | Mobile phone number | Optional (Either serial or isdn must be provided) |
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
| token | String | Authentication token | Required |
| transactionId | String | Unique transaction identifier | Optional |

## 4. Service Flow and Logic

The service performs the following steps:

### 4.1. Input Validation

**Step 1: Basic Parameter Validation**
1. Validate that either `serial` or `isdn` is provided (Error: error.empty.serial)
2. If serial is provided, ensure only one serial is being processed (Error: only.accept.scan.1.serial)
3. Validate dealer's phone number is provided (Error: error.empty.account)
4. Validate customer name is provided (Error: error.empty.name)

**Step 2: ID Document Validation**
1. Validate ID document type is provided (Error: error.empty.id.type)
2. Ensure ID type is not "Others" (Error: error.unallow.id.type)
3. For passport ID types, apply special formatting to the ID number
4. Validate ID number is provided (Error: error.empty.id.no)

**Step 3: Personal Information Validation**
1. Validate date of birth is provided (Error: error.empty.dob)
2. Ensure date of birth is in valid format (DD/MM/YYYY) (Error: error.invalid.dob)
3. Validate gender is provided (Error: error.empty.sex)

**Step 4: Security and Document Validation**
1. Validate authentication token is provided (Error: error.empty.token)
2. Ensure at least 2 identification images are provided (Error: error.empty.image)

### 4.2. Business Logic Validation

**Step 5: Format and Normalize Data**
1. Format the subscriber phone number (ISDN) to standardized format
   - Removes any non-numeric characters and applies standard phone number format
   - Ensures ISDN follows the required format for the system
2. Format the dealer phone number to standardized format 
   - Similar standardization for dealer phone number
   - Error occurs if format is invalid after standardization
3. Apply special formatting to passport numbers if ID type is passport
   - For passports (idType = '3'), special formatting rules are applied
   - Without formatting, subsequent validation may fail

**Step 6: SIM Card/ISDN Verification**
1. Check if the ISDN is valid and active by querying the subscriber database (Error: error.isdn.actived)
   - Error occurs if query returns empty results (ISDN/serial not found)
   - For ISDN check: STATUS = 2 means the subscription is active
   - For serial check: ACT_STATUS = '03' indicates "not active" which is required for registration
   ```sql
   -- If only ISDN is provided:
   SELECT * FROM CM_PRE2.SUB_MB WHERE ISDN = ? AND STATUS = 2
   
   -- If serial is provided:
   SELECT * FROM CM_PRE2.SUB_MB WHERE SERIAL = ? AND STATUS = 2 AND ACT_STATUS = '03'
   ```
2. Verify if customer images are already uploaded for this ISDN/serial (Error: error.upload.already)
   - Error occurs if COUNT(*) > 0 (images already exist for this ISDN/serial)
   - Prevents duplicate registration attempts
   ```sql
   SELECT COUNT(*) FROM CUST_IMAGE WHERE (SERIAL = ? OR ISDN = ?)
   ```

**Step 7: Dealer Account Validation**
1. Check if dealer account exists and is active in the system (Error: error.not.account.agent)
   - Error occurs if no records are found (dealer account doesn't exist)
   - STATUS = 1 indicates an active dealer account
   ```sql
   SELECT * FROM BCCS_IM.ACCOUNT_AGENT WHERE STATUS = 1 AND ISDN = ?
   ```
2. If dealer is staff type (owner_type = 1), validate the staff account is active (Error: error.account.agent.invalid)
   - Error occurs if staff account is not active or doesn't exist
   - STATUS = 1 indicates an active staff account
   ```sql
   SELECT * FROM BCCS_IM.STAFF WHERE STATUS = 1 AND STAFF_ID = ?
   ```
3. If dealer is shop type (owner_type = 2), validate the shop account is active (Error: error.account.agent.not.exist)
   - Error occurs if shop account is not active or doesn't exist
   - STATUS = 1 indicates an active shop
   ```sql
   SELECT * FROM BCCS_IM.SHOP WHERE STATUS = 1 AND SHOP_ID = ?
   ```

**Step 8: Access Control Check**
1. Verify dealer is allowed to register the given ISDN by checking channel permissions (Error: error.dealer.not.allow.register)
   - Error occurs if result = 1 (dealer not authorized for this ISDN/serial)
   - Checks if the serial belongs to the dealer's assigned channel
   - Parameters: serial, owner_id (from account_agent), dealer_isdn
   ```sql
   -- Via stored procedure
   CALL CHECK_ISDN_BELONG_CHANNEL(?, ?, ?)
   ```

**Step 9: Daily Limits Verification**
1. Check if dealer has reached daily image upload limit (Error: error.register.dealer.over)
   - Error occurs if COUNT(*) exceeds the dealer's daily upload limit (typically 50)
   - Limits number of customer registrations per dealer per day
   ```sql
   SELECT COUNT(*) FROM CUST_IMAGE 
   WHERE CREATE_DATE >= TRUNC(SYSDATE) 
   AND CREATE_DATE < TRUNC(SYSDATE) + 1
   AND DEALER_ISDN = ?
   ```
2. Check if sales point has reached daily upload limit (Error: error.register.salepoint.over)
   - Error occurs if COUNT(*) exceeds the sales point's daily upload limit (typically 200)
   - Limits total registrations per sales point per day
   ```sql
   -- Uses same query as above but checks against sales point limit
   SELECT COUNT(*) FROM CUST_IMAGE 
   WHERE CREATE_DATE >= TRUNC(SYSDATE) 
   AND CREATE_DATE < TRUNC(SYSDATE) + 1
   AND SALE_POINT = ?
   ```
3. Verify number of registrations for the given ID number (Error: error_unallow_register_info)
   - Error occurs if COUNT(*) exceeds the limit for this ID (typically 3)
   - Prevents one ID from being used for too many subscriptions
   ```sql
   SELECT COUNT(1) FROM CM_PRE2.SUB_MB a 
   INNER JOIN CM_PRE2.CUSTOMER b ON a.CUST_ID = b.CUST_ID 
   WHERE (a.SUBCOS_HUAWEI IS NULL OR a.SUBCOS_HUAWEI = ?) 
   AND a.STATUS = 2 
   AND LOWER(b.ID_NO) = LOWER(?)
   ```
4. Check number of information updates for today (Error: error_unallow_register_info_in_day)
   - Error occurs if result exceeds the daily limit for information updates
   - Limits frequency of customer data changes per day
   ```sql
   -- Via stored procedure
   CALL CHECK_INFO_SUBS(?, ?, ?, ?, ?, ?, ?)
   ```

### 4.3. Processing

**Step 10: OCR Detection Handling**
1. If OCR detection is disabled, forwards registration request to Customer Care system
2. Forwards request with all customer details and images to CC system
3. Returns the response from CC directly to client

**Step 11: Image Processing**
1. Generate unique sequence IDs for customer images and audit records
2. Create new customer image record in database with customer and dealer information
3. Upload each image to FTP server with proper naming conventions
4. Record image details in database with references to original images

**Step 12: Customer Data Management**
1. Check if customer with provided ID number already exists in database
   ```sql
   SELECT * FROM CUSTOMER WHERE ID_NO = ?
   ```
2. If customer exists, update their information with new details
   ```sql
   UPDATE CUSTOMER 
   SET NAME = ?, SEX = ?, PROVINCE = ?, ADDRESS = ?, ID_TYPE = ?, ID_NO = ?, BIRTHDAY = ?, EXPIRE_DATE = ? 
   WHERE CUST_ID = ?
   ```
3. If customer doesn't exist, create a new customer record
   ```sql
   INSERT INTO CUSTOMER 
   (CUST_ID, NAME, BIRTHDAY, SEX, ID_NO, ID_TYPE, PROVINCE, ADDRESS, EXPIRE_DATE) 
   VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
   ```
4. Link the customer ID with subscriber information in SUB_MB table
   ```sql
   UPDATE SUB_MB 
   SET CUST_ID = ?, NAME = ?, BIRTHDAY = ?, SEX = ?, PROVINCE = ?, ADDRESS = ? 
   WHERE SUB_ID = ?
   ```
5. Update customer image record with final customer ID
   ```sql
   UPDATE CUST_IMAGE SET CUST_ID = ? WHERE ID = ?
   ```

**Step 13: System Integration**
1. Record all actions in audit tables for compliance and tracking
   ```sql
   INSERT INTO ACTION_AUDIT 
   (ACTION_AUDIT_ID, OWNER_ID, ACTION_CODE, OWNER_CODE, REASON_ID, DESCRIPTION, REF_ID, REF_ID1, ACTION_TYPE) 
   VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
   ```
   
   ```sql
   INSERT INTO LOG_ACTION_DETAIL
   (LOG_ACTION_DETAIL_ID, PARAM_VALUE, REASON_ID, ACTION_AUDIT_ID, PARAM_NAME) 
   VALUES (?, ?, ?, ?, ?)
   ```
2. Reconnect to telecom exchange system if configured
3. Activate the subscriber in billing system
   ```sql
   -- Call to provisioning system APIs to activate the subscriber
   ```
4. Update Home Location Register (HLR) with subscriber information
   ```sql
   -- Call to HLR system APIs to update subscriber records
   ```

### 4.4. Response Generation

**Step 14: Generate Response**
1. For successful registration: Return success message with registration status
   ```xml
   <commonOut>
     <errorCode>0</errorCode>
     <errorMessage>Success</errorMessage>
     <msgType>0</msgType>
   </commonOut>
   ```
2. For validation errors: Return appropriate error code and message
   ```xml
   <commonOut>
     <errorCode>1</errorCode>
     <errorMessage>[Specific validation error]</errorMessage>
     <msgType>1</msgType>
   </commonOut>
   ```
3. For business rule violations: Return specific error explaining the violation
4. For processing failures: Return general error about the processing failure

### 4.5. Transaction Management

**Step 15: Ensure Data Consistency**
1. All database operations are executed within a transaction boundary
2. If any step fails during processing, the entire transaction is rolled back
3. Error details are logged in the system for troubleshooting
4. Proper exception handling ensures the client receives meaningful error messages



## 9. Example Usage

### Request Example:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <ns2:createNewInfoCustomer xmlns:ns2="http://ws.cm.bccs.viettel.com/">
      <serial>89855315000123456789</serial>
      <isdn>1234567890</isdn>
      <dealerIsdn>0987654321</dealerIsdn>
      <idType>1</idType>
      <name>John Doe</name>
      <idNo>123456789</idNo>
      <birthDay>01/01/1980</birthDay>
      <expireDate>01/01/2030</expireDate>
      <address>123 Main St, City</address>
      <province>Phnom Penh</province>
      <sex>1</sex>
      <isDetect>1</isDetect>
      <lstImage>
        <image>
          <data>Base64EncodedImageData1</data>
          <name>FRONT</name>
        </image>
        <image>
          <data>Base64EncodedImageData2</data>
          <name>BACK</name>
        </image>
      </lstImage>
      <action>REGISTER</action>
      <token>Auth-Token-123456</token>
      <transactionId>TRX12345678</transactionId>
    </ns2:createNewInfoCustomer>
  </soap:Body>
</soap:Envelope>
```

### Successful Response Example:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <ns2:createNewInfoCustomerResponse xmlns:ns2="http://ws.bccs.viettel.com/">
      <return>
        <commonOut>
          <errorCode>0</errorCode>
          <errorMessage>Success</errorMessage>
          <msgType>0</msgType>
        </commonOut>
        <data>
          <string>REGISTRATION_SUCCESS</string>
        </data>
        <transId>TRX12345678</transId>
        <requireOtp>0</requireOtp>
        <isRegister>1</isRegister>
        <listIsdnTrue>1234567890</listIsdnTrue>
      </return>
    </ns2:createNewInfoCustomerResponse>
  </soap:Body>
</soap:Envelope>
```

### Error Response Example:

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <ns2:createNewInfoCustomerResponse xmlns:ns2="http://ws.bccs.viettel.com/">
      <return>
        <commonOut>
          <errorCode>1</errorCode>
          <errorMessage>Customer images already uploaded</errorMessage>
          <msgType>1</msgType>
        </commonOut>
        <transId>TRX12345678</transId>
      </return>
    </ns2:createNewInfoCustomerResponse>
  </soap:Body>
</soap:Envelope>
```

## 10. Security Considerations

1. **Authentication**: Requires a valid token for access
2. **Authorization**: Verifies dealer permissions and channel access
3. **Input Validation**: Comprehensive validation of all input parameters
4. **Rate Limiting**: Enforces daily limits on registrations and uploads
5. **Data Protection**: Securely stores customer identification information
6. **Access Control**: Ensures dealers can only register customers for their authorized channels
7. **Audit Logging**: Records all operations for security and compliance

## 11. Performance Considerations

1. **Image Handling**: Efficient processing of customer identification images
2. **Database Operations**: Optimized queries and stored procedures
3. **Transaction Management**: Proper transaction boundaries for data consistency
4. **Connection Pooling**: Efficient use of database connections
5. **Error Handling**: Quick failure response when validation checks fail
6. **Logging**: Performance-sensitive logging with timing information
7. **External System Integration**: Efficient communication with provisioning systems