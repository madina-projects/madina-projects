***Staffing Model for Age-Restricted Deliveries***

*The queries can be found in the end of the file*

This project focused on building a staffing model for a food delivery company to **optimize the availability of riders trained for age-restricted deliveries** (e.g., alcohol, tobacco, non-prescription meds). These deliveries require compliance with legal guidelines, and riders must complete specialized training before being assigned this type of order. Age-restricted products are critical for app assortment and the profitability of dark stores.

***Problem Statement:***

The Operations team previously managed rider training reactively, onboarding riders for age-restricted deliveries only after they completed a 90-day tenure. This approach did not consider actual demand or regional order volumes, leading to understaffing issues and disruptions in delivery service. Training is time-intensive, impacting the productive hours of the fleet.

***Objective:***

The goal was to develop a predictive staffing model to determine the number of riders needing training based on historical order volumes, thereby allowing the Operations team to make data-driven decisions on rider training and reduce staffing inefficiencies.

***Key Components of the Model:***

- Order Volume Metrics (Based on the last 3 months):

**avg_orders_zone_per_day:** Average daily number of age-restricted orders per delivery zone.

**avg_orders_rider_day:** Average number of age-restricted orders delivered by a rider per day for each zone.
Using these metrics, the baseline for the required number of skilled riders was calculated:

*Base riders needed = avg_order_zone_per_day/avg_orders_rider_day*

- Skilled Rider Count:

**available_skilled_riders:** The current number of trained riders in each zone, adjusted for suspended and absent riders.

- Buffer Calculation:

A buffer percentage (0.50 or 0.75) is applied to account for fluctuations in rider availability and unexpected absences. The percentage depends on the zone size:

*Riders Needed with Buffer=(Base Riders Needed×Buffer %)+Base Riders Needed*

- Demand vs. Actual Difference:

This metric highlights the staffing gap by comparing the actual number of skilled riders against the required number:

*Difference (Demand vs. Actual)=Available Skilled Riders−Riders Needed with Buffer*

If the difference is negative, it indicates an understaffed zone, signaling the need for additional rider training.

***Methodology:***

- The analysis used BigQuery for data preparation and aggregation, leveraging CTEs and JOINS to create a single, aggregated table. This approach was necessary because Looker Studio has constraints on handling multiple aggregations, so the calculations were performed directly in BigQuery.

- A tabular view with regions and zones sorted in Looker Studio provided quick insights into staffing levels. The ```
Demand_vs_actual``` column used conditional formatting to highlight negative values, indicating zones requiring immediate training interventions (See Visual 1, Query 1)

- Volumes, verticals and cancellations, which provide additional insights into the product performance, were aggregated in Looker Studio (See Visual 2, Query 2)

***Model Constraints:***

- The model does not account for last-minute absences (e.g., no-shows, sicknesses), but the use of buffer percentages mitigates this risk.

- The model relies on average metrics, which may skew results slightly. However, given the historically low variance in age-restricted order volumes, using averages is considered reliable. If order dynamics change significantly, the model can be adapted accordingly.

***Tools Used:***

- **BigQuery:** For data preparation and query execution
- **Looker Studio:** For visualization and reporting

***Impact:***

The implementation of this staffing model enabled the Operations team to:

- **Proactive Rider Training:** By understanding demand trends, the Operations team can train riders in advance, reducing training time and improving service coverage.
- **Reduced Order Cancellations:** With better staffing alignment, the model helps minimize delivery failures due to a lack of skilled riders.
- **Operational Efficiency:** The automated queries eliminate the need for manual data exports, enabling real-time decision-making and freeing up resources for other tasks.

This project has streamlined staffing processes and enhanced the company's ability to meet demand for age-restricted deliveries, ensuring a reliable and compliant service offering.

**Visual 1 - Staffing model in Looker Studio**

*Region and zone names are masked*

<a href="https://ibb.co/GvLDttC"><img src="https://i.ibb.co/PZny991/masked-stffing.png" alt="masked-stffing" border="0"></a>

**Visual 2 - Volumes and other insights**

*Order volume information is masked*

