# CustomerChargeController Flow Documentation

## Overview
The CustomerChargeController handles two main workflows:
1. **Create Flow**: Creating customer charges for optimization sessions
2. **Upload Flow**: Uploading customer charge files (CSV) for processing

---

## 1. Create Customer Charges Flow

### 1.1 Single Instance Create Flow

```mermaid
flowchart TD
    A[CreateConfirm GET] --> B{Permission Check}
    B -->|Fail| C[Redirect to Home]
    B -->|Pass| D[Get AWS Custom Fields]
    D --> E{Validate AWS Setup}
    E -->|Invalid| F[Set Error Alert]
    E -->|Valid| G[Create Model with Session Data]
    G --> H{Validate Rev Product Setup}
    H -->|Invalid| I[Set Product Error Alert]
    H -->|Valid| J[Show Create Confirm View]
    
    J --> K[Create POST Action]
    K --> L{Permission Check}
    L -->|Fail| M[Redirect to Home]
    L -->|Pass| N[Get AWS Credentials]
    N --> O{Validate AWS Credentials}
    O -->|Invalid| P[Set Error & Return to Confirm]
    O -->|Valid| Q[EnqueueCreateCustomerChargesSqs]
    Q --> R{SQS Success?}
    R -->|Fail| S[Set Error Alert & Return]
    R -->|Success| T[Set Success Alert]
    T --> U[Redirect to CustomerChargeConfirm]
```

### 1.2 Session-Based Create Flow

```mermaid
flowchart TD
    A[CreateConfirmSession GET] --> B{Permission Check}
    B -->|Fail| C[Redirect to Home]
    B -->|Pass| D[Get AWS Custom Fields]
    D --> E{Validate AWS Setup}
    E -->|Invalid| F[Set AWS Error Alert]
    E -->|Valid| G[Create Session Model]
    G --> H{Validate Rev Products}
    H -->|Invalid| I[Set Product Error Alert]
    H -->|Valid| J{Single Customer?}
    J -->|Yes| K[Show Single Create Confirm]
    J -->|No| L[Show Session Create Confirm]
    
    L --> M[CreateConfirmSession POST]
    M --> N{Permission Check}
    N -->|Fail| O[Redirect to Home]
    N -->|Pass| P[Parse Push Type]
    P --> Q{Valid Push Type?}
    Q -->|Invalid| R[Set Push Type Error]
    Q -->|Valid| S{Push Type Check}
    
    S -->|Charges/Both| T[Process Charges Flow]
    S -->|Usage/Both| U[Process Usage Flow]
    
    T --> V[Get AWS Credentials]
    V --> W{Validate AWS}
    W -->|Invalid| X[Set AWS Error]
    W -->|Valid| Y[Parse Selected Instances]
    Y --> Z{Valid Instances?}
    Z -->|Invalid| AA[Set Instance Error]
    Z -->|Valid| BB{Push Type: CDR?}
    BB -->|Yes| CC[InitializeUploadCustomerChargeCDRs]
    BB -->|No| DD[Loop Through Instances]
    DD --> EE[EnqueueCreateCustomerChargesWithSessionSqs]
    
    U --> FF[Get Rev FTP Credentials]
    FF --> GG{Validate FTP Setup}
    GG -->|Invalid| HH[Set FTP Error]
    GG -->|Valid| II[Parse Usage Instances]
    II --> JJ{Valid Usage Instances?}
    JJ -->|Invalid| KK[Set Usage Instance Error]
    JJ -->|Valid| LL[Create Usage Data]
    LL --> MM[Generate CSV Content]
    MM --> NN[Upload to Rev FTP]
    NN --> OO[Update IsPushed Flag]
    
    CC --> PP{Any Errors?}
    EE --> PP
    OO --> PP
    PP -->|Yes| QQ[Set Error Alert & Redirect]
    PP -->|No| RR[Set Success Alert & Redirect]
```

---

## 2. Upload Customer Charges Flow

### 2.1 Upload Confirmation Flow

```mermaid
flowchart TD
    A[UploadConfirm] --> B{Permission Check}
    B -->|Fail| C[Redirect to Home]
    B -->|Pass| D[Determine Site Type]
    D --> E[Get AWS Custom Fields]
    E --> F{Validate AWS Setup}
    F -->|Invalid| G[Set AWS Error Alert]
    F -->|Valid| H[Create Detail Model]
    H --> I[Show Upload Confirm View]
```

### 2.2 File Upload Processing Flow

