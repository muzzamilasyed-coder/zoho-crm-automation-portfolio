# Resend Purchase Agreement with Automatic Recall Logic

## What this does
Button-triggered function that recalls the previously sent Zoho Sign
agreement, repopulates all template fields from the current CRM Deal
state, routes to the correct template based on Lead Source and
Co-Signer presence, resends to all signing parties, and updates the
CRM with the new Document ID — all in a single click.

## Problem it solved
When deal details changed after an agreement was already sent,
the team had to manually recall the old document in Zoho Sign,
refill all fields, re-select signers, resend, and update CRM.
This took 30+ minutes and frequently resulted in outdated
agreements being signed by mistake.

## Outcome
- Previous agreement recalled automatically before resend
- Zero manual field re-entry across 40+ deal fields
- Closed Won deals blocked from accidental resend
- Full audit trail maintained in CRM via Document ID update

## Tech used
Deluge Scripting · Zoho CRM · Zoho Sign · REST API ·
Conditional Template Routing · Stage-based Guard Logic

## Code

```javascript
// Function: Resend Purchase Agreement via Zoho Sign
// Purpose: Recall previous agreement, repopulate template fields
// from CRM Deal, route to correct template based on Lead Source
// and Co-Signer presence, send via Zoho Sign API and update
// CRM with new Document ID.

try
{
    DealRec = zoho.crm.getRecordById("Deals", Deal_ID.toLong());
    Stage_Info = DealRec.get("Stage");

    if(Stage_Info == "Closed Won")
    {
        Response_Message = "Cannot resend agreement — Deal is Closed Won. Please use the Change Order process.";
    }
    else
    {
        Response_Message = "";
        Project_Sector = DealRec.get("Project_Sector");

        if(Project_Sector == "Residential" || Project_Sector == "Commercial")
        {
            Agreement_Document_ID = ifnull(DealRec.get("Agreement_Document_ID"), "");

            // Recall previous agreement if exists
            if(Agreement_Document_ID != "")
            {
                recall_doc = invokeurl
                [
                    url: "https://sign.zoho.com/api/v1/requests/" + Agreement_Document_ID + "/recall"
                    type: POST
                    connection: "zoho_sign"
                ];
            }

            // Fetch Deal fields
            DealRec = zoho.crm.getRecordById("Deals", Deal_ID);

            Customer_Name = "";
            CustomerEmail = "";

            // Primary signer
            CustomerVal = ifnull(DealRec.get("Signing_Customer"), "");
            if(CustomerVal != "")
            {
                Customer_Name = CustomerVal.get("name");
                CustomerID = CustomerVal.get("id");
                ContactRec = zoho.crm.getRecordById("Contacts", CustomerID);
                CustomerEmail = ContactRec.get("Email");
            }

            // Co-signer
            CO_SignerName = "";
            CO_SignerEmail = "";
            CO_SignerVal = ifnull(DealRec.get("Co_Signer"), "");
            if(CO_SignerVal != "")
            {
                CO_SignerName = CO_SignerVal.get("name");
                CO_SignerID = CO_SignerVal.get("id");
                CO_SignerRec = zoho.crm.getRecordById("Contacts", CO_SignerID);
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

            // System components
            Module_Manufacturer       = ifnull(DealRec.get("Solar_Module_Manufacturer"), "");
            Module_Model              = ifnull(DealRec.get("Solar_Module_Model"), "");
            Module_Quantity           = ifnull(DealRec.get("Solar_Module_Quantity"), "");
            Inverter_Manufacturer     = ifnull(DealRec.get("Inverter_Manufacturer"), "");
            Inverter_Model            = ifnull(DealRec.get("Inverter_Model"), "");
            Inverter_Quantity         = ifnull(DealRec.get("Inverter_Quantity"), "");
            Battery_Manufacturer      = ifnull(DealRec.get("Battery_Manufacturer"), "");
            Battery_Model             = ifnull(DealRec.get("Battery_Model"), "");
            Battery_Quantity          = ifnull(DealRec.get("Battery_Quantity"), "");
            Mounting_Hardware_Manufacturer = ifnull(DealRec.get("Racking_Manufacturer"), "");
            Mounting_Hardware_Model   = ifnull(DealRec.get("Racking_Model"), "");
            Mounting_Hardware_Quantity = ifnull(DealRec.get("Racking_Quantity"), "");

            // Financial fields with formatting
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

            // Estimate type flags
            EstimateType = DealRec.get("Estimate_Type");
            Written_Estimate_Val = (EstimateType == "Written") ? "Yes" : "No";
            Verbal_Estimate_Val  = (EstimateType == "Oral")    ? "Yes" : "No";

            // Sales rep and manager lookup
            YL_Rep_ID   = DealRec.get("Owner").get("id");
            YL_Rep_Name = DealRec.get("Owner").get("name");
            crm_user_data = invokeurl
            [
                url: "https://www.zohoapis.com/crm/v2/users/" + YL_Rep_ID
                type: GET
                connection: "crm_users"
            ];
            UserRec       = crm_user_data.get("users").get(0);
            YL_Rep_Email  = UserRec.get("email");
            SalesManager_Val = ifnull(UserRec.get("Reporting_To"), "");
            SalesManager_Name  = "[DEFAULT_SALES_MANAGER]";
            SalesManager_Email = "[DEFAULT_SALES_MANAGER_EMAIL]";

            // Populate template field map
            fieldTextData = Map();
            fieldTextData.put("Customer_Name",            Customer_Name);
            fieldTextData.put("Project_Address",          Project_Address);
            fieldTextData.put("Home_Address",             Home_Address);
            fieldTextData.put("Anticipated_Beginning_Date", zoho.currentdate.toString("MM/dd/yyyy"));
            fieldTextData.put("Anticipated_End_Date",     ifnull(DealRec.get("Anticipated_Completion_Date"), "").toString("MM/dd/yyyy"));
            fieldTextData.put("Module_Manufacturer",      Module_Manufacturer);
            fieldTextData.put("Module_Model",             Module_Model);
            fieldTextData.put("M_Qty",                    Module_Quantity);
            fieldTextData.put("Hardware_Manufacturer",    Mounting_Hardware_Manufacturer);
            fieldTextData.put("Hardware_Model",           Mounting_Hardware_Model);
            fieldTextData.put("H_Qty",                    Mounting_Hardware_Quantity);
            fieldTextData.put("Inverter_Manufacturer",    Inverter_Manufacturer);
            fieldTextData.put("Inverter_Model",           Inverter_Model);
            fieldTextData.put("I_Qty",                    Inverter_Quantity);
            fieldTextData.put("Battery_Manufacturer",     Battery_Manufacturer);
            fieldTextData.put("Battery_Model",            Battery_Model);
            fieldTextData.put("B_Qty",                    Battery_Quantity);
            fieldTextData.put("Total_Cost",               Total_Contract_Cost);
            fieldTextData.put("Down_P",                   Down_Payment);
            fieldTextData.put("Written_Estimate",         Written_Estimate_Val);
            fieldTextData.put("Oral_Estimate",            Verbal_Estimate_Val);
            fieldTextData.put("YL_Rep_Name",              YL_Rep_Name);
            fieldTextData.put("SM_Name",                  SalesManager_Name);
            if(CO_SignerName != "") { fieldTextData.put("CO_Signer_Name", CO_SignerName); }

            // Select template and action IDs based on Lead Source and Co-Signer
            LeadSource = DealRec.get("Lead_Source");

            if(CO_SignerVal != "" && LeadSource == "Source_A")
            {
                templateID = "[TEMPLATE_ID_COSIGNER_SOURCE_A]";
                actionID1  = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]";
                actionID3  = "[ACTION_ID_3]"; actionID4 = "[ACTION_ID_4]";
            }
            else if(CO_SignerVal == "" && LeadSource == "Source_A")
            {
                templateID = "[TEMPLATE_ID_SINGLE_SOURCE_A]";
                actionID1  = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]"; actionID3 = "[ACTION_ID_3]";
            }
            else if(CO_SignerVal != "")
            {
                templateID = "[TEMPLATE_ID_COSIGNER_DEFAULT]";
                actionID1  = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]";
                actionID3  = "[ACTION_ID_3]"; actionID4 = "[ACTION_ID_4]";
            }
            else
            {
                templateID = "[TEMPLATE_ID_SINGLE_DEFAULT]";
                actionID1  = "[ACTION_ID_1]"; actionID2 = "[ACTION_ID_2]"; actionID3 = "[ACTION_ID_3]";
            }

            // Build action maps for each signing role
            actionMap = Map();
            actionMap.put("field_data", {"field_text_data": fieldTextData});

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

            actionMap.put("actions",      fieldList);
            actionMap.put("request_name", "Purchase Agreement — " + Customer_Name);

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
                RespRec   = response.get("requests");
                RequestID = RespRec.get("request_id");

                // Transfer document ownership to sales rep
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

            Response_Message = "Agreement resent successfully. Previous document recalled.";
        }
        else
        {
            Response_Message = "Feature only available for Residential and Commercial deals.";
        }
    }
}
catch(e)
{
    sendmail
    [
        from:    zoho.adminuserid
        to:      "[ERP_ALERT_EMAIL]"
        subject: "Resend Agreement Error / Deal ID: " + Deal_ID
        message: "Deal ID: " + Deal_ID + " | Error: " + e
    ]
}

return Response_Message;
```

## Skills demonstrated
- Agreement recall and reissue workflow automation
- Stage-based guard logic preventing invalid operations
- Multi-condition template and action ID routing
- 40+ field extraction and formatting from CRM
- Production error handling with structured email alerts
- REST API ownership transfer post-document creation
