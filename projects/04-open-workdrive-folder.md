# Open WorkDrive Folder with Automatic Attachment Migration

## What this does
Button-triggered function that checks whether a WorkDrive folder
exists for a Deal, creates one if missing, migrates all CRM
attachments into a subfolder, marks the migration as complete
to prevent duplicate transfers, and opens the correct folder
in a popup window — with different behaviour for active deals
versus Closed Won deals.

## Problem it solved
Sales reps had to manually navigate to WorkDrive, locate the
correct deal folder, and move attachments across from CRM one
by one. Folders were inconsistently named and attachments were
frequently lost or duplicated during handover.

## Outcome
- Folder creation and attachment migration fully automated
- Attachment transfer checkbox prevents duplicate migrations
- Correct folder view served based on deal stage
- Zero navigation steps for sales reps — one button click

## Tech used
Deluge Scripting · Zoho CRM · Zoho WorkDrive · REST API ·
File Management Automation · Stage-based Conditional Logic

## Code

```javascript
// Function: Open WorkDrive Folder
// Purpose: Open existing WorkDrive folder for a Deal,
// migrate CRM attachments if not yet transferred,
// open correct folder view based on deal stage.

Deal_details = zoho.crm.getRecordById("Deals", Deal_id);
workdrive_id = Deal_details.get("Workdrive_Folder_Id");
deal_name = Deal_details.get("Deal_Name");
Deal_Number = Deal_details.get("Deal_Number");

if(workdrive_id != null)
{
    workdrive_folder_id = Deal_details.get("Workdrive_Folder_Id");
    stage_details = Deal_details.get("Stage");
    attachment_transferred = Deal_details.get("Attachment_transferred");

    if(attachment_transferred == false)
    {
        // Create subfolder for CRM attachments
        response = zoho.workdrive.createFolder("CRM Attachments", workdrive_folder_id, "zoho_workdrive");
        crm_attachment_id = response.get("data").get("id");

        // Fetch and migrate attachments from CRM Deal
        attachments = invokeurl
        [
            url: "https://www.zohoapis.com/crm/v2/Deals/" + Deal_id + "/Attachments"
            type: GET
            connection: "zoho_crm"
        ];

        for each attachment in attachments.get("data")
        {
            attachment_id = attachment.get("id");
            file_name = attachment.get("File_Name").replaceAll("%", "");
            file = invokeurl
            [
                url: "https://www.zohoapis.com/crm/v2/Deals/" + Deal_id + "/Attachments/" + attachment_id
                type: GET
                connection: "zoho_crm"
            ];

            checkFile = file.toString();
            if(checkFile.contains("DOWNLOAD_NOT_ALLOWED") == false || checkFile == null)
            {
                to_workdrive = zoho.workdrive.uploadFile(file, crm_attachment_id, file_name, true, "zoho_workdrive");
            }
        }

        // Mark attachments as transferred
        checkbox_Value = Map();
        checkbox_Value.put("Attachment_transferred", true);
        update_field = zoho.crm.updateRecord("Deals", Deal_id, checkbox_Value);
    }

    // Open appropriate folder based on Deal stage
    if((workdrive_folder_id != null || workdrive_folder_id != "") && stage_details != "Closed Won")
    {
        openUrl("https://workdrive.zoho.com/[WORKSPACE_URL]/folders/" + workdrive_folder_id, "popup window", "height=800,width=1000");
        Message_out = "Opening Folder";
    }
    else if(stage_details == "Closed Won")
    {
        openUrl("https://workdrive.zoho.com/[WORKSPACE_URL]/folders/" + workdrive_folder_id + "/files", "popup window", "height=800,width=1000");
        Message_out = "Opening Team Folder";
    }
    else
    {
        Message_out = "The folder does not exist on Zoho WorkDrive.";
    }
}
else
{
    // Create new folder if none exists
    response = zoho.workdrive.createFolder(deal_name + " / " + Deal_Number, "[PARENT_FOLDER_ID]", "zoho_workdrive");
    Deal_Folder_id = response.get("data").get("id");

    stage_details = Deal_details.get("Stage");
    attachment_transferred = Deal_details.get("Attachment_transferred");

    update_deal_details = Map();
    update_deal_details.put("Workdrive_Folder_Id", Deal_Folder_id);
    update_deal = zoho.crm.updateRecord("Deals", Deal_id, update_deal_details);

    workdrive_folder_id = Deal_Folder_id;

    if(attachment_transferred == false)
    {
        response = zoho.workdrive.createFolder("CRM Attachments", Deal_Folder_id, "zoho_workdrive");
        crm_attachment_id = response.get("data").get("id");

        attachments = invokeurl
        [
            url: "https://www.zohoapis.com/crm/v2/Deals/" + Deal_id + "/Attachments"
            type: GET
            connection: "zoho_crm"
        ];

        for each attachment in attachments.get("data")
        {
            attachment_id = attachment.get("id");
            file_name = attachment.get("File_Name").replaceAll("%", "");
            file = invokeurl
            [
                url: "https://www.zohoapis.com/crm/v2/Deals/" + Deal_id + "/Attachments/" + attachment_id
                type: GET
                connection: "zoho_crm"
            ];

            checkFile = file.toString();
            if(checkFile.contains("DOWNLOAD_NOT_ALLOWED") == false || checkFile == null)
            {
                to_workdrive = zoho.workdrive.uploadFile(file, crm_attachment_id, file_name, true, "zoho_workdrive");
            }
        }

        checkbox_Value = Map();
        checkbox_Value.put("Attachment_transferred", true);
        update_field = zoho.crm.updateRecord("Deals", Deal_id, checkbox_Value);
    }

    if(stage_details != "Closed Won")
    {
        openUrl("https://workdrive.zoho.com/[WORKSPACE_URL]/folders/" + workdrive_folder_id, "popup window", "height=800,width=1000");
        Message_out = "Opening Folder";
    }
    else if(stage_details == "Closed Won")
    {
        openUrl("https://workdrive.zoho.com/[WORKSPACE_URL]/folders/" + workdrive_folder_id + "/files", "popup window", "height=800,width=1000");
        Message_out = "Opening Team Folder";
    }
    else
    {
        Message_out = "The folder does not exist on Zoho WorkDrive.";
    }
}

return "";
```

## Skills demonstrated
- File system automation across CRM and cloud storage
- Idempotent design via checkbox guard on attachment transfer
- Stage-based conditional folder routing
- REST API file download and re-upload across platforms
- Null-safe field handling throughout
- Production deployment in live sales environment
