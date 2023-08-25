# Fitness Overview

```sql composition_overview
WITH

numbers AS (
SELECT s.createdat::DATE AS ref_date,
       skeletalmusclekg AS skeletal_muscle,
       AVG(skeletalmusclekg) OVER (ROWS BETWEEN 60 PRECEDING AND 30 PRECEDING) AS skeletal_muscle_avg_30d,
       bodyfatpercent AS body_fat_percentage,
       AVG(bodyfatpercent) OVER (ROWS BETWEEN 60 PRECEDING AND 30 PRECEDING) AS body_fat_percentage_avg_30d,
  FROM health.raw.liti_scale AS s
 ORDER BY s.createdat::DATE DESC
)

SELECT ref_date,
       skeletal_muscle,
       (skeletal_muscle - skeletal_muscle_avg_30d)/skeletal_muscle AS skeletal_muscle_comparison,
       body_fat_percentage,
       body_fat_percentage - body_fat_percentage_avg_30d AS body_fat_percentage_comparison
  FROM numbers
```

Latest data from scale: <Value data={composition_overview} value=last_measurement />.

## 💡 Big numbers

<BigValue
    data={composition_overview}
    value=skeletal_muscle
    fmt="#.# kg"
    comparison=skeletal_muscle_comparison
    comparisonTitle="vs. 30d"
    comparisonFmt=pct1
/>

<BigValue
    data={composition_overview}
    value=body_fat_percentage
    title="Body Fat"
    fmt="#.#\%"
    comparison=body_fat_percentage_comparison
    comparisonTitle="vs. 30d"
    comparisonFmt="#.# p\.p\."
    downIsGood=true
/>

```workouts
SELECT *
  FROM health.raw.applehealth_workouts
```

```workout_statistics
WITH

unn AS (
SELECT w.id AS workout_id,
       w.workoutactivitytype AS workout_type,
       UNNEST(JSON_TRANSFORM(w.workoutstatistics,           
           '[{
             "type": "VARCHAR", 
             "startDate": "VARCHAR", 
             "endDate": "VARCHAR", 
             "sum": "DOUBLE",
             "minimum": "DOUBLE",
             "maximum": "DOUBLE",
             "average": "DOUBLE"
           }]')) AS s
  FROM health.raw.applehealth_workouts AS w
 WHERE w.workoutstatistics IS NOT NULL
)

SELECT workout_id, workout_type, s.*
  FROM unn
```

```workout_energy_burned
WITH

active_energy_burned AS (
SELECT ws.workout_id, 
       ws.workout_type,
       ws.sum AS total_active_energy_burned
  FROM ${workout_statistics} AS ws
 WHERE type = 'HKQuantityTypeIdentifierActiveEnergyBurned'
)

SELECT dd.range AS ref_date,
       aeb.workout_type,
       SUM(aeb.total_active_energy_burned) AS total_energy
  FROM RANGE(TODAY() - INTERVAL 30 DAY, TODAY(), INTERVAL 1 DAY) AS dd
       LEFT JOIN ${workouts} AS w
       ON w.startdate::DATE = dd.range
       LEFT JOIN active_energy_burned AS aeb
       ON aeb.workout_id = w.id
 GROUP BY 1, 2
```

## 📅 Last 30 days

### Active Calories from Exercises

<BarChart
    data={workout_energy_burned}
    x=ref_date
    y=total_energy
    series=workout_type
    legend=off
/>

```sql workout_minutes_by_day
WITH
minutes_by_day AS (
SELECT dd.range AS ref_date,
       SUM(w.duration::DOUBLE) AS total_minutes
  FROM RANGE(TODAY() - INTERVAL 60 DAY, TODAY(), INTERVAL 1 DAY) AS dd
       LEFT JOIN ${workouts} AS w
       ON w.startdate::DATE = dd.range
 GROUP BY 1
)

SELECT ref_date,
       total_minutes,
       AVG(COALESCE(total_minutes, 0)) OVER (ORDER BY ref_date ROWS BETWEEN 7 PRECEDING AND CURRENT ROW) AS moving_average_7_days,
  FROM minutes_by_day
 WHERE ref_date BETWEEN TODAY() - INTERVAL 30 DAY AND TODAY()
```

### Total Exercise Minutes

<Chart 
    data={workout_minutes_by_day} 
    x=ref_date
    yFmt=num0
>
    <Bar 
        y=total_minutes
        fillColor=#ddd
    />
    <Line 
        y=moving_average_7_days 
        name="7-day Moving Average"
    />
    <ReferenceArea 
        yMin=45
        color=green
        label="Target"
    />
</Chart>

```sql body_composition_evolution
WITH 

dd_spine AS (
SELECT *,
       s.bonemass + s.musclekg - s.skeletalmusclekg AS etc_masskg
  FROM RANGE(TODAY() - INTERVAL 90 DAY, TODAY(), INTERVAL 1 DAY) AS dd
       LEFT JOIN health.raw.liti_scale AS s
       ON s.createdat::DATE = dd.range
),

window_grouping AS (
SELECT s.range AS ref_date,
       s.bodyfatmasskg,
       SUM(CASE WHEN s.bodyfatmasskg IS NULL THEN 0 ELSE 1 END) OVER (ORDER BY s.range) AS bodyfatmasskg_id,
       s.etc_masskg,
       SUM(CASE WHEN s.etc_masskg IS NULL THEN 0 ELSE 1 END) OVER (ORDER BY s.range) AS etc_masskg_id,
       s.skeletalmusclekg,
       SUM(CASE WHEN s.skeletalmusclekg IS NULL THEN 0 ELSE 1 END) OVER (ORDER BY s.range) AS skeletalmusclekg_id,
  FROM dd_spine AS s
),

final AS (
SELECT wg.ref_date,
       FIRST_VALUE(wg.etc_masskg) OVER (PARTITION BY wg.etc_masskg_id ORDER BY wg.ref_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS total_mass,
       'Bones, etc.' AS mass_type
  FROM window_grouping AS wg
 UNION ALL
SELECT wg.ref_date,
       FIRST_VALUE(wg.bodyfatmasskg) OVER (PARTITION BY wg.bodyfatmasskg_id ORDER BY wg.ref_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS total_mass,
       'Body Fat' AS mass_type
  FROM window_grouping AS wg
 UNION ALL
SELECT wg.ref_date,
       FIRST_VALUE(wg.skeletalmusclekg) OVER (PARTITION BY wg.skeletalmusclekg_id ORDER BY wg.ref_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS total_mass,
       'Muscle' AS mass_type
  FROM window_grouping AS wg
)

SELECT * FROM final
```

### Body Composition

<Tabs>
    <Tab label="Absolute">
        <AreaChart
            data={body_composition_evolution}
            x=ref_date
            y=total_mass
            series=mass_type
        />
    </Tab>
    <Tab label="Percentage">
        <AreaChart
            data={body_composition_evolution}
            x=ref_date
            y=total_mass
            series=mass_type
            type=stacked100
            yFmt=pct0
        />
    </Tab>
</Tabs>


