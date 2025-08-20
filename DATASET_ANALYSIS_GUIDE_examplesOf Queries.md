# MKR - Mining Kubernetes Repositories: The Cloud was Not Built in a Day

Giuseppe Destefanis, Silvia Bartolucci, Daniel Feitosa

MSR - Mining Challenge


## 📊 QL Query Examples for Research and Analysis

This guide demonstrates the analytical potential of MKR through progressively complex SQL queries. 

---

## 🗄️ Database Schema

### Table Overview
- **`comments`** - 1,795,423 rows: Issue and PR comments
- **`commits`** - 130,832 rows: Git commit data
- **`issues`** - 46,768 rows: GitHub issues
- **`pull_requests`** - 83,368 rows: GitHub pull requests

### Key Relationships
- `comments.target_id` → `issues.issue_id` OR `pull_requests.pr_id`
- `author_name` and `author_id` fields link the same person across all tables
- All personally identifiable information has been anonymized

---

## 🟢 **Level 1: Basic Queries**

### 1.1 Simple Counts and Distributions

```sql
-- Total activity overview
SELECT 'Comments' as activity_type, COUNT(*) as count FROM comments
UNION ALL
SELECT 'Commits', COUNT(*) FROM commits
UNION ALL  
SELECT 'Issues', COUNT(*) FROM issues
UNION ALL
SELECT 'Pull Requests', COUNT(*) FROM pull_requests;
```

```sql
-- Bot vs Human activity distribution
SELECT 
  CASE 
    WHEN author_name LIKE 'bot_%' THEN 'Bot'
    ELSE 'Human'
  END as contributor_type,
  COUNT(*) as comment_count
FROM comments 
GROUP BY contributor_type;
```

### 1.2 Basic Time Analysis

```sql
-- Activity by year
SELECT 
  strftime('%Y', created_at) as year,
  COUNT(*) as issues_created
FROM issues 
WHERE created_at IS NOT NULL
GROUP BY year 
ORDER BY year;
```

```sql
-- Monthly PR creation trend (last 2 years)
SELECT 
  strftime('%Y-%m', created_at) as month,
  COUNT(*) as prs_created
FROM pull_requests 
WHERE created_at >= date('now', '-2 years')
GROUP BY month 
ORDER BY month;
```

---

## 🟡 **Level 2: Intermediate Queries**

### 2.1 Cross-Table Analysis

```sql
-- Contributors who both create issues AND submit PRs
SELECT 
  i.author_name,
  COUNT(DISTINCT i.issue_id) as issues_created,
  COUNT(DISTINCT pr.pr_id) as prs_created
FROM issues i
INNER JOIN pull_requests pr ON i.author_name = pr.author_name
GROUP BY i.author_name
HAVING issues_created >= 5 AND prs_created >= 5
ORDER BY (issues_created + prs_created) DESC
LIMIT 20;
```

```sql
-- Issues vs PR comments distribution
SELECT 
  'Issue Comments' as comment_type,
  COUNT(*) as count
FROM comments c
INNER JOIN issues i ON c.target_id = i.issue_id
UNION ALL
SELECT 
  'PR Comments',
  COUNT(*)
FROM comments c  
INNER JOIN pull_requests pr ON c.target_id = pr.pr_id;
```

### 2.2 Engagement Analysis

```sql
-- Most discussed issues (by comment count)
SELECT 
  i.issue_id,
  i.title,
  i.author_name as issue_creator,
  COUNT(c.comment_id) as comment_count,
  COUNT(DISTINCT c.author_name) as unique_commenters
FROM issues i
LEFT JOIN comments c ON i.issue_id = c.target_id
GROUP BY i.issue_id, i.title, i.author_name
HAVING comment_count > 0
ORDER BY comment_count DESC
LIMIT 15;
```

```sql
-- PR review participation (comments per PR)
SELECT 
  pr.pr_id,
  pr.title,
  pr.author_name as pr_author,
  COUNT(c.comment_id) as review_comments,
  COUNT(DISTINCT c.author_name) as reviewers
FROM pull_requests pr
LEFT JOIN comments c ON pr.pr_id = c.target_id
GROUP BY pr.pr_id, pr.title, pr.author_name
HAVING review_comments > 0
ORDER BY reviewers DESC, review_comments DESC
LIMIT 20;
```

---

## 🟠 **Level 3: Advanced Analytics**

### 3.1 Contributor Behavior Patterns

