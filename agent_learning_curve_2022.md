
***Agent Learning Curve for Contact Center Performance Analysis***

*The query can be found in the end of the file*

This project focuses on building an Agent Learning Curve for the Customer Service (CS) team at a food delivery company.
The goal is to provide a data-driven approach to understanding the agent learning process, set realistic targets and expectations based on agent development,and identify the factors that either contribute to or hinder this development.

The learning curve is a graphical representation that illustrates how agent performance evolves over time.

***Objectives***
- Provide realistic insights into the agent learning process.
- Set performance targets and expectations that align with agents' development pace.
- Identify factors that contribute to or block agents' progress in meeting performance metrics.

***Data Overview***
- Sample size: 146 agents
- Team: Customer Service (CS)
- Shift Patterns: 25%, 50%, 75% (part-time types)

***Key Metrics and Targets:***

This project focused on the following key performance metrics, which are closely tied to agent productivity, one of the most important criteria in a contact center environment:

- AHT (Average Handling Time) for chats: 6-7 minutes
- Productivity: 14 (work items per hour)
- AWUT (Average Wrap Up Time) for chats: less than 1 minute

These metrics were selected as they provide a direct correlation to agent productivity and contribute significantly to overall performance evaluation. The project aimed to monitor these metrics closely to help optimize agent efficiency and improve contact center operations.

***Methodology:***
- Data Definitions: Calculations for each metric were based on the company's internal definitions.
- Outlier Removal: Data points that were identified as outliers were removed to ensure accuracy and avoid skewed results.
- SQL Query: The analysis was carried out using SQL queries that extracted, aggregated, and processed the data from the company's internal databases.

***Tools***
- BigQuery for data extraction adn aggregation
- Excel for exploratory analysis and outlier identification
- Excel/Sheets visualization

***Outcome***

- The analysis helped the Training & Quality teams better understand learning trends across different shift types, enabling them to set more realistic targets for part-time agents (shift patterns 25%, 50%, 75%)
- A high churn rate in the first two months post-hire can introduce bias into the learning trends, so it is recommended to exclude this batch from future analyses
- BPO and in-house agents exhibit different learning paces, with in-house staff reaching targets more quickly
- As agents progress through their learning timeline, there is a correlational growth of Productivity, AHT, and AWUT
- Agents in shift pattern 75 reach most of the targets by month 7, which indicates on a potential target adjustment, especially for other part-time shift types
- Certain contact reasons that result longer handling are recommended to be excluded as well to minimize bias

Agent Learning Curve - overall learning pace for all KPIs

<a href="https://ibb.co/SmFsXkb"><img src="https://i.ibb.co/YdHR75m/Screenshot-2024-11-07-at-14-03-06.png" alt="Screenshot-2024-11-07-at-14-03-06" border="0"></a>```sql

Combined visualization of 3 key metrics related to the agent productivity

*The chart shows AVG of each metric for the shift type 75. The targets for the chat handling time and the wrap-up time achieved on month 5-6 from the hire date*
*We can observe that as the agent productivity grows overtime less time required to handle a chat and to wrap it up*

<a href="https://ibb.co/C7GmTgm"><img src="https://i.ibb.co/4TrK5zK/Screenshot-2024-11-07-at-17-10-59.png" alt="Screenshot-2024-11-07-at-17-10-59" border="0"></a>


```sql

WITH prep as (
SELECT 
    a.created_date AS report_date
    ,DATE(c.hire_date) AS hire_date
    , FLOOR(DATE_DIFF(a.created_date, DATE(c.hire_date), DAY) / 30) AS monthly_stage --- required to group performance metrics of agents from hire date to evaluate KPI progression
    , DATE_TRUNC(DATE(c.hire_date),MONTH) AS report_month
    , a.email AS agent_email
    , SUM(
         COALESCE(count_served_chats,0) ---count of chats
         + COALESCE(count_served_calls,0) ---count of calls
         + COALESCE(count_l1_escalated,0) --- count of escalations to the second line of support
         + COALESCE(count_l1_close,0) --- count of written inquiries
         ) AS work_items -- total work items 
    , SUM(count_served_chats) AS served_chats --# Served Chats
    , CAST(SUM(sum_handling_time_chat_sec) AS FLOAT64) AS served_chats_ht_sec --- chat handling time
    , SUM(sum_handling_time_chat_sec) / NULLIF(SUM(count_served_chats),0) / 60 AS served_chats_aht --- AVG chat handling time
    , SUM(sum_wrap_up_time_chat_sec) AS served_chats_wut_sec --- chat wrap-up time in seconds
    , SUM(sum_wrap_up_time_chat_sec) / NULLIF(SUM(count_served_chats),0) / 60 AS served_chats_awut --- AVG chat wrap up time in seconds
    , SUM(COALESCE(sum_handling_time_chat_sec,0)+COALESCE(sum_wrap_up_time_chat_sec,0)) AS served_chats_tht_sec --- total chat handling time (handling+wrap up)
    , SUM(COALESCE(sum_handling_time_chat_sec,0)+COALESCE(sum_wrap_up_time_chat_sec,0)) / NULLIF(SUM(count_served_chats),0) / 60 AS served_chats_atht --- AVG chat total handling time
    , SUM(
         COALESCE(count_l1_escalated,0)
         + COALESCE(count_l1_close,0)
         ) AS non_live_work_items --- non-live work items
    , SUM(clocked_net_sec) / 3600 AS clocked_net_hours

  FROM `agent_performance_metrics` AS a --- masked dataset
  LEFT JOIN `agent_data` c -- masked dataset
         ON a.email = c.email
  LEFT JOIN `iso_date_parameters` AS d --- masked data set
         ON  DATE(c.hire_date)=d.iso_date
  WHERE a.created_date >= '2021-06-29'
    AND a.contact_center = 'Sweden'
    and   a.team_name = 'Customer service'
    AND DATE(c.hire_date) >= '2021-06-29'
  GROUP BY 1,2,3,4,5
) 

SELECT *,
CASE 
        WHEN clocked_net_hours > 0 THEN work_items / clocked_net_hours
        ELSE 0  --- productivity of an agent
    END AS productivity
, CASE 
        WHEN AVG(clocked_net_hours) OVER (PARTITION BY agent_email) BETWEEN 0 AND 10 THEN '25'
        WHEN AVG(clocked_net_hours) OVER (PARTITION BY agent_email)  BETWEEN 10 AND 20 THEN '50'
        WHEN AVG(clocked_net_hours) OVER (PARTITION BY agent_email)  BETWEEN 20 AND 30 THEN '75'
        WHEN AVG(clocked_net_hours) OVER (PARTITION BY agent_email) >= 30 THEN '100'
        ELSE 'Other'
        END AS shift_type --- needed to see the KPI development for different shift types, as there are part-timers (25,50,75)

FROM prep