<a href="https://ibb.co/R2MMXXw"><img src="https://i.ibb.co/gyXX00K/volumes-masked.png" alt="volumes-masked" border="0"></a>

***QUERIES***

**Query 1 - Staffing model**

```sql
CREATE OR REPLACE TABLE
  `age_restricted_looker_dashboard` AS --- creating a table  used in Looker

WITH
  orders AS (
  SELECT
    DISTINCT o.platform_order_code AS order_code,
    d.rider_id,
    o.zone_id,
    ARRAY_TO_STRING(tags, "") AS order_tags, --- contains tags for age verification. requires aggregation to deduplicate the records
    entity.id,
    DATE_TRUNC(o.created_date, DAY) AS created_date
  FROM `orders` o --- masked data set
  LEFT JOIN
    UNNEST(o.deliveries) AS d
  WHERE
    o.country_code = 'se'
    AND o.created_date BETWEEN DATE_TRUNC(DATE_SUB(CURRENT_DATE(), INTERVAL 3 MONTH), MONTH) --- to include 3 full calendar months
    AND CURRENT_DATE() ), --- order prep

  skilled_riders AS (
  SELECT
    region_name,
    zone_name,
    COUNT(DISTINCT rider_id) AS skilled_riders,
    COUNT(DISTINCT
      CASE  WHEN currently_suspended IS TRUE THEN rider_id END) AS suspended_riders,
    COUNT(DISTINCT CASE WHEN absence_created IS TRUE THEN rider_id END) AS absent_riders
  FROM
    `riders`
  WHERE
    currently_active = 'ACTIVE'
    AND age_delivery_skilled = 'skilled'
  GROUP BY
    1,2 ),  --- extracts  riders who passed the compliance training to deliver age restricted goods. maps as well absent and suspended riders for future calcualtions

 age_restricted_orders AS (
  SELECT
    zone_id,
    AVG(order_count) AS avg_orders_per_zone_per_day
  FROM (
    SELECT
      zone_id,
      created_date,
      COUNT(DISTINCT
        CASE WHEN order_tags LIKE '%age_18%' THEN order_code --- to identify age restricted orders END) AS order_count
    FROM
      orders
    GROUP BY
      1, 2 ) AS daily_orders
  GROUP BY
    1 ),  --- prepares avg of 18+ order volume per zone/day from the orders table. this is a preaggregation.

  avg_per_rider AS (
  SELECT
    zone_id,
    ROUND(AVG(order_rider_day)) AS avg_rider_day
  FROM (
    SELECT
      created_date,
      rider_id,
      zone_id,
      COUNT(DISTINCT
        CASE WHEN order_tags LIKE '%age_18%' THEN order_code --- to identify age restricted orders END) AS order_rider_day
    FROM
      orders
    WHERE
      rider_id IS NOT NULL
    GROUP BY
      1,2, 3 )
  WHERE
    order_rider_day > 0
  GROUP BY
    1 ),  --- distribution of avg 18+ order per rider/day in each zone . preaggregates for the final output

  countries AS (
  SELECT
    DISTINCT cc.country_code,
    country_name,
    c.name AS city_name,
    c.id AS city_id,
    z.name AS zone_name,
    z.id AS zone_id,
    region_name
  FROM
    `fulfillment-dwh-production.curated_data_shared.countries` cc
  LEFT JOIN
    UNNEST(cities) AS c
  LEFT JOIN
    UNNEST(c.zones) AS z
  LEFT JOIN
    `foodora-bi-se.bl_performance.lg_dim_regions` r
  ON
    r.city_id = c.id
  WHERE
    cc.country_code = 'se'
) --- zone prep

SELECT
  region_name,
  zone_name,
  avg_orders_zone_per_day,
  avg_orders_rider_day,
  available_skilled_riders,
  buffer,
  ROUND(available_skilled_riders - (SAFE_DIVIDE(avg_orders_zone_per_day, GREATEST(avg_orders_rider_day, 1)) + buffer)) AS demand_vs_actual --- calculations of difference between # of actual skilled riders vs required # of skilled riders. if difference is negative the zone is understaffed and more riders should be trained
FROM 
(
  SELECT
    f1.region_name,
    f1.zone_name,
    avg_orders_zone_per_day,
    avg_orders_rider_day,
    SUM(cr.skilled_riders - suspended_riders - absent_riders) AS available_skilled_riders, -- removing from skilled riders all inactive ones
    CASE WHEN f1.zone_name IN ('Zone1', 'Zone2', 'Zone3', 'Zone4', 'Zone5', 'Zone6', 'Zone7', 'Zone8', 'Zone9', 'Zone10', 'Zone11', 'Zone12', 'Zone13', 'Zone14', 'Zone15', 'Zone16', 'Zone17') 
    THEN ROUND(SAFE_DIVIDE(avg_orders_zone_per_day, GREATEST(avg_orders_rider_day, 1)) * 0.50) 
    ELSE ROUND(SAFE_DIVIDE(avg_orders_zone_per_day, GREATEST(avg_orders_rider_day, 1)) * 0.75) 
    END AS buffer --- calculating # of buffer riders needed for each zone. 
    --- buffer % is defined by Head of Log Ops based on the zone size. zones names masked
  FROM 
  (
    SELECT
      DISTINCT region_name,
      zone_name,
      ROUND(avg_orders_per_zone_per_day) AS avg_orders_zone_per_day,
      ROUND(avg_rider_day) AS avg_orders_rider_day 
    FROM
 age_restricted_orders id
    LEFT JOIN
      countries c
    ON
      c.zone_id = id.zone_id
    LEFT JOIN
      avg_rider ar
    ON
      ar.zone_id = id.zone_id
    WHERE
      id.avg_orders_per_zone_per_day > 0 -- ensure only zones with 18+ orders are included
    GROUP BY
      1,2,3,4
    ORDER BY
     1,2,3 DESC ) f1 -- final output 1
  LEFT JOIN
    skilled_riders cr
  ON
    cr.zone_name = f1.zone_name
  WHERE
    avg_orders_zone_per_day >= 1
  GROUP BY
    1,2,3,4
  ORDER BY
    1,2,3 ASC )
```

