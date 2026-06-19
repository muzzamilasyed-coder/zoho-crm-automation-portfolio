# Live Project Operations Dashboard — SQL

## What this does
A production SQL query powering a real-time operations dashboard
used daily by senior leadership. It joins five tables across Deals,
Projects, Tasks, Task Completion, and Invoices, applies complex
CASE WHEN logic for stage ordering, payment status evaluation,
consultant and project manager initials lookup, and generates
deep-link URLs directly into CRM and project records — all in
a single query.

## Problem it solved
Leadership had no unified view of where every active project stood
across the full 20-stage pipeline. Status updates required manually
checking CRM, Zoho Projects, and Zoho Books separately. Overdue
tasks, payment statuses, and stage durations were invisible without
running multiple separate reports.

## Outcome
- Single dashboard replaced three separate manual reporting tools
- 20-stage pipeline visible in real time with days-in-stage tracking
- Payment status logic surfaced Down Payment, Second Payment,
  and Final Payment states automatically
- Deep-link URLs gave leadership one-click access to any Deal
  or Project directly from the dashboard
- Adopted as the primary daily operations tool for senior management

## Tech used
SQL · Zoho Analytics · 5-table LEFT JOIN · CASE WHEN Logic ·
days_between() · Dynamic URL Generation · Multi-condition
Payment Status Evaluation

## Query