```sql
-- Multi-dimensional contributor activity matrix
SELECT 
  author_name,
  SUM(CASE WHEN activity_type = 'issue' THEN count ELSE 0 END) as issues,
  SUM(CASE WHEN activity_type = 'pr' THEN count ELSE 0 END) as prs,
  SUM(CASE WHEN activity_type = 'comment' THEN count ELSE 0 END) as comments,
  SUM(CASE WHEN activity_type = 'commit' THEN count ELSE 0 END) as commits,
  SUM(count) as total_activity
FROM (
  SELECT author_name, 'issue' as activity_type, COUNT(*) as count FROM issues GROUP BY author_name
  UNION ALL
  SELECT author_name, 'pr', COUNT(*) FROM pull_requests GROUP BY author_name  
  UNION ALL
  SELECT author_name, 'comment', COUNT(*) FROM comments GROUP BY author_name
  UNION ALL
  SELECT author_name, 'commit', COUNT(*) FROM commits GROUP BY author_name
) activity_union
WHERE author_name NOT LIKE 'bot_%'  -- Focus on humans
GROUP BY author_name
HAVING total_activity >= 100
ORDER BY total_activity DESC
LIMIT 25;
```

```sql
-- Temporal contribution patterns (activity by day of week)
SELECT 
  CASE CAST(strftime('%w', created_at) AS INTEGER)
    WHEN 0 THEN 'Sunday'
    WHEN 1 THEN 'Monday' 
    WHEN 2 THEN 'Tuesday'
    WHEN 3 THEN 'Wednesday'
    WHEN 4 THEN 'Thursday'
    WHEN 5 THEN 'Friday'
    WHEN 6 THEN 'Saturday'
  END as day_of_week,
  COUNT(*) as activity_count
FROM (
  SELECT created_at FROM issues WHERE created_at IS NOT NULL
  UNION ALL
  SELECT created_at FROM pull_requests WHERE created_at IS NOT NULL
  UNION ALL  
  SELECT created_at FROM comments WHERE created_at IS NOT NULL
) all_activities
GROUP BY strftime('%w', created_at)
ORDER BY CAST(strftime('%w', created_at) AS INTEGER);
```

### 3.2 Collaboration Network Analysis

```sql
-- Issue-Comment collaboration network (who responds to whom)
SELECT 
  i.author_name as issue_creator,
  c.author_name as commenter,
  COUNT(*) as interactions
FROM issues i
INNER JOIN comments c ON i.issue_id = c.target_id
WHERE i.author_name != c.author_name  -- Exclude self-comments
  AND i.author_name NOT LIKE 'bot_%' 
  AND c.author_name NOT LIKE 'bot_%'
GROUP BY i.author_name, c.author_name
HAVING interactions >= 10
ORDER BY interactions DESC
LIMIT 50;
```

```sql
-- Cross-PR collaboration (people who comment on same PRs)
WITH pr_participants AS (
  SELECT 
    pr.pr_id,
    pr.author_name as pr_author,
    c.author_name as commenter
  FROM pull_requests pr
  INNER JOIN comments c ON pr.pr_id = c.target_id
  WHERE pr.author_name != c.author_name
    AND pr.author_name NOT LIKE 'bot_%'
    AND c.author_name NOT LIKE 'bot_%'
)
SELECT 
  p1.commenter as person1,
  p2.commenter as person2, 
  COUNT(DISTINCT p1.pr_id) as shared_prs
FROM pr_participants p1
INNER JOIN pr_participants p2 ON p1.pr_id = p2.pr_id 
WHERE p1.commenter < p2.commenter  -- Avoid duplicates
GROUP BY p1.commenter, p2.commenter
HAVING shared_prs >= 15
ORDER BY shared_prs DESC
LIMIT 30;
```

---

## 🔴 **Level 4: Complex Research Queries**

### 4.1 Longitudinal Developer Journey Analysis

