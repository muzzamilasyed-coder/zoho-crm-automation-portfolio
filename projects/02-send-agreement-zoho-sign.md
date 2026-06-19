# Send Purchase Agreement via Zoho Sign

## What this does
Workflow-triggered function that pulls 40+ fields from a CRM Deal,
formats financial values, dynamically selects the correct Zoho Sign
template based on Lead Source and Co-Signer presence, sends the
agreement to all signing parties, transfers document ownership to
the sales rep, and updates the CRM with the new Document ID.

## Problem it solved
Sending purchase agreements required manually filling a document
template, selecting the correct signers, and tracking the document
ID back into CRM. With multiple lead sources and co-signer
combinations requiring different templates, errors were frequent
and the process took 20–30 minutes per deal.

## Outcome
- Agreement dispatched in seconds with zero manual field entry
- Correct template selected automatically across 4 routing conditions
- CRM updated with Document ID immediately on send
- Full error alerting via email if function fails

## Tech used
Deluge Scripting · Zoho CRM · Zoho Sign · REST API · Conditional
Template Routing

## Code

```javascript
// Function: Send New Purchase Agreement v2024
// Purpose: Workflow-triggered. Fetches Deal fields, routes to correct
// Zoho Sign template based on Lead Source and Co-Signer,
// sends agreement, transfers ownership to sales rep,
// updates CRM Agreement Document ID.

try
{
    DealRec    = zoho.crm.getRecordById("Deals", Deal_ID);
    LeadSource = DealRec.get("Lead_Source");

    Customer_Name  = "";
    CustomerEmail  = "";

    // Primary signer
    CustomerVal = ifnull(DealRec.get("Signing_Customer"), "");
    if(CustomerVal != "")
    {
        Customer_Name = CustomerVal.get("name");
        CustomerID    = CustomerVal.get("id");
        ContactRec    = zoho.crm.getRecordById("Contacts", CustomerID);
        CustomerEmail = ContactRec.get("Email");
    }

    // Co-signer
    CO_SignerName  = "";
    CO_SignerEmail = "";
    CO_SignerVal   = ifnull(DealRec.get("Co_Signer"), "");
    if(CO_SignerVal != "")
    {
        CO_SignerName  = CO_SignerVal.get("name");
        CO_SignerID    = CO_SignerVal.get("id");
        CO_SignerRec   = zoho.crm.getRecordById("Contacts", CO_SignerID);
        CO_SignerEmail = CO_SignerRec.get("Email");
    }

    // Address fields
    Project_Address = ifnull(DealRec.get("Project_Street"), "") + ", "
                    + ifnull(DealRec.get("Project_City"), "") + ", "
                    + ifnull(DealRec.get("Project_State"), "") + ", "
                    + ifnull(DealRec.get("Project_Zip_Code"), "");

    Home_Address = "";
    if(DealRec.get("Billing_Street") != null)
    {
        Home_Address = ifnull(DealRec.get("Billing_Street"), "") + ", "
                     + ifnull(DealRec.get("Billing_City"), "") + ", "
                     + ifnull(DealRec.get("Billing_State"), "") + ", "
                     + ifnull(DealRec.get("Billing_Zip_Code"), "");
    }

    // Financial formatting
    Total_Contract_Cost = ifnull(DealRec.get("Total_Contract_Cost"), "");
    if(Total_Contract_Cost != "")
    {
        frac = Total_Contract_Cost.frac();
        Total_Contract_Cost = Total_Contract_Cost.round(2).remove("-").toString()
                              .replaceAll("(?<!\.\d)(?<=\d)(?=(?:\d\d\d)+\b)", ",");
        if(frac == 0.00) { Total_Contract_Cost = concat(Total_Contract_Cost, ".00"); }
    }

    Down_Payment = ifnull(DealRec.get("Down_Payment_Amount"), "");
    if(Down_Payment != "")
    {
        dfrac = Down_Payment.frac();
        Down_Payment = Down_Payment.round(2).remove("-").toString()
                       .replaceAll("(?<!\.\d)(?<=\d)(?=(?:\d\d\d)+\b)", ",");
        if(dfrac == 0.00) { Down_Payment = concat(Down_Payment, ".00"); }
    }

    // Sales rep lookup
    YL_Rep_ID    = DealRec.get("Owner").get("id");
    YL_Rep_Name  = DealRec.get("Owner").get("name");
    crm_user_data = invokeurl
    [
        url: "https://www.zohoapis.com/crm/v2/users/" + YL_Rep_ID
        type: GET
        connection: "crm_users"
    ];
    UserRec      = crm_user_data.get("users").get(0);
    YL_Rep_Email = UserRec.get("email");

    // Sales manager routing
    SalesManager_Name  = "[DEFAULT_SALES_MANAGER]";
    SalesManager_Email = "[DEFAULT_SALES_MANAGER_EMAIL]";
    SalesManager_Val   = ifnull(UserRec.get("Reporting_To"), "");
    if(SalesManager_Val != "" && SalesManager_Email != "[DEFAULT_SALES_MANAGER_EMAIL]")
    {
        SalesManager_ID   = SalesManager_Val.get("id");
        sm_data           = invokeurl
        [
            url: "https://www.zohoapis.com/crm/v2/users/" + SalesManager_ID
            type: GET
            connection: "crm_users"
        ];
        SalesManager_Name  = sm_data.get("users").get(0).get("name");
        SalesManager_Email = sm_data.get("users").get(0).get("email");
    }

    // Template routing by Lead Source and Co-Signer
    if(CO_SignerVal != "" && LeadSource != "Source_A" && LeadSource != "Source_B")
    {
        templateID = "[TEMPLATE_ID_COSIGNER_DEFAULT]";
        actionID1 = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]";
        actionID3 = "[ACTION_ID_3]"; actionID4 = "[ACTION_ID_4]";
    }
    else if(CO_SignerVal == "" && LeadSource != "Source_A" && LeadSource != "Source_B")
    {
        templateID = "[TEMPLATE_ID_SINGLE_DEFAULT]";
        actionID1 = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]"; actionID3 = "[ACTION_ID_3]";
    }
    else if(CO_SignerVal != "" && (LeadSource == "Source_A" || LeadSource == "Source_B"))
    {
        templateID = "[TEMPLATE_ID_COSIGNER_SOURCE_AB]";
        actionID1 = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]";
        actionID3 = "[ACTION_ID_3]"; actionID4 = "[ACTION_ID_4]";
    }
    else
    {
        templateID = "[TEMPLATE_ID_SINGLE_SOURCE_AB]";
        actionID1 = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]"; actionID3 = "[ACTION_ID_3]";
    }

    // Build and send document
    fieldTextData = Map();
    fieldTextData.put("Customer_Name",   Customer_Name);
    fieldTextData.put("Project_Address", Project_Address);
    fieldTextData.put("Home_Address",    Home_Address);
    fieldTextData.put("Total_Cost",      Total_Contract_Cost);
    fieldTextData.put("Down_P",          Down_Payment);
    fieldTextData.put("YL_Rep_Name",     YL_Rep_Name);
    fieldTextData.put("SM_Name",         SalesManager_Name);
    if(CO_SignerName != "") { fieldTextData.put("CO_Signer_Name", CO_SignerName); }

    actionMap = Map();
    actionMap.put("field_data",    {"field_text_data": fieldTextData});
    actionMap.put("request_name",  "Purchase Agreement — " + Customer_Name);

    eachActionMap1 = Map();
    eachActionMap1.put("recipient_name",  YL_Rep_Name);
    eachActionMap1.put("recipient_email", YL_Rep_Email);
    eachActionMap1.put("action_type",     "SIGN");
    eachActionMap1.put("action_id",       actionID1);
    eachActionMap1.put("role",            "Consultant");
    eachActionMap1.put("verify_recipient","false");

    eachActionMap2 = Map();
    eachActionMap2.put("recipient_name",  SalesManager_Name);
    eachActionMap2.put("recipient_email", SalesManager_Email);
    eachActionMap2.put("action_type",     "SIGN");
    eachActionMap2.put("action_id",       actionID2);
    eachActionMap2.put("role",            "Sales_Manager");
    eachActionMap2.put("verify_recipient","false");

    eachActionMap3 = Map();
    eachActionMap3.put("recipient_name",  Customer_Name);
    eachActionMap3.put("recipient_email", CustomerEmail);
    eachActionMap3.put("action_type",     "SIGN");
    eachActionMap3.put("action_id",       actionID3);
    eachActionMap3.put("role",            "Signer");
    eachActionMap3.put("verify_recipient","false");

    fieldList = List();
    fieldList.add(eachActionMap1);
    fieldList.add(eachActionMap2);
    fieldList.add(eachActionMap3);

    if(CO_SignerVal != "")
    {
        eachActionMap4 = Map();
        eachActionMap4.put("recipient_name",  CO_SignerName);
        eachActionMap4.put("recipient_email", CO_SignerEmail);
        eachActionMap4.put("action_type",     "SIGN");
        eachActionMap4.put("action_id",       actionID4);
        eachActionMap4.put("role",            "CO_Signer");
        eachActionMap4.put("verify_recipient","false");
        fieldList.add(eachActionMap4);
    }

    actionMap.put("actions", fieldList);
    submitMap  = Map();
    submitMap.put("templates", actionMap);
    parameters = Map();
    parameters.put("is_quicksend", true);
    parameters.put("data",         submitMap);

    response = invokeurl
    [
        url: "https://sign.zoho.com/api/v1/templates/" + templateID + "/createdocument"
        type: POST
        parameters: parameters
        connection: "zoho_sign"
    ];

    checkResp = response.get("code");
    if(checkResp == 0)
    {
        RequestID  = response.get("requests").get("request_id");
        update_map = Map();
        update_map.put("email_id", YL_Rep_Email);
        invokeurl
        [
            url: "https://sign.zoho.com/api/v1/requests/" + RequestID + "/changeownership"
            type: PUT
            parameters: update_map
            connection: "zoho_sign"
        ];
        zoho.crm.updateRecord("Deals", Deal_ID, {"Agreement_Document_ID": RequestID});
    }
}
catch(e)
{
    sendmail
    [
        from:    zoho.adminuserid
        to:      "[ERP_ALERT_EMAIL]"
        subject: "Send Agreement Error / Lead Source: " + LeadSource
        message: "Deal ID: " + Deal_ID + " | Error: " + e
    ]
}
```

## Skills demonstrated
- Dynamic template routing with multi-condition branching
- Financial data formatting and currency handling
- Multi-party document signing workflow automation
- REST API integration with Zoho Sign
- CRM bidirectional update on workflow trigger
- Production error handling with email alerting