```sql
-- Query: Live Project Operations Dashboard
-- Purpose: Joins Deals, Projects, Tasks, Task Completion, and
-- Invoices to produce a real-time operational view of all active
-- and on-hold projects. Includes stage ordering, consultant and
-- PM initials, payment status logic, dynamic CRM and project
-- URLs, and days-since tracking.

SELECT DISTINCT
    "Projects"."Project ID"                          AS "Project ID",
    "Deals"."Deal Number"                            AS "Deal Number",
    "Projects"."Project Name"                        AS "Project Name",
    "Projects"."Expiry Date"                         AS "Finance Expiry",
    "Deals"."Project City"                           AS "City",
    "Deals"."Project State"                          AS "State",
    "Deals"."Project Zip Code"                       AS "Zip",
    "Deals"."Project Sector"                         AS "Project Sector",

    -- Normalise project type into two categories
    CASE
        WHEN "Deals"."Project Type" IN (
            'Inventory Order', 'Panel Addition',
            'Panel and Storage Addition', 'Retrofit',
            'Service', 'Storage Addition'
        ) THEN 'Retrofit'
        WHEN "Deals"."Project Type" = 'New Construction' THEN 'New Construction'
    END AS "Project Type",

    -- Consultant initials lookup
    CASE
        WHEN "Deals"."Deal Owner Name" = 'Consultant A'  THEN 'C.A'
        WHEN "Deals"."Deal Owner Name" = 'Consultant B'  THEN 'C.B'
        WHEN "Deals"."Deal Owner Name" = 'Consultant C'  THEN 'C.C'
        -- Additional consultants follow same pattern
    END AS "Cnslt",

    -- Project Manager initials lookup
    CASE
        WHEN "Projects"."Owner" = 'Manager A' THEN 'M.A'
        WHEN "Projects"."Owner" = 'Manager B' THEN 'M.B'
        WHEN "Projects"."Owner" = 'Manager C' THEN 'M.C'
        -- Additional managers follow same pattern
    END AS "P.M",

    "Deals"."System Size (kW)"                      AS "kW",
    "Deals"."Contract Signed Date"                  AS "CSA D",
    "Projects"."Status"                             AS "Project Status",
    "Projects"."Start Date"                         AS "Start Date",
    days_between("Deals"."Contract Signed Date")    AS "Days Since Signed",
    "Projects"."Stage"                              AS "Project Stage",
    days_between("Projects"."Last Modified Time")   AS "Days in Stage",

    -- Numeric stage ordering for sort
    CASE
        WHEN "Projects"."Stage" = 'Project Initiation'                  THEN '02 - Project Initiation'
        WHEN "Projects"."Stage" = 'Site Evaluation'                     THEN '03 - Site Evaluation'
        WHEN "Projects"."Stage" = 'Design'                              THEN '04 - Design'
        WHEN "Projects"."Stage" = 'PM Approval'                         THEN '05 - PM Approval'
        WHEN "Projects"."Stage" = 'Homeowner Approval'                  THEN '06 - Homeowner Approval'
        WHEN "Projects"."Stage" = 'Permit & Interconnection Submission'  THEN '07 - Permit & Interconnection Submission'
        WHEN "Projects"."Stage" = 'Permit & Interconnection Approval'    THEN '08 - Permit & Interconnection Approval'
        WHEN "Projects"."Stage" = 'Inventory Approval'                   THEN '09 - Inventory Approval'
        WHEN "Projects"."Stage" = 'Inventory Order'                      THEN '10 - Inventory Order'
        WHEN "Projects"."Stage" = 'Inventory Received'                   THEN '11 - Inventory Received'
        WHEN "Projects"."Stage" = 'Installation'                         THEN '12 - Installation'
        WHEN "Projects"."Stage" = 'Inspection'                           THEN '13 - Inspection'
        WHEN "Projects"."Stage" = 'Final Interconnection'                THEN '14 - Final Interconnection'
        WHEN "Projects"."Stage" = 'Bi-Directional Meter'                 THEN '15 - Bi-Directional Meter'
        WHEN "Projects"."Stage" = 'Closeout'                             THEN '16 - Closeout'
        WHEN "Projects"."Stage" = 'Customer Feedback'                    THEN '17 - Customer Feedback'
        WHEN "Projects"."Stage" = 'Commissions'                          THEN '18 - Commissions'
        WHEN "Projects"."Stage" = 'Collections'                          THEN '19 - Collections'
        WHEN "Projects"."Stage" = 'Completed'                            THEN '20 - Completed'
        WHEN "Projects"."Stage" = 'On Hold'                              THEN '21 - On Hold'
        WHEN "Projects"."Stage" = 'Cancelled'                            THEN '22 - Cancelled'
        ELSE '01'
    END AS "Project Stage Order",

    -- Site evaluation task status
    CASE
        WHEN "Tasks"."Site Evaluation" = 'Active'    THEN 'Active'
        WHEN "Tasks"."Site Evaluation" = 'Completed' THEN 'Completed'
        WHEN "Tasks"."Site Evaluation" = 'Cancelled' THEN 'N/A'
        WHEN "Tasks"."Site Evaluation" IS NULL
          OR "Tasks"."Site Evaluation" = ''          THEN 'Deferred'
        ELSE "Tasks"."Site Evaluation"
    END AS "Site Eval",

    -- Customer feedback call status with payment gate check
    CASE
        WHEN "Tasks"."Final Payment"         = 'Completed'
         AND "Tasks"."Bi-Directional Meter"  = 'Completed'
         AND "Tasks"."Customer Feedback Call"= 'Active'    THEN 'Active'
        WHEN "Tasks"."Customer Feedback Call"= 'Active'    THEN 'Pending'
        WHEN "Tasks"."Customer Feedback Call"= 'Completed' THEN 'Completed'
        WHEN "Tasks"."Customer Feedback Call"= 'Cancelled' THEN 'Un-assigned'
        WHEN "Tasks"."Customer Feedback Call" IS NULL
          OR "Tasks"."Customer Feedback Call" = ''         THEN 'Deferred'
        ELSE "Tasks"."Customer Feedback Call"
    END AS "Feedback Call",

    -- Down payment status
    CASE
        WHEN "Tasks"."D.P" IN ('Active','Deferred','Cancelled','Completed','N/A')
         AND "Projects"."Expiry Date" IS NULL
         AND "Projects"."Financing Model" <> 'Cash'        THEN 'U/A'
        WHEN "Tasks"."D.P" = 'Active'                      THEN 'P'
        WHEN "Tasks"."D.P" = 'Completed'                   THEN 'R'
        WHEN "Tasks"."D.P" = 'Cancelled'                   THEN 'C'
        WHEN "Tasks"."D.P" = 'Deferred'                    THEN 'D'
        ELSE ''
    END AS "Down Payment",

    -- Second payment status
    CASE
        WHEN "Tasks"."Second Payment" = 'Active'    THEN 'I/A'
        WHEN "Tasks"."Second Payment" = 'Completed' THEN 'R'
        WHEN "Tasks"."Second Payment" = 'Cancelled' THEN 'C'
        WHEN "Tasks"."Second Payment" = 'Deferred'  THEN 'D'
        ELSE "Tasks"."Second Payment"
    END AS "Second Payment",

    -- Final payment status
    CASE
        WHEN "Tasks"."Final Payment" = 'Active'    THEN 'I/A'
        WHEN "Tasks"."Final Payment" = 'Completed' THEN 'R'
        WHEN "Tasks"."Final Payment" = 'Cancelled' THEN 'C'
        WHEN "Tasks"."Final Payment" = 'Deferred'  THEN 'D'
        ELSE "Tasks"."Final Payment"
    END AS "Final Payment",

    "Projects"."Financing Model"                             AS "Financing Mode",
    "Deals"."Solar Module Quantity"                          AS "Panel Qty",
    "Deals"."Solar Module Manufacturer"                      AS "Panel Brand",
    "Deals"."Solar Module Model"                             AS "Panel Model",
    "Deals"."Battery Quantity"                               AS "Battery Count",
    "Deals"."Inverter Manufacturer"                          AS "Inverter",
    "Deals"."Anticipated Completion Date"                    AS "Expected Completion",
    "Deals"."Total Contract Cost"                            AS "Contract Value",

    CASE
        WHEN "Invoices"."Receivable" IS NULL THEN 0
        ELSE "Invoices"."Receivable"
    END AS "Receivable",

    CASE
        WHEN "Invoices"."Balance" IS NULL THEN 0
        ELSE "Invoices"."Balance"
    END AS "Balance",

    "Projects"."Site Evaluation Date"                        AS "Site Eval Date",
    "Deals"."Project Manager Updates"                        AS "Notes",
    "Task Completion"."overdue tasks"                        AS "Overdue Tasks",

    -- Deep-link URLs for direct CRM and Project access
    concat('[CRM_BASE_URL]/tab/Potentials/', "Deals"."ID")                   AS "Deal Link",
    concat('[PROJECTS_BASE_URL]/dashboard/', "Projects"."Project ID")         AS "Project Link",
    concat("Deals"."Project Street", ' ', "Deals"."Project City", ' ',
           "Deals"."Project State", ' ', "Deals"."Project Zip Code")         AS "Address"

FROM "Deals"
LEFT JOIN "Projects"         ON "Projects"."Project ID"  = "Deals"."Zoho Project ID"
LEFT JOIN "Tasks"            ON "Tasks"."Project ID"     = "Deals"."Zoho Project ID"
LEFT JOIN "Task Completion"  ON "Task Completion"."Project ID" = "Deals"."Zoho Project ID"
LEFT JOIN "Invoices"         ON "Invoices"."Project Name" = "Projects"."Project Name"

WHERE  "Deals"."Stage"    = 'Closed Won'
AND   ("Projects"."Status" = 'Active' OR "Projects"."Status" = 'On Hold')

ORDER BY "Project Stage Order" ASC, "Projects"."Stage" ASC;
```

## Skills demonstrated
- Complex 5-table LEFT JOIN across live operational data
- 20-stage pipeline ordering via CASE WHEN sort logic
- Multi-condition payment status evaluation per task state
- days_between() for real-time stage duration tracking
- Dynamic URL generation for deep-link navigation
- Production deployment as primary leadership decision tool
- Consultant and PM initials lookup across 25+ team members
