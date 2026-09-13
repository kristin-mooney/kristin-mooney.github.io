---
layout: page
title: "Colorado Broadband — SQL Queries"
permalink: /colorado-broadband-sql/
---

All queries run against a MySQL database named `fiber_analytics`.
Source tables: `fcc_co_fiber`, `fcc_co_cable`, `telco_churn`.

---

## Dashboard 1 — Colorado's Broadband Landscape
### tech_overview

```sql
USE fiber_analytics;

select * from fcc_co_fiber limit 10;

select * from fcc_co_cable limit 10;

SELECT
    CASE technology
        WHEN 50 THEN 'Fiber (FTTP)'
        WHEN 40 THEN 'Cable (HFC)'
    END                                                          AS technology_type,
    COUNT(DISTINCT provider_id)                                  AS num_providers,
    COUNT(DISTINCT location_id)                                  AS locations_served,
    ROUND(AVG(max_advertised_download_speed), 0)                 AS avg_download_mbps,
    ROUND(AVG(max_advertised_upload_speed), 0)                   AS avg_upload_mbps
FROM (
    SELECT technology, provider_id, location_id,
           max_advertised_download_speed, max_advertised_upload_speed
    FROM fcc_co_fiber
    UNION ALL
    SELECT technology, provider_id, location_id,
           max_advertised_download_speed, max_advertised_upload_speed
    FROM fcc_co_cable
) AS combined
GROUP BY technology;
```

---

## Dashboard 2 — Competitive Battleground
### fiber_and_cable_competitors

```sql
USE fiber_analytics;

SELECT
    'Fiber'                                                      AS service,
    brand_name,
    COUNT(DISTINCT location_id)                                  AS locations,
    ROUND(AVG(max_advertised_download_speed), 0)                 AS avg_download_mbps,
    ROUND(AVG(max_advertised_upload_speed), 0)                   AS avg_upload_mbps
FROM fcc_co_fiber
GROUP BY brand_name

UNION ALL

SELECT
    'Cable'                                                      AS service,
    brand_name,
    COUNT(DISTINCT location_id)                                  AS locations,
    ROUND(AVG(max_advertised_download_speed), 0)                 AS avg_download_mbps,
    ROUND(AVG(max_advertised_upload_speed), 0)                   AS avg_upload_mbps
FROM fcc_co_cable
GROUP BY brand_name;
```

---

## Dashboard 3 — Expansion Strategy in CO
### expansion_priority

```sql
USE fiber_analytics;

SELECT
    county_name,
    county_fips,
    cable_only_locations,
    pct_cable_only,
    xfinity_avg_upload_mbps
FROM (
    SELECT 'Adams'      AS county_name, '001' AS county_fips, 67800 AS cable_only_locations, 42.4 AS pct_cable_only, 531.0 AS xfinity_avg_upload_mbps
    UNION ALL SELECT 'Douglas',         '035',                67444,                          57.3,                  150.0
    UNION ALL SELECT 'Broomfield',      '014',                14199,                          63.3,                  191.0
    UNION ALL SELECT 'Eagle',           '037',                15297,                          85.4,                  400.0
    UNION ALL SELECT 'Summit',          '117',                14047,                          89.4,                   38.0
    UNION ALL SELECT 'Garfield',        '045',                13230,                          78.1,                   49.0
    UNION ALL SELECT 'Pitkin',          '097',                 7268,                          97.1,                   89.0
    UNION ALL SELECT 'Fremont',         '043',                 9500,                          65.0,                 1000.0
) AS county_data
ORDER BY county_name ASC;
```

### cable_incumbents

```sql
USE fiber_analytics;

SELECT
    CASE SUBSTRING(block_geoid, 3, 3)
        WHEN '001' THEN 'Adams'       WHEN '014' THEN 'Broomfield'
        WHEN '035' THEN 'Douglas'     WHEN '037' THEN 'Eagle'
        WHEN '043' THEN 'Fremont'     WHEN '045' THEN 'Garfield'
        WHEN '097' THEN 'Pitkin'      WHEN '117' THEN 'Summit'
    END                                                          AS county_name,
    SUBSTRING(block_geoid, 3, 3)                                 AS county_fips,
    brand_name                                                   AS cable_provider,
    COUNT(DISTINCT location_id)                                  AS locations_served,
    ROUND(AVG(max_advertised_download_speed), 0)                 AS avg_download_mbps,
    ROUND(AVG(max_advertised_upload_speed), 0)                   AS avg_upload_mbps
FROM fcc_co_cable
WHERE SUBSTRING(block_geoid, 3, 3) IN ('001','014','035','037','043','045','097','117')
GROUP BY county_fips,
brand_name
ORDER BY county_fips,
locations_served DESC;
```

### county_competition

