# FitTrack Pro: Intelligent Fitness Analytics & Performance Monitoring System
## Overview 

This database is a comprehensive fitness and health tracking system designed to monitor user progress across biometric data, workouts, nutrition, and personal goals. It integrates multiple modules:

 •Users: Stores personal details including demographics and account info.

 •Biometrics: Logs body metrics like weight, body fat, heart rate, and muscle mass over time.

 •Exercise & Workouts: Tracks structured workout sessions, types of exercises performed, duration, weight lifted, and calories burned.

 •Nutrition: Manages food items and records meals with detailed macro- and micronutrient tracking via meals and meal-food relationships.

 •Goals: Supports multiple goal types (weight loss, muscle gain, endurance), with progress calculations and time tracking.


Advanced SQL queries provide deep insights such as:

 •Weight change trends

 •Workout consistency and performance progression

 •Nutrient intake and most consumed foods

 •Goal completion percentages and time-based progress

## Objectives 

To automate progress tracking and generate adaptive workout/nutrition recommendations based on historical user data and goal targets.

## Creating Database 
``` sql
CREATE DATABASE MD13thJ6_db;
USE MD13thJ6_db;
```
## Creating Tables
### Table:users
``` sql
CREATE TABLE users(
    user_id       INT PRIMARY KEY AUTO_INCREMENT,
    username      TEXT,
    email         TEXT,
    password_hash TEXT,
    first_name    TEXT,
    last_name     TEXT,
    birth_date    DATE,
    gender        TEXT,
    created_at    DECIMAL(10,2)
);

SELECT * FROM users ;
```
### Table:biometrics
``` sql
CREATE TABLE biometrics(
    biometric_id                INT PRIMARY KEY AUTO_INCREMENT,
    user_id                     INT,
    record_date                 DATE,
    weight_kg                   DECIMAL(10,2), 
    body_fat_percent            DECIMAL(10,2),
    muscle_mass_kg              DECIMAL(10,2),
    resting_heart_rate          DECIMAL(10,2),
    blood_pressure_systolic     DECIMAL(10,2), 
    blood_pressure_diastolic    DECIMAL(10,2), 
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

SELECT * FROM biometrics ;
```
### Table:exercise_types
``` sql
CREATE TABLE exercise_types(
    exercise_type_id           INT PRIMARY KEY AUTO_INCREMENT,
    name                       TEXT,
    category                   TEXT,
    description                TEXT, 
    calories_burned_per_min    DECIMAL(10,2)
);

SELECT * FROM exercise_types ;
```
### Table:workouts
``` sql
CREATE TABLE workouts(
    workout_id             INT PRIMARY KEY AUTO_INCREMENT,
    user_id                INT,
    workout_date           DATE,
    start_time             TIME,
    end_time               TIME,
    notes                  TEXT,
    total_calories_burned  DECIMAL(10,2),
    perceived_exertion     DECIMAL(10,2),
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

SELECT * FROM workouts ;
```
### Table:workout_exercises
``` sql
CREATE TABLE workout_exercises(
    workout_exercise_id        INT PRIMARY KEY AUTO_INCREMENT,
    workout_id                 INT,
    exercise_type_id           INT,
    sets                       INT,
    reps                       INT,
    weight_kg                  DECIMAL(10,2),
    duration_minutes           DECIMAL(10,2),
    distance_km                DECIMAL(10,2),
    calories_burned            DECIMAL(10,2),
    FOREIGN KEY (workout_id) REFERENCES workouts(workout_id),
    FOREIGN KEY (exercise_type_id) REFERENCES exercise_types(exercise_type_id)
);

SELECT * FROM workout_exercises ;
```
### Table:food_items
``` sql
CREATE TABLE food_items(
    food_id               INT PRIMARY KEY AUTO_INCREMENT,
    name                  TEXT,
    brand                 TEXT,
    serving_size          TEXT,
    calories_per_serving  DECIMAL(10,2),
    protein_g             DECIMAL(10,2),
    carbs_g               DECIMAL(10,2),
    fats_g                DECIMAL(10,2)
);

SELECT * FROM food_items ;
```
### Table:meals
``` sql
CREATE TABLE meals(
    meal_id           INT PRIMARY KEY AUTO_INCREMENT,
    user_id           INT,
    meal_date         DATE,
    meal_time         TIME,
    meal_type         TEXT,
    total_calories    DECIMAL(10,2),
    protein_g         DECIMAL(10,2),
    carbs_g           DECIMAL(10,2),
    fats_g            DECIMAL(10,2),
    notes             TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

SELECT * FROM meals ;
```
### Table:meal_foods
``` sql
CREATE TABLE meal_foods(
    meal_food_id   INT PRIMARY KEY AUTO_INCREMENT,
    meal_id        INT,
    food_id        INT,
    servings       DECIMAL(10,2),
    FOREIGN KEY (meal_id) REFERENCES meals(meal_id),
    FOREIGN KEY (food_id) REFERENCES food_items(food_id)
);

SELECT * FROM meal_foods ;
```
### Table:goals
``` sql
CREATE TABLE goals(
    goal_id         INT PRIMARY KEY AUTO_INCREMENT,
    user_id         INT,
    goal_type       TEXT,
    target_value    DECIMAL(10,2),
    target_date     DATE,
    current_value   DECIMAL(10,2),
    start_date      DATE, 
    is_completed    BOOLEAN,
    notes           TEXT,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);

SELECT * FROM goals ;
```
## KEY Queries 

