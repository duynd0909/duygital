---
title: "Dashboard Data Solution Documentation"
author: duygital
pubDatetime: 2025-05-06T01:37:31Z
slug: Dashboard Data Solution Documentation
featured: true
draft: false
tags:
  - Dashboard
description: This document describes the data solution for a dashboard that visualizes permission changes across systems. The dashboard displays metrics such as total permission changes, daily trends, system-specific statuses, and historical logs. The data originates in Elasticsearch and must be synced to PostgreSQL for efficient querying and aggregation to support the dashboard's requirements.
---

# Dashboard Data Solution Documentation

## Introduction

This document describes the data solution for a dashboard that visualizes permission changes across systems. The dashboard displays metrics such as total permission changes, daily trends, system-specific statuses, and historical logs. The data originates in Elasticsearch and must be synced to PostgreSQL for efficient querying and aggregation to support the dashboard's requirements.

The solution involves detecting changes in Elasticsearch, syncing them to PostgreSQL, and updating aggregated views for the dashboard. This document outlines the architecture, data flow, and implementation details.

## Solution Architecture

![Dashboard Data Solution Documentation](@/assets/images/image.png)

### Overview

The solution architecture consists of four main components:

- **Elasticsearch**: The source of permission change events, stored in the `permission_changes_index`.
- **Java Sync Process**: A Java program that detects changes in Elasticsearch and syncs them to PostgreSQL.
- **PostgreSQL**: The database storing raw data (`permission_changes`) and aggregated views (`daily_summary`).
- **Dashboard**: The final application that queries PostgreSQL to display metrics and logs.

### Data Flow

1. **Query Changes**: The Java sync process queries Elasticsearch for new or updated permission change events since the last sync, using a timestamp-based approach.
2. **Upsert/Delete**: Changes are synced to PostgreSQL by upserting new or updated records into the `permission_changes` table and deleting records as needed.
3. **Query Data**: The dashboard queries PostgreSQL to retrieve raw and aggregated data for display.

## Implementation Details

### Database Schema

The PostgreSQL database includes the following key tables and views:

- `permission_changes`: Stores raw permission change events. The schema for this table is as follows:

```sql
CREATE TABLE permission_changes (
    change_id SERIAL PRIMARY KEY,
    system_id INT REFERENCES systems(system_id),
    user_id INT REFERENCES users(user_id),
    type VARCHAR,
    level INT,
    change_timestamp TIMESTAMP NOT NULL,
    details TEXT, -- Additional details about the change
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

- `daily_permission_changes_view`: A view that aggregates permission changes by day, system, type, and level, for efficient querying by the dashboard. The view is defined as follows:

```sql
CREATE VIEW daily_permission_changes_view AS
SELECT
    DATE_TRUNC('day', change_timestamp) AS change_date,
    system_id,
    type,
    level,
    COUNT(*) AS change_count
FROM permission_changes
GROUP BY DATE_TRUNC('day', change_timestamp), system_id, type_id, level_id;
```

- `daily_summary`: A summary table for aggregated data (e.g., daily counts).
- `sync_metadata`: Tracks the last sync timestamp.

### Sync Process

The sync process is implemented in Java, using the Elasticsearch High-Level REST Client and PostgreSQL JDBC driver. Below are the key steps:

1. **Retrieve Last Sync Timestamp**: Query the `sync_metadata` table to get the last sync timestamp.
2. **Query Elasticsearch**: Use the timestamp to fetch new or updated documents from `permission_changes_index`.
3. **Sync Changes to PostgreSQL**: Upsert changes into the `permission_changes` table.
4. **Update Last Sync Timestamp**: Update the `sync_metadata` table with the current timestamp.
5. **Refresh Aggregations**: Refresh materialized views or update summary tables.

Below is a snippet of the Java code for syncing changes:

```java
// Process and sync changes to PostgreSQL
private static void syncChangesToPostgres(SearchResponse searchResponse, Connection pgConn) throws Exception {
    String upsertSql = """
        INSERT INTO permission_changes (change_id, system_id, user_id, type, level, change_timestamp, details, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?, ?)
        ON CONFLICT (change_id)
        DO UPDATE SET
            system_id = EXCLUDED.system_id,
            user_id = EXCLUDED.user_id,
            type = EXCLUDED.type,
            level = EXCLUDED.level,
            change_timestamp = EXCLUDED.change_timestamp,
            details = EXCLUDED.details,
            created_at = EXCLUDED.created_at
    """;
}
```

### Scheduling

The sync process is scheduled to run every 5 minutes using a cron job:

```bash
*/5 * * * *
```

### Performance Optimizations

- **Indexes**: Indexes on `change_timestamp`, `system_id`, `user_id`, `type_id`, and `level_id` in the `permission_changes` table ensure fast queries.
- **Materialized Views**: Aggregated data (e.g., daily counts) is precomputed in materialized views like `daily_permission_changes`. The `daily_permission_changes_view` provides a lightweight alternative for dynamic aggregation.
- **Batching**: The sync process fetches and processes changes in batches of 1000 documents to avoid overloading Elasticsearch or PostgreSQL.

## Conclusion

This solution enables efficient syncing of permission change data from Elasticsearch to PostgreSQL, ensuring the dashboard can display up-to-date metrics and logs. Future improvements could include real-time streaming with Kafka or additional optimizations for large-scale data.