```sql
-- Developer evolution: First activity to current (activity timeline)
WITH first_activity AS (
  SELECT 
    author_name,
    MIN(activity_date) as first_seen,
    MAX(activity_date) as last_seen
  FROM (
    SELECT author_name, created_at as activity_date FROM issues WHERE created_at IS NOT NULL
    UNION ALL
    SELECT author_name, created_at FROM pull_requests WHERE created_at IS NOT NULL
    UNION ALL
    SELECT author_name, created_at FROM comments WHERE created_at IS NOT NULL
    UNION ALL
    SELECT author_name, author_date FROM commits WHERE author_date IS NOT NULL
  ) all_dates
  WHERE author_name NOT LIKE 'bot_%'
  GROUP BY author_name
),
activity_periods AS (
  SELECT 
    fa.*,
    ROUND(JULIANDAY(last_seen) - JULIANDAY(first_seen)) as days_active,
    CASE 
      WHEN JULIANDAY(last_seen) - JULIANDAY(first_seen) <= 30 THEN 'Short-term (<1 month)'
      WHEN JULIANDAY(last_seen) - JULIANDAY(first_seen) <= 180 THEN 'Medium-term (1-6 months)'
      WHEN JULIANDAY(last_seen) - JULIANDAY(first_seen) <= 365 THEN 'Long-term (6-12 months)'
      ELSE 'Very Long-term (>1 year)'
    END as engagement_duration
  FROM first_activity fa
)
SELECT 
  ap.engagement_duration,
  COUNT(*) as contributor_count,
  AVG(ap.days_active) as avg_days_active,
  MIN(ap.first_seen) as earliest_first_activity,
  MAX(ap.last_seen) as latest_last_activity
FROM activity_periods ap
GROUP BY ap.engagement_duration
ORDER BY 
  CASE ap.engagement_duration 
    WHEN 'Short-term (<1 month)' THEN 1
    WHEN 'Medium-term (1-6 months)' THEN 2  
    WHEN 'Long-term (6-12 months)' THEN 3
    ELSE 4
  END;
```

### 4.2 Issue Resolution and PR Merge Patterns

```sql
-- Issue resolution analysis with discussion intensity
WITH issue_metrics AS (
  SELECT 
    i.issue_id,
    i.author_name,
    i.created_at,
    i.closed_at,
    i.state,
    COUNT(c.comment_id) as comment_count,
    COUNT(DISTINCT c.author_name) as unique_participants,
    CASE 
      WHEN i.closed_at IS NOT NULL AND i.created_at IS NOT NULL 
      THEN ROUND(JULIANDAY(i.closed_at) - JULIANDAY(i.created_at))
      ELSE NULL 
    END as resolution_days
  FROM issues i
  LEFT JOIN comments c ON i.issue_id = c.target_id
  WHERE i.created_at IS NOT NULL
  GROUP BY i.issue_id, i.author_name, i.created_at, i.closed_at, i.state
)
SELECT 
  CASE 
    WHEN resolution_days IS NULL THEN 'Open/No resolution time'
    WHEN resolution_days <= 1 THEN 'Same day (0-1 days)'
    WHEN resolution_days <= 7 THEN 'Same week (2-7 days)'  
    WHEN resolution_days <= 30 THEN 'Same month (8-30 days)'
    WHEN resolution_days <= 90 THEN 'Same quarter (31-90 days)'
    ELSE 'Long-term (>90 days)'
  END as resolution_category,
  COUNT(*) as issue_count,
  AVG(comment_count) as avg_comments,
  AVG(unique_participants) as avg_participants,
  AVG(resolution_days) as avg_resolution_days
FROM issue_metrics
GROUP BY resolution_category
ORDER BY 
  CASE resolution_category
    WHEN 'Same day (0-1 days)' THEN 1
    WHEN 'Same week (2-7 days)' THEN 2
    WHEN 'Same month (8-30 days)' THEN 3  
    WHEN 'Same quarter (31-90 days)' THEN 4
    WHEN 'Long-term (>90 days)' THEN 5
    ELSE 6
  END;
```

### 4.3 Advanced Commit Analysis

```sql
-- Commit patterns and code change analysis
SELECT 
  c.author_name,
  COUNT(*) as total_commits,
  AVG(c.total_additions) as avg_additions_per_commit,
  AVG(c.total_deletions) as avg_deletions_per_commit,
  AVG(c.files_changed_count) as avg_files_per_commit,
  SUM(c.total_additions) as total_lines_added,
  SUM(c.total_deletions) as total_lines_deleted,
  SUM(c.total_additions + c.total_deletions) as total_lines_changed,
  COUNT(CASE WHEN c.is_merge_commit = 1 THEN 1 END) as merge_commits,
  ROUND(COUNT(CASE WHEN c.is_merge_commit = 1 THEN 1 END) * 100.0 / COUNT(*), 2) as merge_commit_percentage
FROM commits c
WHERE c.author_name NOT LIKE 'bot_%'
  AND c.total_additions IS NOT NULL 
  AND c.total_deletions IS NOT NULL
GROUP BY c.author_name
HAVING total_commits >= 20
ORDER BY total_lines_changed DESC
LIMIT 25;
```

### 4.4 Project Health Metrics