```sql
USE fiber_analytics;

SELECT
    CASE SUBSTRING(block_geoid, 3, 3)
        WHEN '001' THEN 'Adams'          WHEN '003' THEN 'Alamosa'
        WHEN '005' THEN 'Arapahoe'       WHEN '007' THEN 'Archuleta'
        WHEN '009' THEN 'Baca'           WHEN '011' THEN 'Bent'
        WHEN '013' THEN 'Boulder'        WHEN '014' THEN 'Broomfield'
        WHEN '015' THEN 'Chaffee'        WHEN '017' THEN 'Cheyenne'
        WHEN '019' THEN 'Clear Creek'    WHEN '021' THEN 'Conejos'
        WHEN '023' THEN 'Costilla'       WHEN '025' THEN 'Crowley'
        WHEN '027' THEN 'Custer'         WHEN '029' THEN 'Delta'
        WHEN '031' THEN 'Denver'         WHEN '033' THEN 'Dolores'
        WHEN '035' THEN 'Douglas'        WHEN '037' THEN 'Eagle'
        WHEN '039' THEN 'Elbert'         WHEN '041' THEN 'El Paso'
        WHEN '043' THEN 'Fremont'        WHEN '045' THEN 'Garfield'
        WHEN '047' THEN 'Gilpin'         WHEN '049' THEN 'Grand'
        WHEN '051' THEN 'Gunnison'       WHEN '053' THEN 'Hinsdale'
        WHEN '055' THEN 'Huerfano'       WHEN '057' THEN 'Jackson'
        WHEN '059' THEN 'Jefferson'      WHEN '061' THEN 'Kiowa'
        WHEN '063' THEN 'Kit Carson'     WHEN '065' THEN 'Lake'
        WHEN '067' THEN 'La Plata'       WHEN '069' THEN 'Larimer'
        WHEN '071' THEN 'Las Animas'     WHEN '073' THEN 'Lincoln'
        WHEN '075' THEN 'Logan'          WHEN '077' THEN 'Mesa'
        WHEN '079' THEN 'Mineral'        WHEN '081' THEN 'Moffat'
        WHEN '083' THEN 'Montezuma'      WHEN '085' THEN 'Montrose'
        WHEN '087' THEN 'Morgan'         WHEN '089' THEN 'Otero'
        WHEN '091' THEN 'Ouray'          WHEN '093' THEN 'Park'
        WHEN '095' THEN 'Phillips'       WHEN '097' THEN 'Pitkin'
        WHEN '099' THEN 'Prowers'        WHEN '101' THEN 'Pueblo'
        WHEN '103' THEN 'Rio Blanco'     WHEN '105' THEN 'Rio Grande'
        WHEN '107' THEN 'Routt'          WHEN '109' THEN 'Saguache'
        WHEN '111' THEN 'San Juan'       WHEN '113' THEN 'San Miguel'
        WHEN '115' THEN 'Sedgwick'       WHEN '117' THEN 'Summit'
        WHEN '119' THEN 'Teller'         WHEN '121' THEN 'Washington'
        WHEN '123' THEN 'Weld'           WHEN '125' THEN 'Yuma'
        ELSE 'Other'
    END                                                                              AS county_name,
    SUBSTRING(block_geoid, 3, 3)                                                     AS county_fips,
    'Colorado'                                                                       AS state,
    COUNT(*)                                                                         AS total_locations,
    SUM(CASE WHEN has_fiber = 1 AND has_cable = 1 THEN 1 ELSE 0 END)               AS fiber_and_cable,
    SUM(CASE WHEN has_fiber = 1 AND has_cable = 0 THEN 1 ELSE 0 END)               AS fiber_only,
    SUM(CASE WHEN has_fiber = 0 AND has_cable = 1 THEN 1 ELSE 0 END)               AS cable_only,
    ROUND(SUM(CASE WHEN has_fiber = 0 AND has_cable = 1 THEN 1 ELSE 0 END)
          * 100.0 / COUNT(*), 1)                                                     AS pct_cable_only,
    ROUND(SUM(CASE WHEN has_fiber = 1 THEN 1 ELSE 0 END)
          * 100.0 / COUNT(*), 1)                                                     AS pct_has_fiber
FROM (
    SELECT
        a.location_id,
        a.block_geoid,
        CASE WHEN f.location_id IS NOT NULL THEN 1 ELSE 0 END AS has_fiber,
        CASE WHEN c.location_id IS NOT NULL THEN 1 ELSE 0 END AS has_cable
    FROM (
        SELECT location_id, block_geoid FROM fcc_co_fiber
        UNION
        SELECT location_id, block_geoid FROM fcc_co_cable
    ) AS a
    LEFT JOIN (SELECT DISTINCT location_id FROM fcc_co_fiber) AS f
        ON a.location_id = f.location_id
    LEFT JOIN (SELECT DISTINCT location_id FROM fcc_co_cable) AS c
        ON a.location_id = c.location_id
) AS competition
GROUP BY county_fips
ORDER BY pct_cable_only DESC;
```

---

## Dashboard 4 — Retention & Risk Strategy
### telco_churn

```sql
USE fiber_analytics;

SELECT
    InternetService,
    tenure,
    gender,
    partner,
    contract,
    paymentmethod,
    churn,
    COUNT(*)                                                        AS total_customers,
    SUM(CASE WHEN Churn = 'Yes' THEN 1 ELSE 0 END)                 AS churned,
    ROUND(SUM(CASE WHEN Churn = 'Yes' THEN 1 ELSE 0 END)
          * 100.0 / COUNT(*), 1)                                    AS churn_rate_pct,
    ROUND(AVG(tenure), 1)                                           AS avg_tenure_months,
    ROUND(AVG(MonthlyCharges), 2)                                   AS avg_monthly_charge,
    ROUND(SUM(CASE WHEN Churn = 'Yes' THEN 1 ELSE 0 END)
          * AVG(MonthlyCharges), 0)                                 AS est_monthly_revenue_lost
FROM telco_churn
GROUP BY InternetService,
tenure,
gender,
partner,
contract,
paymentmethod,
churn
ORDER BY churn_rate_pct DESC;
```