```mermaid
flowchart TD
    A[Upload POST] --> B{Permission Check}
    B -->|Fail| C[Redirect to Home]
    B -->|Pass| D{File Exists?}
    D -->|No| E[Return OK]
    D -->|Yes| F{Valid Filename?}
    F -->|No| G[Return Error Message]
    F -->|Yes| H{CSV File?}
    H -->|No| I[Return Format Error]
    H -->|Yes| J{Filename Length OK?}
    J -->|No| K[Return Length Error]
    J -->|Yes| L{File Already Exists?}
    L -->|Yes| M[Return Duplicate Error]
    L -->|No| N[Parse CSV File]
    
    N --> O{Valid CSV Data?}
    O -->|No| P[Return Parse Error]
    O -->|Yes| Q{Has Valid Lines?}
    Q -->|No| R[Return No Data Error]
    Q -->|Yes| S[Get Rev Integration Auth]
    S --> T{Valid Rev Credentials?}
    T -->|No| U[Return Credentials Error]
    T -->|Yes| V[Validate Rev Service Numbers]
    V --> W{Valid Service Numbers?}
    W -->|No| X[Return Service Number Error]
    W -->|Yes| Y[Validate Rev Product Types]
    Y --> Z{Valid Product Types?}
    Z -->|No| AA[Return Product Type Error]
    Z -->|Yes| BB[Upload File to AWS S3]
    
    BB --> CC{S3 Upload Success?}
    CC -->|No| DD[Return S3 Error]
    CC -->|Yes| EE[Save App File Record]
    EE --> FF[Process M2M Lines]
    FF --> GG[Process Mobility Lines]
    GG --> HH[Create Upload File Record]
    HH --> II[Map Queue Entries]
    II --> JJ[Begin Database Transaction]
    JJ --> KK[Save Queue Entries to DB]
    KK --> LL[Enqueue to SQS]
    LL --> MM{SQS Success?}
    MM -->|No| NN[Rollback & Return Error]
    MM -->|Yes| OO[Commit Transaction]
    OO --> PP[Set Success Alert]
    PP --> QQ[Return OK]
```

---

## 3. Key Components and Helper Methods

### 3.1 SQS Message Queuing

#### EnqueueCreateCustomerChargesSqs
- Creates SQS message for single instance processing
- Includes instance metadata and authentication details
- Returns success/error status

#### EnqueueCreateCustomerChargesWithSessionSqs
- Similar to above but with session-specific handling
- Includes delay for last instance (90 seconds)
- Handles multiple instance processing

#### EnqueueUploadCustomerChargeCDRs
- Processes CDR (Call Detail Record) customer charges
- Uses retry policy for reliability
- Handles portal type differentiation

### 3.2 File Processing

#### CSV Validation Steps:
1. File format validation (.csv)
2. Filename uniqueness check
3. Content parsing with CsvHelper
4. Data validation (positive charges only)
5. Rev.IO service number validation
6. Product type validation

#### AWS S3 Integration:
1. Upload original file to S3
2. Store S3 reference in database
3. Use for audit and backup purposes

### 3.3 Database Operations

#### Transaction Flow:
1. Begin database transaction
2. Create queue entries for processing
3. Enqueue SQS messages
4. Commit on success or rollback on failure

#### Queue Entry Types:
- `OptimizationDeviceResult_CustomerChargeQueue` (M2M devices)
- `OptimizationMobilityDeviceResult_CustomerChargeQueue` (Mobility devices)

---

## 4. Error Handling

### Common Error Scenarios:
1. **Permission Errors**: User lacks required module access
2. **AWS Setup Errors**: Missing or invalid AWS credentials
3. **Rev.IO Setup Errors**: Missing product types or integration auth
4. **File Validation Errors**: Invalid format, duplicate names, parsing issues
5. **Database Errors**: Transaction failures, constraint violations
6. **SQS Errors**: Queue not found, message send failures
7. **FTP Errors**: Invalid credentials or connection issues

### Error Response Pattern:
- Set error message in session alert
- Set alert type to "danger"
- Redirect to appropriate view or return error response
- Log errors for debugging

---

## 5. Security and Permissions

### Required Permissions:
- **CustomerCharge Module**: Base access for all operations
- **RevCustomers Module**: Required for Rev.IO operations
- **M2M/Mobility Modules**: Required for device editing

### Site Type Logic:
- Defaults to Rev if user has RevCustomers access
- Falls back to AMOP if RevCustomers access denied
- Affects data filtering and processing logic

---

## 6. Integration Points

### External Systems:
1. **AWS SQS**: Message queuing for async processing
2. **AWS S3**: File storage and backup
3. **Rev.IO**: Customer billing integration
4. **Rev FTP**: Usage data upload
5. **Database**: Transaction management and data persistence

### Custom Fields Required:
- AWS Access Key
- AWS Secret Access Key
- Customer Charge Queue Name
- Create Customer Charge Queue Name
- S3 Bucket Name
- Rev FTP credentials (Host, Username, Password, Path)