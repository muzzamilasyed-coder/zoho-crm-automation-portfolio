# Sync Project Manager from Zoho Analytics to CRM

## What this does
Scheduled function that exports a reconciliation table from Zoho
Analytics via API, maps each row to a structured object comparing
the Project Manager assigned in Zoho Projects against the Project
Manager field in the CRM Deal, and automatically updates any
mismatches or missing values in CRM — keeping both systems in sync
without any manual intervention.

## Problem it solved
Project managers were being reassigned in Zoho Projects but the
change was never reflected in the CRM Deal record. This caused
reporting errors, incorrect commission calculations, and
miscommunication between sales and operations teams about who
owned each project.

## Outcome
- CRM Project Manager field kept in sync with Zoho Projects
  automatically on every scheduled run
- Missing PM assignments detected and filled programmatically
- Mismatch rate reduced to zero across all active deals
- Eliminated a recurring source of reporting and commission errors

## Tech used
Deluge Scripting · Zoho CRM · Zoho Analytics · REST API ·
Data Reconciliation · Scheduled Automation

## Code

```javascript
// Function: Sync Project Manager from Zoho Projects to CRM Deals
// Purpose: Exports reconciliation table from Zoho Analytics,
// compares Project Manager fields between Zoho Projects and
// CRM Deals, updates mismatches in CRM automatically.

email         = "[ANALYTICS_USER_EMAIL]";
workspaceName = "[ANALYTICS_WORKSPACE_NAME]";
tableName     = "[RECONCILIATION_TABLE_NAME]";

paramsMap = Map();
paramsMap.put("ZOHO_ACTION",        "EXPORT");
paramsMap.put("ZOHO_OUTPUT_FORMAT", "JSON");
paramsMap.put("ZOHO_ERROR_FORMAT",  "JSON");
paramsMap.put("ZOHO_API_VERSION",   "1.0");

resp     = invokeurl
[
    url: "https://analyticsapi.zoho.com/api/" + encodeUrl(email) + "/"
       + encodeUrl(workspaceName) + "/" + encodeUrl(tableName)
    type: POST
    parameters: paramsMap
    connection: "zoho_analytics"
];

mainResp = resp.get("response");

if(mainResp.containsKey("result"))
{
    Result    = mainResp.get("result").toMap();
    TableRows = Result.get("rows");

    simp = List();
    for each tblrow in TableRows
    {
        inMap = Map();
        inMap.put("Zoho Project ID",             tblrow.get(0));
        inMap.put("Deal ID",                     tblrow.get(1));
        inMap.put("Zoho Project Project Manager",tblrow.get(2));
        inMap.put("Zoho Project Manager CRM ID", tblrow.get(3));
        inMap.put("Deal Project Manager",        tblrow.get(4));
        inMap.put("CRM PM Zoho User ID",         tblrow.get(5));
        simp.add(inMap);
    }

    if(simp.size() > 0)
    {
        for each samplerow in simp
        {
            p_id          = samplerow.get("Zoho Project ID");
            crm_id        = samplerow.get("Deal ID");
            zp_pm_crm_id  = samplerow.get("Zoho Project Manager CRM ID");
            d_pm          = samplerow.get("Deal Project Manager");

            // Update CRM if PM is missing or mismatched
            if(d_pm == "" || zp_pm_crm_id != d_pm)
            {
                update_crm_map = Map();
                update_crm_map.put("Project_Manager", zp_pm_crm_id);
                zoho.crm.updateRecord("Deals", crm_id, update_crm_map);
            }
        }
    }
}
```

## Skills demonstrated
- Cross-platform data reconciliation via Analytics API export
- Structured row mapping from raw API response to objects
- Conditional update logic with three-way comparison
- Scheduled automation for ongoing data integrity
- Zero-touch synchronisation between two live systems
- Production deployment supporting commission and reporting accuracy