#### 1-List all users with their age (calculated from birth date) and BMI (using the most recent weight record).
``` sql
SELECT 
    u.username,
    u.gender,
    TIMESTAMPDIFF(YEAR, u.birth_date, CURDATE()) AS age,
    u.created_at AS Height_cm,
    b.weight_kg,
    ROUND(b.weight_kg / POWER(u.created_at / 100, 2), 2) AS BMI
FROM users u
JOIN (
    SELECT b1.*
    FROM biometrics b1
    JOIN (
        SELECT user_id, MAX(record_date) AS latest_date
        FROM biometrics
        GROUP BY user_id
    ) b2 ON b1.user_id = b2.user_id AND b1.record_date = b2.latest_date
) b ON u.user_id = b.user_id;
```
#### 2-Show the gender distribution of users in the database.
``` sql
SELECT 
    gender,
    COUNT(*) AS user_count,
    ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM users), 2) AS percentage
FROM users
GROUP BY gender;
```
#### 3-Display the weight change over time for a specific user.
``` sql
SELECT 
    u.username,
    b.record_date,
    b.weight_kg,
    ROUND(b.weight_kg - LAG(b.weight_kg) OVER (PARTITION BY b.user_id ORDER BY b.record_date), 2) AS weight_change
FROM biometrics b
JOIN users u ON u.user_id = b.user_id
WHERE b.user_id = 1  -- ← Change to desired user
ORDER BY b.record_date;
```
#### 4-Find users whose resting heart rate decreased by more than 5 bpm over a 30-day period.
``` sql
SELECT 
    u.username,
    first.record_date AS start_date,
    first.resting_heart_rate AS starting_hr,
    last.record_date AS end_date,
    last.resting_heart_rate AS ending_hr,
    (first.resting_heart_rate - last.resting_heart_rate) AS hr_drop,
    DATEDIFF(last.record_date, first.record_date) AS Day_tracked
FROM users u
JOIN (
    SELECT user_id, record_date, resting_heart_rate
    FROM biometrics
    WHERE (user_id, record_date) IN (
        SELECT user_id, MIN(record_date)
        FROM biometrics
        GROUP BY user_id
    )
) first ON u.user_id = first.user_id
JOIN (
    SELECT user_id, record_date, resting_heart_rate
    FROM biometrics
    WHERE (user_id, record_date) IN (
        SELECT user_id, MAX(record_date)
        FROM biometrics
        GROUP BY user_id
    )
) last ON u.user_id = last.user_id
WHERE DATEDIFF(last.record_date, first.record_date) >= 30
  AND (first.resting_heart_rate - last.resting_heart_rate) > 1;
```
#### 5-Calculate the average body fat percentage by gender.
``` sql
SELECT 
    u.gender,
    ROUND(AVG(b.body_fat_percent), 2) AS avg_body_fat_percent
FROM biometrics b
JOIN users u ON u.user_id = b.user_id
GROUP BY u.gender;
```
#### 7-Find the most popular exercise types among all users.
``` sql
SELECT 
    et.name AS exercise_name,
    et.category,
    COUNT(we.workout_exercise_id) AS usage_count
FROM workout_exercises we
JOIN exercise_types et ON we.exercise_type_id = et.exercise_type_id
GROUP BY et.exercise_type_id, et.name, et.category
ORDER BY usage_count DESC
LIMIT 3;
```
#### 8-Show users who consistently workout more than 4 times per week.
``` sql
SELECT 
    u.user_id,
    u.username,
    COUNT(DISTINCT CONCAT(YEAR(w.workout_date), '-', WEEK(w.workout_date))) AS active_weeks,
    MAX(weekly_count) AS max_weekly_workouts
FROM (
    SELECT 
        user_id,
        YEAR(workout_date) AS yr,
        WEEK(workout_date) AS wk,
        COUNT(*) AS weekly_count
    FROM workouts
    GROUP BY user_id, YEAR(workout_date), WEEK(workout_date)
    HAVING COUNT(*) > 4
) AS weekly_summary
JOIN users u ON u.user_id = weekly_summary.user_id
JOIN workouts w ON w.user_id = u.user_id
GROUP BY u.user_id, u.username
HAVING COUNT(DISTINCT CONCAT(YEAR(w.workout_date), '-', WEEK(w.workout_date))) >= 2
ORDER BY active_weeks DESC;
```
#### 10-Display the progression of maximum weight lifted for strength exercises (like Deadlifts) for each user.
``` sql
WITH strength_logs AS (
    SELECT 
        u.username,
        et.name AS exercise_name,
        w.workout_date,
        we.weight_kg,
        ROW_NUMBER() OVER (PARTITION BY u.user_id, et.name ORDER BY w.workout_date ASC) AS rn_first,
        ROW_NUMBER() OVER (PARTITION BY u.user_id, et.name ORDER BY w.workout_date DESC) AS rn_last
    FROM workout_exercises we
    JOIN workouts w ON we.workout_id = w.workout_id
    JOIN exercise_types et ON we.exercise_type_id = et.exercise_type_id
    JOIN users u ON w.user_id = u.user_id
    WHERE et.category = 'Strength' AND we.weight_kg IS NOT NULL
),
first_lifts AS (
    SELECT 
        username,
        exercise_name,
        workout_date AS first_date,
        weight_kg AS first_weight
    FROM strength_logs
    WHERE rn_first = 1
),
latest_lifts AS (
    SELECT 
        username,
        exercise_name,
        workout_date AS latest_date,
        weight_kg AS latest_weight
    FROM strength_logs
    WHERE rn_last = 1
)
SELECT 
    f.username,
    f.exercise_name,
    f.first_date,
    f.first_weight,
    l.latest_date,
    l.latest_weight,
    ROUND(l.latest_weight - f.first_weight, 2) AS progress_kg,
    ROUND((l.latest_weight - f.first_weight) * 100.0 / NULLIF(f.first_weight, 0), 2) AS progress_percent,
    DATEDIFF(l.latest_date, f.first_date) AS time_period_days
FROM first_lifts f
JOIN latest_lifts l 
  ON f.username = l.username AND f.exercise_name = l.exercise_name
ORDER BY progress_percent DESC;
```
#### 11-Find users who improved their running duration by more than 20% over 3 months.
``` sql
WITH running_logs AS (
    SELECT 
        u.user_id,
        u.username,
        w.workout_date,
        we.duration_minutes,
        ROW_NUMBER() OVER (PARTITION BY u.user_id ORDER BY w.workout_date ASC) AS rn_asc,
        ROW_NUMBER() OVER (PARTITION BY u.user_id ORDER BY w.workout_date DESC) AS rn_desc
    FROM workout_exercises we
    JOIN workouts w ON we.workout_id = w.workout_id
    JOIN exercise_types et ON we.exercise_type_id = et.exercise_type_id
    JOIN users u ON u.user_id = w.user_id
    WHERE et.name = 'Running' AND we.duration_minutes IS NOT NULL
)
SELECT 
    early.user_id,
    early.username,
    early.workout_date AS start_date,
    early.duration_minutes AS start_duration,
    late.workout_date AS end_date,
    late.duration_minutes AS end_duration,
    ROUND((late.duration_minutes - early.duration_minutes) / early.duration_minutes * 100, 2) AS percent_improvement
FROM running_logs early
JOIN running_logs late 
  ON early.user_id = late.user_id 
  AND early.rn_asc = 1 
  AND late.rn_desc = 1
WHERE DATEDIFF(late.workout_date, early.workout_date) >= 90
  AND (late.duration_minutes - early.duration_minutes) / early.duration_minutes > 0.20;
```
#### 13-List all meals for a user with their macro nutrient breakdown (protein, carbs, fats).
``` sql
SELECT 
        meal_date,meal_type,total_calories,protein_g,carbs_g,fats_g
FROM meals 
WHERE user_id=1 -- ← Change this to desired user ID
ORDER BY meal_date;
```
#### 14-Find the most commonly consumed food items across all users.
``` sql
SELECT 
    fi.name AS food_name,
    fi.brand,
    COUNT(*) AS times_consumed
FROM meal_foods mf
JOIN food_items fi ON mf.food_id = fi.food_id
GROUP BY fi.name, fi.brand
ORDER BY times_consumed DESC
LIMIT 3;
```
#### 15-Calculate the average daily calorie intake by user.
``` sql
SELECT 
    u.username,
    ROUND(AVG(daily_calories), 2) AS avg_daily_calories
FROM (
    SELECT 
        user_id,
        meal_date,
        SUM(total_calories) AS daily_calories
    FROM meals
    GROUP BY user_id, meal_date
) AS daily_totals
JOIN users u ON u.user_id = daily_totals.user_id
GROUP BY u.user_id, u.username
ORDER BY avg_daily_calories DESC;
```
#### 17-List all active goals with their current progress percentage.
``` sql
SELECT 
    u.username,
    g.goal_type,
    g.target_date,
    ROUND(
        CASE 
            WHEN g.goal_type = 'Weight Loss' AND g.current_value IS NOT NULL AND g.target_value IS NOT NULL THEN
                (b.weight_kg - g.current_value) * 100.0 / NULLIF(b.weight_kg - g.target_value, 0)

            WHEN g.goal_type = 'Muscle Gain' AND g.current_value IS NOT NULL AND g.target_value IS NOT NULL THEN
                (g.current_value - b.muscle_mass_kg) * 100.0 / NULLIF(g.target_value - b.muscle_mass_kg, 0)

            WHEN g.goal_type = 'Endurance' AND g.current_value IS NOT NULL AND g.target_value IS NOT NULL THEN
                (g.current_value - b.resting_heart_rate) * 100.0 / NULLIF(g.target_value - b.resting_heart_rate, 0)

            ELSE NULL
        END
    , 2) AS progress_percent
FROM goals g
JOIN users u ON g.user_id = u.user_id
JOIN biometrics b ON b.user_id = g.user_id AND b.record_date = g.start_date
WHERE g.is_completed IS NULL OR g.is_completed = FALSE;
```
#### 18-Find users who achieved at least 80% of their weight loss goals.
``` sql
SELECT 
    u.username,
    ROUND(start_weight - g.target_value, 2) AS target_loss,
    ROUND(start_weight - g.current_value, 2) AS achieved_loss,
    ROUND(
        (start_weight - g.current_value) * 100.0 / NULLIF(start_weight - g.target_value, 0), 2
    ) AS completion_percentage
FROM goals g
JOIN users u ON g.user_id = u.user_id
JOIN (
    SELECT user_id, record_date, weight_kg AS start_weight
    FROM biometrics
) b ON g.user_id = b.user_id AND b.record_date = g.start_date
WHERE g.goal_type = 'Weight Loss'
  AND g.current_value IS NOT NULL
  AND g.target_value IS NOT NULL
  AND (start_weight - g.current_value) * 100.0 / NULLIF(start_weight - g.target_value, 0) >= 80;
```
#### 20-Correlate workout frequency with weight loss progress for users with weight loss goals.
``` sql
WITH weight_loss_goals AS (
    SELECT 
        g.goal_id,
        g.user_id,
        g.start_date,
        g.target_date,
        g.current_value AS current_weight,
        g.target_value,
        u.username,
        b.weight_kg AS start_weight,
        DATEDIFF(g.target_date, g.start_date) / 7.0 AS duration_weeks
    FROM goals g
    JOIN users u ON g.user_id = u.user_id
    JOIN biometrics b ON b.user_id = g.user_id AND b.record_date = g.start_date
    WHERE g.goal_type = 'Weight Loss'
      AND g.current_value IS NOT NULL
      AND g.target_value IS NOT NULL
),
workouts_during_goal AS (
    SELECT 
        w.user_id,
        COUNT(*) AS total_workouts
    FROM workouts w
    JOIN weight_loss_goals g ON g.user_id = w.user_id
    WHERE w.workout_date BETWEEN g.start_date AND g.target_date
    GROUP BY w.user_id
)
SELECT 
    g.username,
    ROUND(w.total_workouts / g.duration_weeks, 2) AS workouts_per_week,
    ROUND(g.start_weight - g.current_weight, 2) AS weight_loss
FROM weight_loss_goals g
LEFT JOIN workouts_during_goal w ON g.user_id = w.user_id;
```
## Conclusion 

This relational database offers a holistic view of a user's fitness journey, enabling data-driven decisions and personalized health strategies. With its modular design and rich data relationships, it's well-suited for integration with mobile apps, dashboards, or AI-powered recommendations. The structure supports flexible goal setting, performance monitoring, and nutrition planning, making it ideal for fitness platforms, personal trainers, or wellness programs.

