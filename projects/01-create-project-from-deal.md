// Function: Create Project on Zoho Projects from CRM Deal
// Purpose: On Deal close, creates a linked Zoho Project, populates 11 custom fields,
//          assigns client company, links users, sets project owner via ZUID,
//          triggers webhook fallback on permission error.

try
{
    Deal_Details = zoho.crm.getRecordById("Deals", deal_id.toLong());
    deal_name    = ifnull(Deal_Details.get("Deal_Name"), "");

    // Ensure Account exists — create if missing
    CheckAccount = Deal_Details.get("Account_Name");
    if(CheckAccount == null)
    {
        ContactName = Deal_Details.get("Contact_Name").get("name");
        Contact_id  = Deal_Details.get("Contact_Name").get("id");
        contact_details  = zoho.crm.getRecordById("Contacts", Contact_id);
        contact_Account  = contact_details.get("Account_Name");

        if(contact_Account != null)
        {
            AccountId = contact_details.get("Account_Name").get("id");
        }
        else
        {
            mp = Map();
            mp.put("Account_Name", ContactName);
            CreateAccountResp = zoho.crm.createRecord("Accounts", mp);
            AccountId = CreateAccountResp.get("id");

            mp = Map();
            mp.put("Account_Name", AccountId);
            zoho.crm.updateRecord("Contacts", Contact_id, mp);
        }

        mp_deal = Map();
        mp_deal.put("Account_Name", AccountId);
        zoho.crm.updateRecord("Deals", deal_id, mp_deal);
        Deal_Details = zoho.crm.getRecordById("Deals", deal_id.toLong());
    }

    Fixed_Cost     = ifnull(Deal_Details.get("Total_Contract_Cost"), "0");
    Project_Sector = ifnull(Deal_Details.get("Project_Sector"), "");

    // Build rich project description from Deal fields
    Get_Deal_Info    = zoho.crm.getRecordById("Deals", deal_id.toLong());
    Contact_info     = Get_Deal_Info.get("Contact_Name");
    Get_Contact_Id   = Contact_info.get("id");
    Contact_Details  = zoho.crm.getRecordById("Contacts", Get_Contact_Id);

    newline = "<br>";
    b = Get_Deal_Info.get("Deal_Number");
    c = Get_Deal_Info.get("Owner").get("name");
    g = Get_Deal_Info.get("Contact_Name").get("name");
    L = Contact_Details.get("Phone");
    M = Contact_Details.get("Mobile");
    N = Contact_Details.get("Email");
    O = Get_Deal_Info.get("Project_Street");
    P = Get_Deal_Info.get("Project_City");
    Q = Get_Deal_Info.get("Project_State");
    R = Get_Deal_Info.get("Project_Zip_Code");
    h = Get_Deal_Info.get("Project_Type");
    i = Get_Deal_Info.get("Project_Sector");
    Y = Get_Deal_Info.get("Project_Specifics");
    Z = Get_Deal_Info.get("Description");
    U = Get_Deal_Info.get("Utility");
    V = Get_Deal_Info.get("Annual_Usage_kW_h");
    W = Get_Deal_Info.get("Homeowner_Association_Name");
    X = Get_Deal_Info.get("HOA_Info");

    pref = list();
    for each data in Get_Deal_Info.get("Owner_Contact_Preference_s") { pref.add(data); }
    Preference = pref.tostring();

    AllDescription = "";
    if(b != null) { AllDescription = AllDescription + "<b>Deal Number: </b>" + b + newline; }
    if(c != null) { AllDescription = AllDescription + "<b>Sales Rep: </b>" + c + newline; }
    if(g != null) { AllDescription = AllDescription + "<b>Contact Name: </b>" + g + newline; }
    if(N != null) { AllDescription = AllDescription + "<b>Contact Email: </b>" + N + newline; }
    if(L != null) { AllDescription = AllDescription + "<b>Home Phone: </b>" + L + newline; }
    if(M != null) { AllDescription = AllDescription + "<b>Mobile: </b>" + M + newline; }
    if(O != null && O != "NA") { AllDescription = AllDescription + "<b>Street: </b>" + O + newline; }
    if(P != null && P != "NA") { AllDescription = AllDescription + "<b>City: </b>" + P + newline; }
    if(Q != null) { AllDescription = AllDescription + "<b>State: </b>" + Q + newline; }
    if(R != null) { AllDescription = AllDescription + "<b>Zip: </b>" + R + newline; }
    if(h != null) { AllDescription = AllDescription + "<b>Project Type: </b>" + h + newline; }
    if(i != null) { AllDescription = AllDescription + "<b>Project Sector: </b>" + i + newline; }
    if(Y != null) { AllDescription = AllDescription + "<b>Project Specifics: </b>" + Y + newline; }
    if(Z != null) { AllDescription = AllDescription + "<b>Description: </b>" + Z + newline; }

    // Find project group by sector name
    groups       = invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/groups"
        type: GET
        connection: "projects"
    ];
    Group_Details = groups.get("groups");
    for each data in Group_Details
    {
        if(data.get("name") == Project_Sector) { group_id = data.get("id"); }
    }

    // Create project
    mp = Map();
    mp.put("name",                 deal_name);
    mp.put("template_id",          [PROJECT_TEMPLATE_ID]);
    mp.put("group_id",             group_id);
    mp.put("description",          AllDescription);
    mp.put("show_project_overview","yes");
    mp.put("start_date",           today.toString("MM-dd-yyyy"));
    mp.put("end_date",             today.addMonth(2).toString("MM-dd-yyyy"));
    mp.put("UDF_CHAR1",            deal_id);
    mp.put("UDF_CHAR3",            g);
    mp.put("UDF_CHAR4",            N);
    mp.put("UDF_CHAR16",           c);
    if(Fixed_Cost.toLong() > 0)
    {
        mp.put("billing_method", 3);
        mp.put("fixed_cost",     Fixed_Cost.toLong());
    }

    resp       = invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/"
        type: POST
        parameters: mp
        connection: "projects"
    ];
    ZProj_id   = resp.get("projects").toMap().get("id_string");

    // Link client company
    customer_id   = Deal_Details.get("Account_Name").get("id");
    customer_data = zoho.crm.getRecordById("Accounts", customer_id);

    if(customer_data.get("zp_client_id") == null)
    {
        param_client = Map();
        param_client.put("name", Deal_Details.get("Account_Name").get("name"));
        get_client   = invokeurl
        [
            url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/clients/"
            type: POST
            parameters: param_client
            connection: "projects"
        ];
        client_id = get_client.get("clients").get(0).get("id");
        zoho.crm.updateRecord("Accounts", customer_id, {"zp_client_id": client_id.tostring()});
    }
    else
    {
        client_id = customer_data.get("zp_client_id");
    }

    param_link = Map();
    param_link.put("company_id",     client_id);
    param_link.put("primary_client", "yes");
    invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/" + ZProj_id + "/clients/"
        type: POST
        parameters: param_link
        connection: "projects"
    ];

    // Link project to CRM Deal
    zoho.crm.updateRecord("Deals", deal_id, {"Zoho_Project_ID": ZProj_id});

    // Assign users
    Owner_ID    = Deal_Details.get("Owner").get("id");
    Owner_data  = zoho.crm.getRecordById("users", Owner_ID);
    Owner_email = Owner_data.get("users").toMap().get("email");

    mp2 = Map();
    mp2.put("email", Owner_email);
    invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/" + ZProj_id + "/users/"
        type: POST
        parameters: mp2
        connection: "projects"
    ];

    // Set default project owner
    update_map = Map();
    update_map.put("owner", [DEFAULT_PM_ZUID]);
    Update_Proj = invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/" + ZProj_id + "/"
        type: POST
        parameters: update_map
        connection: "projects"
    ];

    // Webhook fallback on permission error
    if(Update_Proj.contains("error") == true)
    {
        if(Update_Proj.get("error").get("code") == "6401")
        {
            parameter_map = Map();
            parameter_map.put("Deal_ID", deal_id);
            invokeurl
            [
                url: "[WEBHOOK_URL]"
                type: POST
                parameters: parameter_map
            ];
        }
    }

    // Populate 11 custom fields from CRM to Projects
    Deal_Details            = zoho.crm.getRecordById("Deals", deal_id);
    Potential_Financing_Model = Deal_Details.get("Potential_Financing_Model");
    Solar_Module_Manufacturer = Deal_Details.get("Solar_Module_Manufacturer");
    Solar_Module_Model        = Deal_Details.get("Solar_Module_Model");
    Solar_Module_Quantity     = Deal_Details.get("Solar_Module_Quantity");
    Inverter_Manufacturer     = Deal_Details.get("Inverter_Manufacturer");
    Inverter_Model            = Deal_Details.get("Inverter_Model");
    Inverter_Quantity         = ifnull(Deal_Details.get("Inverter_Quantity"), 0);
    Battery_Manufacturer      = Deal_Details.get("Battery_Manufacturer");
    Battery_Model             = Deal_Details.get("Battery_Model");
    Battery_Quantity          = Deal_Details.get("Battery_Quantity");
    Finance_Valid_Till        = Deal_Details.get("Finance_Valid_Till");

    Project_main_map = Map();
    Project_main_map.put("name",       deal_name);
    Project_main_map.put("UDF_CHAR12", Potential_Financing_Model);
    Project_main_map.put("UDF_CHAR13", Solar_Module_Manufacturer);
    Project_main_map.put("UDF_CHAR14", Solar_Module_Model);
    Project_main_map.put("UDF_LONG3",  Solar_Module_Quantity);
    Project_main_map.put("UDF_CHAR8",  Inverter_Manufacturer);
    Project_main_map.put("UDF_CHAR9",  Inverter_Model);
    if(Inverter_Quantity.isNumber()) { Project_main_map.put("UDF_LONG1", Inverter_Quantity); }
    Project_main_map.put("UDF_CHAR10", Battery_Manufacturer);
    Project_main_map.put("UDF_CHAR11", Battery_Model);
    Project_main_map.put("UDF_LONG2",  Battery_Quantity);
    if(Finance_Valid_Till != null) { Project_main_map.put("UDF_DATE1", Finance_Valid_Till); }

    invokeurl
    [
        url: "https://projectsapi.zoho.com/restapi/portal/[PORTAL_ID]/projects/" + ZProj_id + "/"
        type: POST
        parameters: Project_main_map
        connection: "projects"
    ];
}
catch(e)
{
    sendmail
    [
        from:    zoho.adminuserid
        to:      "[ERP_ALERT_EMAIL]"
        subject: "Create Project Error / Deal ID: " + deal_id
        message: "ID: " + deal_id + " | Error: " + e
    ]
}
