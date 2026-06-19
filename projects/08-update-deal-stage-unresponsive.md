# Auto-Update Deal Stage to Unresponsive

## What this does
Activity-triggered function that fires on every call logged against
a Deal. It fetches the last 3 calls via Zoho CRM's COQL query
language, evaluates each call's result and duration, counts how
many are unresponsive or under 2 minutes, and if all 3 qualify,
automatically updates the consecutive unresponsive field and
triggers a Blueprint stage transition to move the Deal to
Unresponsive — without any manual intervention from the sales team.

## Problem it solved
Sales reps were manually reviewing call logs to decide whether a
deal should be marked unresponsive, which was inconsistent and
time-consuming. Some deals sat in the New Deal stage for weeks
with no activity because no one flagged them. This distorted
pipeline reporting and wasted follow-up resources.

## Outcome
- Unresponsive deals identified and stage-transitioned
  automatically after 3 qualifying calls
- Pipeline reporting accuracy improved immediately
- Sales managers gained a reliable view of stuck deals
  without manual review
- Blueprint transition enforced consistent stage governance
  across all sales reps

## Tech used
Deluge Scripting · Zoho CRM · COQL · Blueprint Automation ·
Activity Triggers · REST API

## Code

```javascript
// Function: Update Deal Stage to Unresponsive
// Purpose: Triggered on call activity. Fetches last 3 calls
// via COQL, evaluates result and duration. If all 3 are
// unresponsive or under 2 minutes, updates consecutive
// unresponsive count and triggers Blueprint stage transition.

Activity_Details = zoho.crm.getRecordById("Activities", activity_id);
Module           = Activity_Details.get("$se_module");
Lead_Deal_ID     = Activity_Details.get("What_Id").get("id");

if(Module == "Deals")
{
    deal_detail = zoho.crm.getRecordById("Deals", Lead_Deal_ID);
    deal_Stage  = deal_detail.get("Stage");

    if(deal_Stage == "New Deal")
    {
        // Fetch last 3 calls via COQL
        query = "{\"select_query\":\"select Call_Start_Time, Call_Result, Call_Duration_in_seconds "
              + "from Calls where (What_Id = '" + Lead_Deal_ID + "') "
              + "order by Call_Start_Time DESC limit 0,3\"}";

        All_Calls = invokeurl
        [
            url: "https://www.zohoapis.com/crm/v2/coql"
            type: POST
            parameters: query
            connection: "zoho_crm_coql"
        ];

        CallsList = All_Calls.get("data");

        if(CallsList.size() < 3) { return; }

        BadCount = 0;
        for each Call in CallsList
        {
            result           = ifnull(Call.get("Call_Result"), "");
            duration_seconds = ifnull(Call.get("Call_Duration_in_seconds"), 0);
            duration_minutes = duration_seconds / 60;

            if(result == "No Response/Busy" || duration_minutes < 2)
            {
                BadCount = BadCount + 1;
            }
        }

        if(BadCount == 3)
        {
            // Update consecutive unresponsive field
            mp = Map();
            mp.put("Unresponsive_Calls_Consecutive", 5);
            zoho.crm.updateRecord("Deals", Lead_Deal_ID, mp, {"trigger": {"workflow"}});

            // Trigger Blueprint stage transition
            blueprint_entry = Map();
            blueprint_entry.put("transition_id", "[BLUEPRINT_TRANSITION_ID]");
            blueprint_list  = List();
            blueprint_list.add(blueprint_entry);
            payload = Map();
            payload.put("blueprint", blueprint_list);

            invokeurl
            [
                url: "https://www.zohoapis.com/crm/v6/Deals/" + Lead_Deal_ID + "/actions/blueprint"
                type: PUT
                parameters: payload.toString()
                connection: "zoho_crm"
            ];
        }
    }
}
```

## Skills demonstrated
- COQL query execution for dynamic activity data retrieval
- Multi-condition call quality evaluation logic
- Blueprint stage transition via REST API
- Activity-triggered automation with guard conditions
- Pipeline integrity enforcement without manual oversight
- Production deployment in live sales operations environment
