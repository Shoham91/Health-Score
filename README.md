SQL CODE FOR DATA SET:


WITH jira_issue_group_names AS (

 SELECT DISTINCT issue_key, 
 TRIM(c.value::string) as group_names_separated
 FROM "ANALYTICS"."DW_ACS"."DIM_ACS_JIRA_ISSUES_V2", lateral flatten(input=>split(groups, ',')) c
    
),

jira_issue_coe_names AS (
SELECT DISTINCT issue_key, TRIM(c.value::string) as coe_names_separated
      FROM ANALYTICS.DW_ACS.DIM_ACS_JIRA_ISSUES_V2, lateral flatten(input=>split(centers_of_excellence, ',')) c
      
)--,

/*jira_issue_coe_names AS (
SELECT DISTINCT issue_key, TRIM(c.value::string) as coe_names_separated
FROM ANALYTICS.DW_ACS.DIM_ACS_JIRA_ISSUES_V2, lateral flatten(input=>split(centers_of_excellence, ',')) c

) */

SELECT
acs_analytics_jira.issue_key,
resolution,
product,
severity,
updated_timestamp,
platform,
resolved_timestamp,
group_names_separated,
created_timestamp,
case when trim(coe_names_separated) <> 'Customer Success' then coe_names_separated end
coe_names_separated_copy,
CASE WHEN issuetype = 'Escalated User Issue' AND severity='P4' THEN 'SIB'
WHEN issuetype = 'Escalated User Issue' AND (severity='P1' or severity='P2' or severity='P3') THEN 'EUI'
ELSE NULL END
eui_or_sib,
CASE WHEN severity = 'P1' THEN 6
WHEN severity = 'P2' THEN 72
WHEN severity = 'P3' THEN 360
WHEN eui_or_sib = 'EUI' AND severity = 'P4' THEN 1032
WHEN eui_or_sib = 'SIB' AND severity = 'P4' THEN 1368
ELSE NULL END
resolution_time,
CASE WHEN severity = 'P1' THEN DATEDIFF('hh',created_timestamp,resolved_timestamp)
WHEN severity = 'P2' THEN
(DATEDIFF('day', created_timestamp, resolved_timestamp) + 1 -
DATEDIFF('week', created_timestamp, DATEADD('day', 1, resolved_timestamp)) -
DATEDIFF('week', created_timestamp, resolved_timestamp))*24
ELSE DATEDIFF('day', created_timestamp, resolved_timestamp)*24
END
hour_to_completion,
CASE WHEN hour_to_completion IS NOT NULL AND hour_to_completion <= resolution_time THEN TRUE ELSE FALSE END
FROM analytics.dw_acs.fct_acs_jira_issue_status_history_v2 AS acs_analytics_jira
LEFT JOIN jira_issue_group_names ON acs_analytics_jira.issue_key = jira_issue_group_names.issue_key
LEFT JOIN jira_issue_coe_names ON acs_analytics_jira.issue_key = jira_issue_coe_names.issue_key
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12,13,14
