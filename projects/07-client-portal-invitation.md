# Send Client Portal Invitation on Agreement Signing

## What this does
Workflow-triggered function that fires automatically when a purchase
agreement is signed. It imports the CRM Account into Zoho Books,
fetches the primary contact, checks whether the client portal is
already enabled, enables it if not, sets the invitation sent date
and next reminder date as custom fields in Zoho Books, and sends
an internal alert email if no email address exists on the contact.

## Problem it solved
After agreements were signed, the accounts team had to manually
import the customer into Zoho Books, locate the correct contact,
enable their client portal access, and track when the invitation
was sent. This was frequently delayed by 2–3 days, causing
customers to miss payment deadlines and follow-up reminders to
be sent inconsistently.

## Outcome
- Client portal invitation sent automatically within seconds
  of agreement signing
- Invitation date and next reminder date set programmatically
  in Zoho Books — no manual tracking required
- Internal alert fired immediately if contact has no email,
  preventing silent failures
- Eliminated 2–3 day delay in customer onboarding to portal

## Tech used
Deluge Scripting · Zoho CRM · Zoho Books · REST API ·
Workflow Automation · Custom Field Management

## Code

```javascript
// Function: Send Client Portal Invitation when Agreement Signed
// Purpose: Imports CRM Account into Zoho Books, checks portal
// status, enables portal for primary contact, sets invitation
// date and next reminder date as custom fields.
// Sends alert if email missing.

Customers = invokeurl
[
    url: "https://www.zohoapis.com/books/v3/crm/account/" + Accountid + "/import?organization_id=[ORG_ID]"
    type: POST
    connection: "books"
];

CustomerId = Customers.get("data").get("customer_id");

book_info      = zoho.books.getRecordsByID("Contacts", "[ORG_ID]", CustomerId, "books");
contact_name   = book_info.get("contact").get("contact_name");
primary_contact = book_info.get("contact").get("primary_contact_id");
portal_status  = book_info.get("contact").get("portal_status");

if(portal_status != "enabled")
{
    email = book_info.get("contact").get("email");

    if(email != "")
    {
        C_persons = List();
        map = Map();
        map.put("contact_person_id", primary_contact);
        C_persons.add(map);

        data = Map();
        data.put("contact_persons", C_persons);
        Json_data = Map();
        Json_data.put("JSONString", data);

        ENABLE = invokeurl
        [
            url: "https://www.zohoapis.com/books/v3/contacts/" + CustomerId + "/portal/enable?organization_id=[ORG_ID]"
            type: POST
            parameters: Json_data
            connection: "books"
        ];

        if(ENABLE.get("code") == 0)
        {
            currentdate   = now.toString("yyyy-MM-dd");
            Next_Reminder = currentdate.addDay(3).toString("yyyy-MM-dd");

            CustomFields = List();

            CustomField1 = Map();
            CustomField1.put("label", "Portal_Invitation_Sent");
            CustomField1.put("value", currentdate);

            CustomField2 = Map();
            CustomField2.put("label", "Client_Portal_Next_Reminder");
            CustomField2.put("value", Next_Reminder);

            CustomFields.add(CustomField1);
            CustomFields.add(CustomField2);

            json = Map();
            json.put("custom_fields", CustomFields);
            zoho.books.updateRecord("Contacts", "[ORG_ID]", CustomerId, json, "books");
        }
    }
    else
    {
        sendmail
        [
            from:    zoho.loginuserid
            to:      "[ACCOUNTS_ALERT_EMAIL]"
            subject: "Client Portal Invitation Not Sent — " + contact_name
            message: "Portal invitation could not be sent to " + contact_name
                   + " because no email address exists on the contact record."
        ]
    }
}
```

## Skills demonstrated
- Cross-platform CRM to billing system integration
- Portal access management via Zoho Books API
- Custom field automation for date tracking and reminders
- Silent failure prevention via email alerting
- Idempotent design — portal enable skipped if already active
- Production deployment in live customer onboarding workflow
