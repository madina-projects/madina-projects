***Automated Daily Rider Performance Reporting***

*The query can be found in the end of the file*

In this project, I developed an automated SQL query to streamline daily monitoring of rider performance for **Team Leads** in a food delivery company. Previously, Team Leads depended on the Operations team to manually export performance data in CSV format, a process that was inefficient, prone to disruptions, and created unnecessary dependencies. Due to access restrictions, Team Leads could not directly use the company dashboard for real-time data.

***Solution:***
- I designed a SQL query in BigQuery and connected it to Google Sheets to automatically export main rider performance KPIs.
- This setup eliminated the need for manual CSV exports, enabling Team Leads to access up-to-date performance metrics directly and independently.

***Key Performance Indicators (KPIs) Tracked:***
*Calculations are based on the internal company definitions*
- Utilization Rate (UTR)
- Working Time (Hours)
- Late Arrivals: Late 5 min count, Late 30 min count
- Shift Metrics: Shifts under 60 min count, No-shows
- Idle Time Percentage
- Acceptance Rate Percentage
- Break Time Metrics: Break time (minutes), Break percentage (ratio of break time to total worked time)
- Average Time at Customer (Minutes)
- Cancellations: Before pickup, After pickup

***Tools Used:***
*BigQuery* for SQL querying and data processing
*Google Sheets* for automated data export and access

***Impact:***
- Streamlined daily performance monitoring for over 30 Team Leads, supporting a fleet of 5,000 riders in Sweden.
- Eliminated manual CSV exports, reducing inefficiencies and dependencies between Team Leads and the Operations team.
- The automated data export has become a crucial resource for staffing decisions, compliance sessions, and contract extensions.
- This project significantly improved the workflow, empowering Team Leads with direct access to critical performance data and enhancing their ability to provide timely feedback to riders.

```sql
WITH
  date_param AS (
    SELECT current_date-28 AS start_date, current_date-1 AS end_date),

  decline_reason_prep AS (
  SELECT
    rider_id,
    created_date,
    COUNT(*) AS n_cancellations
  FROM
    `decline_reasons` d ---masked data set
  WHERE
    created_date BETWEEN current_date-28  AND current_date-1
    AND decline_reason_type = 'TRAFFIC_MANAGER'
    AND decline_reason IN ('Customer never received the order',
      'Food_quality_spillage',
      'Courier_unreachable',
      'Customer_received_wrong_order') --- elaborates order cancellation reasons related to the Logistics (courier) and happened before the order pickup
    AND rider_id IS NOT NULL
  GROUP BY 1,2),

  regions AS (
  SELECT
    region_name,
    city_name,
    area_manager,
    city_id
  FROM
    `regions` --- masked data set
  WHERE
    country_code ='se')

SELECT
  report_date,
  rider_id,
  rider_name,
  city_name,
  captain_name,
  UTR,
  working_time_hours,
  late_5min_count,
  late_30min_count,
  shifts_under_60_count,
  no_shows,
  CASE
    WHEN idle_time < 0 THEN NULL
    ELSE idle_time
END
  AS idle_time_percentage,
  acceptance_rate_percentage,
  break_time_minutes,
  ROUND(SAFE_DIVIDE(break_time_minutes,total_worked_min)*100,2) AS break_percentage, --- ratio of break time to total worked time
  avg_at_customer_time_minutes,
  cancellations_before_pickup,
  cancelled_deliveries_after_pickup,
  EXTRACT (week
  FROM (DATE_TRUNC(report_date, isoweek))) AS week,
  entity,
  region_name,
FROM (
  SELECT
    *,
    ROUND(SUM(COALESCE(break_time_minutes,0) + COALESCE(working_time_min,0)),2) AS total_worked_min 
  FROM (
    SELECT
      DISTINCT report_date,
      r.rider_id,
      rider_name,
      region_name,
      r.city_name,
      team_lead_name, ---rider team lead name required to sort riders and their performance to a corresponding manager who will be checking their KPIs
      ROUND(SAFE_DIVIDE (completed_deliveries, working_time/3600), 2) AS UTR,  ---utilization rate for deliveries, one of key logistics KPIs in the food delivery industry
      count_late_login_5_min AS late_5min_count,  --- logged in 5 min after the start
      count_late_login_30_min AS late_30min_count, --- logged in  30 min after the start
      count_shifts_under_60_min AS shifts_under_60_count,  --- shifts worked less than 60 min
      no_shows, --- rider not showing up to the shift
      ROUND(SAFE_DIVIDE((transition_working_time-transition_busy_time),transition_working_time)*100,1) AS idle_time,  --- time during worked hours without deliveries
      ROUND(SAFE_DIVIDE(rider_accepted,rider_notified)*100,2) AS acceptance_rate_percentage,  ---rider accepting the order
      ROUND(SAFE_DIVIDE(break_time,60),2) AS break_time_minutes, --- break time
      ROUND(SAFE_DIVIDE(at_customer_time_sum,at_customer_time_count)/60,2) AS avg_at_customer_time_minutes, --- how much time rider spends at the customer location
      CASE WHEN n_cancellations IS NULL THEN 0 ELSE n_cancellations END AS cancellations_before_pickup, ---count of cancellations caused by logistics/rider
      ROUND(SAFE_DIVIDE(working_time,3600),2) AS working_time_hours, --- worked hours
      ROUND(SAFE_DIVIDE(working_time,60),2) AS working_time_min, --- worked minutes
      entity --- business entity needed to sort in-hours and 3PL riders
    FROM
      `rider_performance`  r ---masked data set
    LEFT JOIN 
    cancellation_prep
    ON cancellation_prep.rider_id = r.rider_id
      AND cancellation_prep.created_date=r.report_date
    LEFT JOIN
      `courier_entity` e ---masked data set
    ON
      e.job_title = r.job_title
    LEFT JOIN
      regions re
    ON
      r.city_name = re.city_name
    WHERE
      entity IN ('BRAND_NAME_1',
        'BRAND_NAME_2',
        '3PL_1',
        '3PL_2') --- brands and 3PL partner names masked
      AND country_code = 'se'
      AND report_date BETWEEN (
      SELECT
        start_date
      FROM
        date_param)
      AND (
      SELECT
        end_date
      FROM
        date_param))
  GROUP BY
    1,2,3,4,5,6,7,8,9,10,11,12,13,14,15,16,17,18,19,20,21)
WHERE
  region_name IN ('Region_name_1',
    'Region_name_2',
    'Region_name_3',
    'Region_name_4',
    'Region_name_5') --- masked business region names
ORDER BY
  report_date desc
