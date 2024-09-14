# IPL SQL Queries

This repository contains SQL queries for analyzing IPL matches. The dataset includes information about matches, teams, players, and deliveries. Below are the table structures and SQL queries for various analytical tasks.

## Tables

### `Matches`

| Column           | Data Type |
|------------------|-----------|
| `match_id`       | INT       |
| `season_year`    | INT       |
| `team1_id`       | INT       |
| `team2_id`       | INT       |
| `winner_team_id` | INT       |
| `venue`          | VARCHAR   |
| `date`           | DATE      |

### `Teams`

| Column      | Data Type |
|-------------|-----------|
| `team_id`   | INT       |
| `team_name` | VARCHAR   |
| `home_city` | VARCHAR   |

### `Players`

| Column        | Data Type |
|---------------|-----------|
| `player_id`   | INT       |
| `player_name` | VARCHAR   |
| `team_id`     | INT       |
| `role`        | VARCHAR   | 

### `Deliveries`

| Column         | Data Type |
|----------------|-----------|
| `delivery_id`  | INT       |
| `match_id`     | INT       |
| `inning`       | INT       |
| `bowler_id`    | INT       |
| `batsman_id`   | INT       |
| `runs_scored`  | INT       |
| `wicket`       | BOOLEAN   |

---

## Queries

### 1. Find the team with the most wins in the 2021 IPL season

```sql
WITH cte AS (
    SELECT t.team_name, COUNT(t.team_name) AS TotalWins
    FROM Matches m
    INNER JOIN Teams t ON m.winner_team_id = t.team_id
    WHERE m.season_year = 2021
    GROUP BY t.team_name
), cte2 AS (
    SELECT team_name, TotalWins, DENSE_RANK() OVER(ORDER BY TotalWins DESC) AS Drnk
    FROM cte
)
SELECT team_name, TotalWins
FROM cte2
WHERE Drnk = 1;
```

### 2. Identify the top 3 batsmen with the highest total runs across all matches

```sql
WITH cte AS (
    SELECT p.player_name AS batsman_name, SUM(runs_scored) AS TotalRuns
    FROM Deliveries d
    INNER JOIN Players p ON d.batsman_id = p.player_id
    GROUP BY p.player_name
), cte2 AS (
    SELECT batsman_name, TotalRuns, DENSE_RANK() OVER(ORDER BY TotalRuns DESC) AS Drnk
    FROM cte
)
SELECT batsman_name, TotalRuns
FROM cte2
WHERE Drnk <= 3;
```

### 3. List all the matches where the total number of wickets taken was more than 10

```sql
SELECT m.match_id, m.date, SUM(d.wicket) AS TotalWickets
FROM Matches m
INNER JOIN Deliveries d ON m.match_id = d.match_id
GROUP BY m.match_id, m.date
HAVING SUM(d.wicket) > 10;
```

### 4. Find the player with the highest average runs per match in the 2020 IPL season
```sql
WITH cte AS (
    SELECT p.player_name, m.match_id, SUM(runs_scored) AS total_runs
    FROM Deliveries d
    INNER JOIN Players p ON d.batsman_id = p.player_id
    INNER JOIN Matches m ON m.match_id = d.match_id
    WHERE m.season_year = 2020
    GROUP BY p.player_name, m.match_id
), cte2 AS (
    SELECT player_name, SUM(total_runs) AS total_runs, COUNT(DISTINCT match_id) AS matches_played,
           SUM(total_runs) / COUNT(DISTINCT match_id) AS average_runs
    FROM cte
    GROUP BY player_name
)
SELECT player_name, average_runs
FROM cte2
ORDER BY average_runs DESC
LIMIT 1;
```

### 5. Determine the venue where the highest number of wickets were taken in a single match
```sql
WITH cte AS (
    SELECT m.venue, m.match_id, SUM(d.wicket) AS TotalWickets
    FROM Matches m
    INNER JOIN Deliveries d ON m.match_id = d.match_id
    GROUP BY m.venue, m.match_id
), cte2 AS (
    SELECT venue, TotalWickets, DENSE_RANK() OVER(ORDER BY TotalWickets DESC) AS Drnk
    FROM cte
)
SELECT venue, TotalWickets
FROM cte2
WHERE Drnk = 1;
```

### 6. Find the player who has played for the most teams across all seasons
```sql
WITH cte AS (
    SELECT d.bowler_id, p.player_name, m.team1_id
    FROM Deliveries d
    INNER JOIN Matches m ON d.match_id = m.match_id
    INNER JOIN Players p ON p.player_id = d.bowler_id
    UNION ALL
    SELECT d.batsman_id, p.player_name, m.team2_id
    FROM Deliveries d
    INNER JOIN Matches m ON d.match_id = m.match_id
    INNER JOIN Players p ON p.player_id = d.batsman_id
), cte2 AS (
    SELECT player_name, bowler_id, COUNT(DISTINCT team1_id) AS NoofTeams
    FROM cte
    GROUP BY bowler_id, player_name
), cte3 AS (
    SELECT player_name, NoofTeams, DENSE_RANK() OVER(ORDER BY NoofTeams DESC) AS Drnk
    FROM cte2
)
SELECT player_name, NoofTeams
FROM cte3
WHERE Drnk = 1;
```

### 7. List the teams that have played in all IPL seasons
```sql
WITH cte AS (
    SELECT team1_id, season_year
    FROM Matches
    UNION ALL
    SELECT team2_id, season_year
    FROM Matches
)
SELECT t.team_name
FROM cte c
INNER JOIN Teams t ON c.team1_id = t.team_id
GROUP BY t.team_name
HAVING COUNT(DISTINCT c.season_year) = (SELECT COUNT(DISTINCT season_year) FROM Matches);
```

### 8. Find the top 5 batsmen with the highest strike rate in the 2021 IPL season (minimum 100 balls faced)
```sql
WITH cte AS (
    SELECT p.player_name, p.player_id, COUNT(*) AS TotalBallsFaced, SUM(d.runs_scored) AS TotalRuns,
           SUM(d.runs_scored) / COUNT(*) * 100 AS StrikeRate
    FROM Deliveries d
    INNER JOIN Players p ON d.batsman_id = p.player_id
    INNER JOIN Matches m ON m.match_id = d.match_id
    WHERE m.season_year = 2021
    GROUP BY p.player_name, p.player_id
    HAVING COUNT(*) >= 100
), cte2 AS (
    SELECT player_name, StrikeRate, DENSE_RANK() OVER(ORDER BY StrikeRate DESC) AS Drnk
    FROM cte
)
SELECT player_name, StrikeRate
FROM cte2
WHERE Drnk <= 5;
```