**Query 2 - Volumes and other insights**

```sql
CREATE OR REPLACE TABLE
  `age_restricted_volumes` AS --- masked table name, used in looker dashboard 


WITH
  countries AS (
  SELECT
    DISTINCT cc.country_code,
    country_name,
    c.name AS city_name,
    c.id AS city_id,
    z.name AS zone_name,
    z.id AS zone_id,
    region_name
  FROM
    `countries` cc --- masked dataset
  LEFT JOIN
    UNNEST(cities) AS c
  LEFT JOIN
    UNNEST(c.zones) AS z
  LEFT JOIN
    UNNEST(z.starting_points) AS sp
  LEFT JOIN
    `regions` r --- masked dataset
  ON
    r.city_id = c.id
  WHERE
    cc.country_code = 'se'),

  orders AS (
  SELECT
    DISTINCT platform_order_code AS order_code,
    d.rider_id,
    o.created_date,
    zone_id,
    ARRAY_TO_STRING(tags, "","") AS order_tags, 
    vendor.vendor_code,
    vendor.name,
    vendor.vertical_type,
    entity.id,
    order_status,
    cancellation.source AS cancellation_source,
    cancellation.performed_by,
    cancellation.reason AS cancellation_reason
  FROM
    `orders` o --- masked data set
  LEFT JOIN
    UNNEST (deliveries) d
  WHERE
    country_code = 'se'
    AND o.created_date BETWEEN current_date - 180
    AND current_date)

SELECT
  DISTINCT order_code,
  region_name,
  zone_name,
  o2.created_date,
  vendor_code,
  vertical_type,
  rider_id,
  order_tags,
  order_status,
  id AS global_entity_id,
  decline_type,
  performed_by,
  decline_reason
FROM
  orders o2
LEFT JOIN
  countries c
ON
  c.zone_id = o2.zone_id
WHERE
  order_tags LIKE '%age_18%'
  AND o2.created_date BETWEEN current_date -180
  AND current_date 












