# WorkDrive Folder Sync on Lead-to-Deal Conversion

## What this does
Workflow-triggered function that fires automatically when a Lead
is converted to a Deal. It checks whether a WorkDrive folder
already exists, renames and moves it to the Deals parent folder
if it does, or creates a brand new folder if it does not — then
updates the CRM Deal record with the correct folder ID.

## Problem it solved
When leads were converted to deals, their WorkDrive folders
remained in the Leads directory with the old lead name. Project
managers had to manually find, rename, and move folders — or
create new ones — every time a conversion happened, causing
folders to go missing during handover.

## Outcome
- Folder renamed and relocated automatically on every conversion
- CRM Deal updated with correct folder ID immediately
- Zero orphaned folders in the Leads directory
- Consistent folder naming convention enforced across all deals

## Tech used
Deluge Scripting · Zoho CRM · Zoho WorkDrive · REST API ·
PATCH requests · Workflow Automation

## Code

```javascript
// Function: Sync WorkDrive Folder on Lead-to-Deal Conversion
// Purpose: On conversion, renames existing WorkDrive folder
// to match new Deal name and number, moves it to the Deals
// parent folder, or creates a new folder if none exists.
// Updates CRM Deal with folder ID.

try
{
    resp               = zoho.crm.getRecordById("Deals", Deal_ID);
    workdrive_folder_id = resp.get("Workdrive_Folder_Id");
    contact_name       = ifnull(resp.get("Contact_Name"), "").get("name");
    Deal_Number        = resp.get("Deal_Number");
    folder_name        = contact_name + " / " + Deal_Number;
    parentFolder       = "[PARENT_FOLDER_ID]";

    if(workdrive_folder_id != null)
    {
        // Rename existing folder
        attributeMap = Map();
        attributeMap.put("name", folder_name);
        payLoad = Map();
        payLoad.put("attributes", attributeMap);
        payLoad.put("type",       "files");
        data = Map();
        data.put("data", payLoad);

        headersMap = Map();
        headersMap.put("Accept",       "application/vnd.api+json");
        headersMap.put("Content-Type", "application/json;charset=UTF-8");

        invokeurl
        [
            url: "https://workdrive.zoho.com/api/v1/files/" + workdrive_folder_id
            type: PATCH
            parameters: data.toString()
            headers: headersMap
            connection: "zoho_workdrive"
        ];

        // Move folder to Deals parent
        attributeMap = Map();
        attributeMap.put("parent_id", parentFolder);
        payLoad = Map();
        payLoad.put("attributes", attributeMap);
        payLoad.put("id",         workdrive_folder_id);
        payLoad.put("type",       "files");
        list_A = list();
        list_A.add(payLoad);
        data = Map();
        data.put("data", list_A);

        invokeurl
        [
            url: "https://workdrive.zoho.com/api/v1/files"
            type: PATCH
            parameters: data.toString()
            headers: headersMap
            connection: "zoho_workdrive"
        ];
    }
    else
    {
        // Create new folder
        resp         = zoho.workdrive.createFolder(folder_name, parentFolder, "zoho_workdrive");
        Get_Folder_Id = resp.get("data").get("id");

        mp = Map();
        mp.put("Workdrive_Folder_Id", Get_Folder_Id);
        zoho.crm.updateRecord("Deals", Deal_ID, mp);
    }
}
catch(e)
{
    sendmail
    [
        from:    zoho.adminuserid
        to:      "[ERP_ALERT_EMAIL]"
        subject: "WorkDrive Sync Error / Deal ID: " + Deal_ID
        message: e
    ]
}
```

## Skills demonstrated
- Workflow-triggered automation on CRM record conversion
- REST API PATCH requests for file system operations
- Conditional create-or-update logic
- Consistent naming convention enforcement via automation
- Error handling with structured email alerting
- Production deployment in live lead management pipeline
