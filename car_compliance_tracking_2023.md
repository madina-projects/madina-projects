***Car Compliance Tracking in Food Delivery Operations***

*The query can be found in the end of the file*

***Objective***: 
  
To ensure legal compliance in a food delivery company by identifying instances where multiple riders use the same vehicle plate number but operate different vehicles during overlapping shifts. This is a key compliance measure, as per commercial law, each car must be registered for delivery services, and unauthorized sharing of vehicle plates is prohibited.

***Problem:*** 

Riders were found to be sharing vehicle plates across different cars, leading to non-compliance with commercial regulations. This issue needed a systematic process to identify and address unauthorized usage of vehicle plates during delivery shifts.

***Solution:***

Using BigQuery, I developed a SQL query that tracks overlapping shifts for riders using the same vehicle plate. The query utilizes LAG and LEAD window functions to analyze shift start and end times, partitioned by the vehicle plate number. This approach helps in pinpointing cases where two or more riders have shifts at the same time using the same vehicle plate, indicating potential misuse.

***Key Components:***

**Shift Data:** Start and end times of each rider's shift.

**Vehicle Plate Number:** Registered vehicle plate used for deliveries.

**Window Functions:**
LAG and LEAD functions are applied to detect overlapping shifts with the same vehicle plate.

The query filters out instances where riders share the same vehicle plate and have overlapping shifts. The output highlights these cases, enabling the Operations team to take appropriate actions, such as providing feedback, issuing warnings, suspending, or terminating the rider, depending on the frequency of the violations.

***Impact:***

This project improved regulatory compliance and reduced the risk of legal repercussions associated with unregistered vehicle usage. The proactive identification of non-compliant riders helped maintain operational integrity and standardize vehicle usage across the delivery fleet.

- Enabled daily monitoring and identification of non-compliant vehicle usage.
- Streamlined compliance checks, reducing manual effort and improving regulatory adherence.
- Enhanced overall fleet compliance, minimizing legal risks for the company.

***Tools:***

**BigQuery:** For querying and analyzing the data.

**Sheets**: For exporting query results and monitoring flagged instances. The query is connected to the sheets and is updated automatically

```sql
WITH
  shifts AS (
  SELECT
    *
  FROM (
    SELECT
      created_date,
      s.rider_id,
      ri.rider_name,
      city_id,
      CAST(FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S',actual_start_at, 'Europe/Stockholm') AS datetime) AS actual_start,
      CAST(FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%S',actual_end_at, 'Europe/Stockholm') AS datetime) AS actual_end
    FROM
      `shifts` s. --- masked dataset
    LEFT JOIN
      `riders` ri --- masked dataset
    ON
      s.rider_id = ri.rider_id
    WHERE
      ri.country_code = 'se'
      AND s.country_code = 'se'
      AND s.shift_state = 'COMPLETED')
  WHERE
    created_date >= current_date - 14
  ORDER BY
    1 ASC), --- worked shift prep

  active AS (
  SELECT
    *
  FROM (
    SELECT
      DISTINCT r.rider_id,
      DATETIME(c.start_at,'Europe/Stockholm') AS contract_start,
      DATETIME(c.end_at,'Europe/Stockholm') AS contract_end,
      CASE
        WHEN c.end_at IS NULL THEN 'INACTIVE'
        WHEN DATETIME(c.end_at) >= current_date THEN 'ACTIVE'
        ELSE 'INACTIVE'
    END AS currently_active,
      DENSE_RANK() OVER(PARTITION BY r.rider_id ORDER BY c.updated_at DESC) rank
    FROM
      `riders` r --- masked dataset
    LEFT JOIN
      UNNEST (contracts)c
    WHERE
      country_code = 'se')
  WHERE
    rank=1), ---active riders prep

  car_owners AS (
  SELECT
    *
  FROM (
    SELECT
      *,
      COUNT(rider_id) OVER(PARTITION BY plate ORDER BY plate) rider_count
    FROM (
      SELECT
        r.rider_id,
        email,
        c.value AS plate,
        COUNT(*) OVER (PARTITION BY c.value ORDER BY c.value) count_registrations,
        currently_active
      FROM
        `riders` r --- masked dataset
      LEFT JOIN
        UNNEST (custom_fields) c
      LEFT JOIN
        active a
      ON
        a.rider_id = r.rider_id
      WHERE
        c.name = 'vehicle_plate_number'
        AND country_code = 'se'
      GROUP BY
        1,2,3,5)
    WHERE
      count_registrations > 1
      AND currently_active = 'ACTIVE')
  WHERE
    rider_count >1), -- to exclude inactive riders having the same plate with active


  cities AS (
  SELECT
    region_name,
    cn.city_id,
    cn.city_name AS city_name
  FROM (
    SELECT
      cp.id AS city_id,
      cp.name AS city_name
    FROM
      `countries` --- masked dataset
    LEFT JOIN
      UNNEST(cities) AS cp
    WHERE
      country_code = 'se') cn
  LEFT JOIN
  `regions` rg ---masked dataset
  ON
    cn.city_id =rg.city_id)

SELECT
  created_date,
  rider_id,
  rider_name,
  email,
  plate,
  actual_start,
  actual_end,
  city_name,
  EXTRACT (week
  FROM (DATE_TRUNC(created_date, isoweek))) AS week,
  region_name,
    count_plates_day
  FROM (
        SELECT
          created_date,
          rider_id,
          rider_name,
          email,
          plate,
          actual_start,
          actual_end,
          CASE
         WHEN (actual_start BETWEEN (LAG(actual_start,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)) AND (LAG(actual_end,1) OVER (PARTITION BY plate ORDER BY actual_start ASC))) 
            OR (actual_end BETWEEN (LAG(actual_start,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)) AND (LAG(actual_end,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)))
             OR (actual_start < (LAG(actual_start,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)) AND actual_end > (LAG(actual_end,1) OVER (PARTITION BY plate ORDER BY actual_start ASC))) 
             OR (actual_start >(LAG(actual_start,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)) AND actual_end < (LAG(actual_end,1) OVER (PARTITION BY plate ORDER BY actual_start ASC))) THEN 'yes'
        WHEN (LAG(actual_start,1) OVER (PARTITION BY plate ORDER BY actual_start ASC)) IS NULL THEN 'no overlap'
            ELSE 'no overlap'
        END
          AS shift_overlap,
          region_name,
          city_name,
          COUNT(*) OVER (PARTITION BY plate, created_date) AS count_plates_day
        FROM (
          SELECT
            s.created_date,
            o.rider_id,
            rider_name,
            email,
            plate,
            actual_start,
            actual_end,
            region_name,
            city_name
          FROM
            owners o
          LEFT JOIN
            shifts s
          ON
            o.rider_id=s.rider_id
          LEFT JOIN
            cities c
          ON
            c.city_id = s.city_id
          WHERE
            created_date >= current_date - 14
            AND count_registrations > 1
            AND s.created_date >= current_date - 14))
  WHERE
    count_plates_day >1)-- to exclude same riders whose start and end are the same
WHERE
ORDER BY
  1,5, 6 ASC









ChatGPT can make mistakes. C