```sql
-- Project health: New contributor onboarding vs retention
WITH monthly_stats AS (
  SELECT 
    strftime('%Y-%m', first_activity) as month,
    COUNT(*) as new_contributors
  FROM (
    SELECT 
      author_name,
      MIN(activity_date) as first_activity
    FROM (
      SELECT author_name, created_at as activity_date FROM issues WHERE created_at IS NOT NULL
      UNION ALL
      SELECT author_name, created_at FROM pull_requests WHERE created_at IS NOT NULL  
      UNION ALL
      SELECT author_name, created_at FROM comments WHERE created_at IS NOT NULL
    ) all_activities
    WHERE author_name NOT LIKE 'bot_%'
    GROUP BY author_name
  ) first_activities
  GROUP BY strftime('%Y-%m', first_activity)
),
retention_stats AS (
  SELECT 
    strftime('%Y-%m', first_activity) as onboard_month,
    COUNT(*) as onboarded,
    SUM(CASE WHEN continued_after_month = 1 THEN 1 ELSE 0 END) as retained
  FROM (
    SELECT 
      author_name,
      MIN(activity_date) as first_activity,
      CASE 
        WHEN MAX(activity_date) > datetime(MIN(activity_date), '+30 days') THEN 1 
        ELSE 0 
      END as continued_after_month
    FROM (
      SELECT author_name, created_at as activity_date FROM issues WHERE created_at IS NOT NULL
      UNION ALL
      SELECT author_name, created_at FROM pull_requests WHERE created_at IS NOT NULL
      UNION ALL  
      SELECT author_name, created_at FROM comments WHERE created_at IS NOT NULL
    ) activities
    WHERE author_name NOT LIKE 'bot_%'
    GROUP BY author_name
  ) contributor_lifecycle  
  GROUP BY strftime('%Y-%m', first_activity)
)
SELECT 
  ms.month,
  ms.new_contributors,
  COALESCE(rs.retained, 0) as retained_contributors,
  ROUND(COALESCE(rs.retained, 0) * 100.0 / ms.new_contributors, 2) as retention_rate_percent
FROM monthly_stats ms
LEFT JOIN retention_stats rs ON ms.month = rs.onboard_month
WHERE ms.month >= '2022-01'  -- Focus on recent period
ORDER BY ms.month;
```

---

## 🔬 **Research Use Cases**

### Software Engineering Research Applications



1. **Open Source Community Health**
   ```sql
   -- Diversity of participation in discussions
   SELECT 
     CASE 
       WHEN comment_count = 1 THEN 'One-time commenter'
       WHEN comment_count <= 10 THEN 'Occasional commenter (2-10)'
       WHEN comment_count <= 50 THEN 'Regular commenter (11-50)'
       ELSE 'Very active commenter (50+)'
     END as participation_level,
     COUNT(*) as contributor_count
   FROM (
     SELECT author_name, COUNT(*) as comment_count
     FROM comments 
     WHERE author_name NOT LIKE 'bot_%'
     GROUP BY author_name
   ) contributor_activity
   GROUP BY participation_level;
   ```

2. **Bot Impact Analysis**
   ```sql
   -- Bot vs Human response patterns
   SELECT 
     'Bot' as responder_type,
     AVG(JULIANDAY(c.created_at) - JULIANDAY(i.created_at)) as avg_response_time_days
   FROM issues i
   INNER JOIN comments c ON i.issue_id = c.target_id
   WHERE c.author_name LIKE 'bot_%'
     AND c.created_at > i.created_at
   UNION ALL
   SELECT 
     'Human',
     AVG(JULIANDAY(c.created_at) - JULIANDAY(i.created_at))
   FROM issues i  
   INNER JOIN comments c ON i.issue_id = c.target_id
   WHERE c.author_name NOT LIKE 'bot_%'
     AND c.created_at > i.created_at;
   ```

---

## 📈 **Performance Tips**

### Indexing Recommendations
```sql
-- Create these indexes for better query performance
CREATE INDEX idx_comments_target_id ON comments(target_id);
CREATE INDEX idx_comments_author_name ON comments(author_name);  
CREATE INDEX idx_issues_author_name ON issues(author_name);
CREATE INDEX idx_pull_requests_author_name ON pull_requests(author_name);
CREATE INDEX idx_commits_author_name ON commits(author_name);
CREATE INDEX idx_created_at_issues ON issues(created_at);
CREATE INDEX idx_created_at_prs ON pull_requests(created_at);
CREATE INDEX idx_created_at_comments ON comments(created_at);
```

### Query Optimization
- Use `LIMIT` clauses for exploratory analysis
- Filter out bots early when focusing on human behavior
- Use `EXPLAIN QUERY PLAN` to understand query execution
- Consider creating temporary tables for complex multi-step analysis

---



