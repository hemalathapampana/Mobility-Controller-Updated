# UpdateRequest ServiceCharacteristic Analysis

## Request Flow and Initialization

Based on the code analysis, here's how the `updateRequest.ServiceCharacteristic` is initialized and flows through the system:

### 1. Request Creation in BuildUpdateICCIDorIMEI Method

**Location**: `MobilityController.cs`, lines 3743-3781

The `ServiceCharacteristic` collection is created in the `CreateUpdateICCIDorIMEIChangeRequest` method:

```csharp
private static string CreateUpdateICCIDorIMEIChangeRequest(string iccid, string imei, MobilityDevice device, string zipCode, string reasonCode, string technologyType, BulkchangeUpdateICCIDorIMEI model)
{
    var characteristicList = new List<ServiceCharacteristic>
    {
        new ServiceCharacteristic
        {
            Name = "reasonCode",
            Value = reasonCode
        },
        new ServiceCharacteristic
        {
            Name = "serviceZipCode", 
            Value = zipCode
        },
        new ServiceCharacteristic
        {
            Name = "technologyType",
            Value = string.IsNullOrEmpty(technologyType) ? Resources.CommonStrings.MobiltiyTechnologyTypeDefault : technologyType
        },
        new ServiceCharacteristic
        {
            Name = "IMEI",
            Value = string.IsNullOrEmpty(imei) ? device.IMEI : imei  // THIS IS WHERE OLD IMEI COMES FROM
        },
        new ServiceCharacteristic
        {
            Name = "sim",
            Value = string.IsNullOrEmpty(iccid) ? device.ICCID : iccid  // THIS IS WHERE SIM/ICCID COMES FROM
        }
    };
    
    var changeEquipmentRequest = new TelegenceUpdateICCIDorIMEIRequest()
    {
        ServiceCharacteristic = characteristicList
    };
    
    var request = new BulkChangeStatusUpdateRequest<TelegenceUpdateICCIDorIMEIRequest>
    {
        Request = changeEquipmentRequest
    };
    
    // Request is serialized and stored
    return JsonConvert.SerializeObject(request);
}
```

### 2. Key Initialization Points

#### IMEI Value Initialization:
```csharp
Name = "IMEI",
Value = string.IsNullOrEmpty(imei) ? device.IMEI : imei
```
- **Source**: `device.IMEI` (from the database device record)
- **Logic**: If no new IMEI provided, uses the existing device IMEI
- **Issue**: This `device.IMEI` comes from the device found by the lookup method

#### SIM/ICCID Value Initialization:
```csharp
Name = "sim", 
Value = string.IsNullOrEmpty(iccid) ? device.ICCID : iccid
```
- **Source**: `device.ICCID` (from the database device record) or new `iccid` parameter
- **Logic**: If no new ICCID provided, uses the existing device ICCID

### 3. Parameter Flow to CreateUpdateICCIDorIMEIChangeRequest

The method is called from `BuildUpdateICCIDorIMEI` with these parameters:

```csharp
// Line 3730 in BuildUpdateICCIDorIMEI
deviceChanges.Add(new Mobility_DeviceChange(
    CreateUpdateICCIDorIMEIChangeRequest(
        newICCID,           // From model.NewICCIDs[index]
        newIMEI,            // From model.NewIMEIs[index] 
        device,             // Device found by lookup (CRITICAL!)
        device.ServiceZipCode,
        CommonStrings.UpdateICCIDorIMEIReasonCode,
        device.TechnologyType,
        model
    ), 
    device.id, 
    device.ICCID, 
    phoneNumber, 
    createdBy
));
```

### 4. The Core Issue Explanation

**The Problem with Current Logic:**

1. **Device Lookup**: `BuildUpdateICCIDorIMEI` uses `GetDevicesByNumber()` to find devices by phone number
2. **IMEI Source**: The IMEI value in ServiceCharacteristic comes from `device.IMEI` of the device found by phone number
3. **Mismatch**: But the request contains ICCID values that should map to specific devices with specific IMEIs

**Where the Request Values Come From:**

```csharp
// In BuildUpdateICCIDorIMEI method (lines 3702-3710)
var newICCID = string.Empty;
if (model.NewICCIDs != null && model.NewICCIDs.Count > modelDevice.index)
{
    newICCID = model.NewICCIDs[modelDevice.index];  // NEW ICCID from request
}

var newIMEI = string.Empty; 
if (model.NewIMEIs != null && model.NewIMEIs.Count > modelDevice.index)
{
    newIMEI = model.NewIMEIs[modelDevice.index];    // NEW IMEI from request
}

// The OLD IMEI comes from the device record
string oldIMEI = device.IMEI;  // PROBLEM: This device may not correspond to the ICCID in the request
```

### 5. Lambda Processing Context

The lambda code you mentioned appears to be processing the serialized request:

```csharp
// Your lambda code processes the deserialized request
var newICCID = updateRequest.ServiceCharacteristic.FirstOrDefault(x => x.Name == "sim")?.Value;
var newIMEI = updateRequest.ServiceCharacteristic.Where(x => x.Name == "IMEI").Select(x => x.Value).FirstOrDefault();
```

**Where `updateRequest` comes from:**
- It's the deserialized version of the `BulkChangeStatusUpdateRequest<TelegenceUpdateICCIDorIMEIRequest>`
- Originally created by `CreateUpdateICCIDorIMEIChangeRequest` method
- Stored as JSON in the database and later processed by the lambda function

### 6. Corrected Flow

To fix the issue, the `BuildUpdateICCIDorIMEI` method should:

1. Use `GetDevicesByIccid()` instead of `GetDevicesByNumber()`
2. Ensure the `device.IMEI` used in ServiceCharacteristic corresponds to the device that actually has the ICCID from the request
3. This way, when the lambda processes `updateRequest.ServiceCharacteristic`, the IMEI value correctly corresponds to the SIM/ICCID value

### 7. Request Structure Summary

The `updateRequest.ServiceCharacteristic` contains:
- `"sim"` → ICCID value (old or new)
- `"IMEI"` → IMEI value (old or new) 
- `"reasonCode"` → Change reason
- `"serviceZipCode"` → Service zip code
- `"technologyType"` → Technology type

**Critical Point**: The IMEI value in the characteristic must come from the device that has the specific ICCID being processed, not from a device found by phone number lookup.