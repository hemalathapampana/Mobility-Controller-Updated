# CustomerChargeController - Simple Flow Documentation

## 1. Create Button Click Flow Table

| Step | Action | Cross Provider Check | Storage Location | Key Data |
|------|--------|---------------------|------------------|----------|
| 1 | User clicks "Create" | Check `OptimizationSettings.OptIntoCrossProviderCustomerOptimization` | Session State | Permission validation |
| 2 | Load CreateConfirm page | If cross-provider: Check `OptimizationSessions.ServiceProviderIds` | Memory | AWS credentials |
| 3 | Validate AWS setup | Cross-provider uses different queue logic | Custom Fields table | awsAccessKey, awsSecretAccessKey |
| 4 | Check Rev products | Cross-provider may skip Rev validation | Database cache | RevProductTypeId, RevProductId |
| 5 | User confirms create | Cross-provider: `portalTypeId = CrossProvider` | Form submission | selectedInstances |
| 6 | Process instances | Cross-provider: Create `CustomerChargeQueueToProcess` records | `CustomerChargeQueueToProcess` table | instanceIds, queueId |
| 7 | Send to SQS | Cross-provider uses CDR queue | AWS SQS | Queue message with metadata |
| 8 | Database commit | Cross-provider: Additional queue tracking | Transaction log | Success/failure status |

## 2. Upload Button Click Flow Table

| Step | Action | Storage Location | Validation | Data Stored |
|------|--------|------------------|------------|-------------|
| 1 | User selects CSV file | Browser memory | File extension check (.csv) | File object |
| 2 | Click "Upload" | Server memory | Filename uniqueness check | File stream |
| 3 | File validation | Temporary storage | CSV format validation | Parsed CSV data |
| 4 | Parse CSV content | Memory buffer | CsvHelper parsing | `CustomerChargeCsvRow` objects |
| 5 | Validate data | Memory collection | Positive charges only | Filtered charge records |
| 6 | Check Rev services | Database query | Service number validation | Valid service numbers |
| 7 | Upload to S3 | AWS S3 bucket | S3 upload success | S3 file reference |
| 8 | Save file record | `AppFile` table | Database constraints | File metadata |
| 9 | Create queue entries | `OptimizationDeviceResult_CustomerChargeQueue` | Business logic validation | Charge queue records |
| 10 | Send to SQS | AWS SQS queue | Queue availability | SQS message |
| 11 | Commit transaction | Database | Transaction integrity | All records saved |
| 12 | Success response | Session alert | User feedback | Success message |

## 3. Cross Provider Optimization Logic

### When Cross Provider is Enabled:
```
if (permissionManager.OptimizationSettings?.OptIntoCrossProviderCustomerOptimization)
{
    var optimizationSession = altaWrxDb.OptimizationSessions.Where(x => x.SessionId.Equals(sessionId)).FirstOrDefault();
    if (!string.IsNullOrWhiteSpace(optimizationSession.ServiceProviderIds))
    {
        isCrossProviderCustomerOptimization = true;
        portalTypeId = (int)PortalTypes.CrossProvider;
        // Use CDR customer charge processing
    }
}
```

### Storage Differences in Cross Provider:

| Regular Flow | Cross Provider Flow |
|--------------|-------------------|
| Direct SQS to Rev.IO | Create CDR queue records first |
| Single service provider | Multiple service providers |
| Standard portal type | `PortalTypes.CrossProvider` |
| Direct charge creation | Queue-based batch processing |
| Simple instance tracking | Complex queue management |

## 4. Database Storage Details

### Create Flow Storage:
- **CustomerChargeQueueToProcess**: Queue tracking for cross-provider
- **OptimizationSessions**: Session metadata and service provider IDs
- **SQS Messages**: AWS queue for async processing
- **Session State**: Temporary user selections

### Upload Flow Storage:
- **AppFile**: File metadata and S3 reference
- **CustomerCharge_UploadedFile**: Upload tracking
- **OptimizationDeviceResult_CustomerChargeQueue**: M2M device charges
- **OptimizationMobilityDeviceResult_CustomerChargeQueue**: Mobility device charges
- **AWS S3**: Physical file storage
- **SQS Queue**: Processing messages

## 5. Create Flow - 12 Key Points

1. **Permission Check**: Validate `ModuleEnum.CustomerCharge` access
2. **AWS Validation**: Check AWS credentials from custom fields
3. **Rev Product Check**: Validate `RevProductTypeId` and `RevProductId` setup
4. **Cross Provider Detection**: Check optimization settings and session data
5. **Instance Selection**: Parse and validate selected customer instances
6. **Queue Creation**: Create appropriate queue records (regular vs CDR)
7. **SQS Message**: Send processing message to AWS queue
8. **Portal Type**: Set correct portal type (regular vs CrossProvider)
9. **Delay Logic**: Add 90-second delay for last instance in batch
10. **Error Handling**: Comprehensive error catching and user feedback
11. **Transaction Management**: Database transaction for data integrity
12. **Success Redirect**: Navigate to confirmation page with success message

## 6. Upload Flow - 12 Key Points

1. **File Selection**: Validate file exists and has .csv extension
2. **Uniqueness Check**: Ensure filename hasn't been processed before
3. **CSV Parsing**: Use CsvHelper to parse file content into objects
4. **Data Filtering**: Remove zero or negative charge amounts
5. **Service Validation**: Check Rev.IO service numbers exist
6. **Product Validation**: Verify Rev.IO product type IDs are valid
7. **S3 Upload**: Store original file in AWS S3 bucket
8. **File Record**: Create AppFile record with S3 reference
9. **Queue Mapping**: Map CSV data to device charge queue entities
10. **Database Transaction**: Save all records in single transaction
11. **SQS Enqueue**: Send file processing message to AWS queue
12. **Success Response**: Return success status and user notification

## 7. Key Storage Tables Summary

| Table Name | Purpose | Key Fields |
|------------|---------|------------|
| `OptimizationSessions` | Session tracking | SessionId, ServiceProviderIds |
| `CustomerChargeQueueToProcess` | Cross-provider queue management | QueueId, InstanceIds |
| `AppFile` | File metadata | AmazonFileName, FileName, TenantId |
| `CustomerCharge_UploadedFile` | Upload tracking | FileName, IntegrationAuthenticationId |
| `OptimizationDeviceResult_CustomerChargeQueue` | M2M charges | RevProductTypeId, ChargeAmount, RevServiceNumber |
| `OptimizationMobilityDeviceResult_CustomerChargeQueue` | Mobility charges | RevProductTypeId, ChargeAmount, RevServiceNumber |

## 8. Cross Provider vs Regular Flow Comparison

| Aspect | Regular Flow | Cross Provider Flow |
|--------|--------------|-------------------|
| **Queue Type** | Direct SQS | CDR Queue + SQS |
| **Portal Type** | Service Provider specific | CrossProvider |
| **Processing** | Immediate | Batch with delay |
| **Tracking** | Simple instance tracking | Complex queue management |
| **Service Providers** | Single | Multiple |
| **Error Handling** | Standard | Enhanced for multiple providers |