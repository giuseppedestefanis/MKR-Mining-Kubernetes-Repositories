# MKR - Mining Kubernetes Repositories: The Cloud was Not Built in a Day

Giuseppe Destefanis, Silvia Bartolucci, Daniel Feitosa

MSR - Mining Challenge

# Sample Dataset for Reviewers
## Anonymized Kubernetes Development Dataset (Sample)

---

## 📋 **Overview**

This sample dataset contains **41,000 records** across 4 CSV files, representing a curated subset of the complete MKR dataset. The sample maintains all key characteristics, relationships, and anonymization properties of the full dataset while staying within practical file size limits for review purposes.

### **Purpose**
- Enable reviewers to evaluate dataset quality and structure
- Demonstrate anonymization methodology 
- Show preserved relationships between different data types
- Provide realistic examples for potential research applications

---

## 📁 **Files Included**

| File | Records | Size | Description |
|------|---------|------|-------------|
| `sample_issues.csv` | 8,000 | 24.8 MB | GitHub issues with diverse authors and states |
| `sample_pull_requests.csv` | 6,000 | 15.5 MB | Pull requests including merged and closed PRs |
| `sample_comments.csv` | 12,000 | 6.9 MB | Comments on issues and PRs with maintained relationships |
| `sample_commits.csv` | 15,000 | 12.3 MB | Git commits with author and committer information |

**Total Sample Size:** 59.6 MB (all files under 50 MB limit)

---

## 🎯 **Sampling Strategy**

### **Representative Sampling**
- **Recent Activity**: Includes recent issues, PRs, and comments
- **Diverse Authors**: Maximum 2 items per author to ensure diversity  
- **Cross-Table Links**: Comments are linked to sampled issues and PRs
- **Temporal Span**: Covers 2014-2025 (11 years of development)
- **State Diversity**: Mix of open/closed issues, merged/unmerged PRs

### **Relationship Preservation**
- **100% Comment Links**: All 12,000 comments link to sampled issues or PRs
  - 4,287 comments → issues (35.7%)
  - 7,713 comments → PRs (64.3%)
- **Cross-Table Authors**: 1,134 authors (19.5%) appear in multiple tables
- **Author Consistency**: Same `author_name` = same person across all tables

---

## 🔒 **Anonymization Details**

### **Complete Privacy Protection**
- ✅ **Zero PII**: No personally identifiable information remains
- ✅ **Consistent Hashing**: All `author_id` fields use SHA256 + salt  
- ✅ **Anonymized Names**: `contributor_XXXXX` for humans, `bot_XXXXX` for bots
- ✅ **Content Preserved**: Issue/PR/comment content maintained for research

### **Author Anonymization**
- **5,794 Human Contributors**: Anonymized as `contributor_00001` through `contributor_XXXXX`
- **14 Bot Accounts**: Anonymized as `bot_00001` through `bot_XXXXX`
- **Identity Linking**: Same contributor ID across all tables = same person

### **Data Integrity**
- **Foreign Keys**: Comments properly link to issues/PRs via `target_id`
- **Timestamps**: All dates preserved for temporal analysis
- **Metadata**: Labels, states, commit statistics maintained

---

## 📊 **Dataset Characteristics**

### **Activity Distribution**

| Activity Type | Human | Bot | Total |
|---------------|-------|-----|-------|
| **Issues** | 7,995 (99.9%) | 5 (0.1%) | 8,000 |
| **Pull Requests** | 6,000 (100%) | 0 (0%) | 6,000 |
| **Comments** | 8,555 (71.3%) | 3,445 (28.7%) | 12,000 |
| **Commits** | 11,499 (76.7%) | 3,501 (23.3%) | 15,000 |

### **Cross-Table Participation**
- **Single Table**: 4,674 authors (80.5%) - specialized contributors
- **Two Tables**: 754 authors (13.0%) - cross-functional activity  
- **Three Tables**: 300 authors (5.2%) - highly engaged
- **All Four Tables**: 80 authors (1.4%) - complete ecosystem participation

