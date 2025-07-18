# BuildUpdateICCIDorIMEI Method Analysis and Corrected Logic

## Current Issue Analysis

### Request Source Investigation

Based on the code analysis, the request flow for the `BuildUpdateICCIDorIMEI` method is as follows:

1. **Request Entry Point**: `PostChangeICCIDorIMEI(BulkchangeUpdateICCIDorIMEI model)`
   - File: `MobilityController.cs`, line 514

2. **Model Structure**: `BulkchangeUpdateICCIDorIMEI` contains:
   - `model.Devices` - Collection of phone numbers (MSISDNs)
   - `model.NewICCIDs` - Collection of new ICCID values
   - `model.NewIMEIs` - Collection of new IMEI values
   - Index-based mapping between these collections

3. **Current Device Lookup**: 
   - Uses `GetDevicesByNumber()` method (line 3676)
   - Looks up devices by **phone number (MSISDN)**, not by ICCID or SIM
   - Returns devices based on `device.MSISDN` field

### Problem with Current Logic

**Issue Location**: Line 3713 in `BuildUpdateICCIDorIMEI` method
```csharp
string oldIMEI = device.IMEI;
```

**The Problem**: 
- The method retrieves devices by phone number (MSISDN)
- It then gets `oldIMEI` from the device record found by phone number
- However, for bulk ICCID/IMEI changes, the request contains ICCID values in the "sim" field
- The current logic doesn't map the ICCID from the request to the corresponding device's IMEI
- It assumes the device found by phone number has the correct old IMEI, but this may not match the ICCID-IMEI relationship in the request

## Corrected Logic for BuildUpdateICCIDorIMEI Method

### Updated Method Logic

```csharp
internal static IEnumerable<Mobility_DeviceChange> BuildUpdateICCIDorIMEI(AltaWorxCentral_Entities awxDb,
    HttpSessionStateBase session, PermissionManager permissionManager, BulkchangeUpdateICCIDorIMEI model)
{
    var createdBy = SessionHelper.GetAuditByName(session);
    
    // CORRECTED: Use GetDevicesByIccid instead of GetDevicesByNumber
    // since the request contains ICCID values that need to be mapped to devices
    var devicesByIccids = GetDevicesByIccid(awxDb, model.ServiceProviderId, model.Devices);
    var archivedICCIDs = CheckDevicesArchivedByIccid(awxDb, model.ServiceProviderId, model.Devices);
    var deviceChanges = new List<Mobility_DeviceChange>();

    // Load IMEI master table once outside the loop
    var imeiRangeList = awxDb.IMEI_DeviceType_CarrierRatePlan.Where(x => x.IsActive).ToList();

    foreach (var modelDevice in model.Devices.Select((item, index) => new { item, index }))
    {
        var currentICCID = modelDevice.item; // This is actually an ICCID value, not phone number
        
        if (!string.IsNullOrWhiteSpace(currentICCID))
        {
            // CORRECTED: Look up device by ICCID instead of phone number
            if (!devicesByIccids.ContainsKey(currentICCID) || !devicesByIccids.TryGetValue(currentICCID, out var device))
            {
                if (archivedICCIDs.Contains(currentICCID))
                {
                    deviceChanges.Add(CreateDeviceChangeError(currentICCID, 
                        string.Format(CommonStrings.MobilityDeviceICCIDIsArchivedError, currentICCID), createdBy));
                }
                else
                {
                    deviceChanges.Add(CreateDeviceChangeError(currentICCID, 
                        string.Format(CommonStrings.MobilityDeviceICCIDNotExistError, currentICCID), createdBy));
                }
            }
            else
            {
                var newICCID = string.Empty;
                if (model.NewICCIDs != null && model.NewICCIDs.Count > modelDevice.index)
                {
                    newICCID = model.NewICCIDs[modelDevice.index];
                }
                
                var newIMEI = string.Empty;
                if (model.NewIMEIs != null && model.NewIMEIs.Count > modelDevice.index)
                {
                    newIMEI = model.NewIMEIs[modelDevice.index];
                }

                // CORRECTED: Get oldIMEI from the device that matches the current ICCID
                // This ensures we get the IMEI that corresponds to the specific ICCID in the request
                string oldIMEI = device.IMEI;

                // Validate if oldIMEI and newIMEI have eSIM pattern
                bool oldIMEIIsESIM = IsIMEIeSIM(oldIMEI, imeiRangeList);
                bool newIMEIIsESIM = IsIMEIeSIM(newIMEI, imeiRangeList);

                // Business rule: if old is eSIM but new is not, reject
                if (oldIMEIIsESIM && !newIMEIIsESIM)
                {
                    deviceChanges.Add(CreateDeviceChangeError(device.MSISDN,
                        $"ICCID/IMEI swap failed. Device requires eSIM-compatible IMEI.",
                        createdBy));
                    continue;
                }

                deviceChanges.Add(new Mobility_DeviceChange(
                    CreateUpdateICCIDorIMEIChangeRequest(newICCID, newIMEI, device, device.ServiceZipCode, 
                        CommonStrings.UpdateICCIDorIMEIReasonCode, device.TechnologyType, model), 
                    device.id, 
                    device.ICCID, 
                    device.MSISDN, // Use the device's phone number for tracking
                    createdBy));
            }
        }
    }

    return deviceChanges;
}
```

### Key Changes Made

1. **Device Lookup Method**: Changed from `GetDevicesByNumber()` to `GetDevicesByIccid()`
   - Reason: The request contains ICCID values in the "sim" field, not phone numbers

2. **Variable Naming**: Changed `phoneNumber` to `currentICCID` to reflect the actual data type

3. **Device Retrieval**: Now looks up devices by ICCID to ensure the correct device-IMEI relationship

4. **Error Handling**: Updated error messages to handle ICCID-based lookups instead of phone number lookups

5. **Old IMEI Assignment**: The `oldIMEI = device.IMEI` now correctly gets the IMEI from the device that matches the specific ICCID from the request

### Why This Fixes the Issue

- **Correct Mapping**: Ensures that the oldIMEI corresponds to the device that has the specific ICCID mentioned in the request
- **Data Integrity**: Maintains the relationship between ICCID and IMEI as intended in the bulk change request
- **Consistent Logic**: Aligns with the request structure where "sim" values are ICCID identifiers

### Required Supporting Methods

The corrected logic assumes these helper methods exist (which they do based on the code analysis):
- `GetDevicesByIccid()` - Already exists at line 2134
- `CheckDevicesArchivedByIccid()` - Needs to be implemented if not existing
- `CreateDeviceChangeError()` - Already exists
- `CreateUpdateICCIDorIMEIChangeRequest()` - Already exists at line 3743