### **Temporal Coverage**
- **Issues**: July 2014 → March 2025 (10+ years)
- **Pull Requests**: August 2014 → March 2025 (10+ years)  
- **Comments**: December 2014 → March 2025 (10+ years)
- **Commits**: June 2014 → July 2025 (11+ years)

---

## 🗄️ **Database Schema**

### **Table Relationships**
```
ISSUES (1) ←→ (M) COMMENTS (via target_id)
PULL_REQUESTS (1) ←→ (M) COMMENTS (via target_id)

Same author_name across tables = Same person
```

### **Key Fields**

#### **Common Fields (All Tables)**
- `author_name` - Anonymized contributor identifier
- `author_id` - Hashed original author ID
- `created_at` - Creation timestamp

#### **Issues (`sample_issues.csv`)**
- `issue_id` (PK), `issue_number`, `title`, `body`, `state`, `closed_at`

#### **Pull Requests (`sample_pull_requests.csv`)**  
- `pr_id` (PK), `pr_number`, `title`, `body`, `state`, `merged_at`

#### **Comments (`sample_comments.csv`)**
- `comment_id` (PK), `target_id` (FK), `body`, `user_type`

#### **Commits (`sample_commits.csv`)**
- `commit_id` (PK), `message`, `committer_name`, `total_additions`, `total_deletions`

---

## 🔍 **Sample Analysis Queries**

### **Cross-Table Author Analysis**
```sql
-- Find authors active across multiple areas
SELECT author_name,
       COUNT(DISTINCT 'issue') as creates_issues,
       COUNT(DISTINCT 'pr') as creates_prs,
       COUNT(DISTINCT 'comment') as writes_comments,
       COUNT(DISTINCT 'commit') as makes_commits
FROM (
    SELECT author_name, 'issue' as activity FROM sample_issues
    UNION ALL
    SELECT author_name, 'pr' FROM sample_pull_requests  
    UNION ALL
    SELECT author_name, 'comment' FROM sample_comments
    UNION ALL
    SELECT author_name, 'commit' FROM sample_commits
)
GROUP BY author_name
HAVING creates_issues + creates_prs + writes_comments + makes_commits > 1
ORDER BY (creates_issues + creates_prs + writes_comments + makes_commits) DESC
LIMIT 20;
```

### **Bot vs Human Activity**
```sql  
-- Compare bot vs human contribution patterns
SELECT 
    CASE WHEN author_name LIKE 'bot_%' THEN 'Bot' ELSE 'Human' END as contributor_type,
    COUNT(*) as total_comments,
    COUNT(DISTINCT author_name) as unique_contributors
FROM sample_comments
GROUP BY contributor_type;
```

### **Issue Discussion Intensity**
```sql
-- Find most discussed issues
SELECT 
    i.issue_id,
    i.title,
    i.author_name as creator,
    COUNT(c.comment_id) as comment_count,
    COUNT(DISTINCT c.author_name) as unique_commenters
FROM sample_issues i
LEFT JOIN sample_comments c ON i.issue_id = c.target_id
GROUP BY i.issue_id, i.title, i.author_name
ORDER BY comment_count DESC
LIMIT 10;
```


---

## ⚠️ **Important Notes**

### **Ethical Use**
- This data is provided for **research purposes only**
- **No re-identification attempts** should be made
- Original contributor privacy must be **strictly protected**

### **Limitations**
- **Sample Size**: 2.8% of full dataset (41K of 1.4M+ records)
- **Content Truncation**: Comment/issue bodies limited to 2,000 characters
- **Selection Bias**: Sampling may not represent all edge cases

### **Data Quality**
- **100% Relationship Integrity**: All foreign keys valid
- **100% Anonymization**: All PII removed or anonymized  
- **Complete Temporal Coverage**: 11-year span preserved

---

