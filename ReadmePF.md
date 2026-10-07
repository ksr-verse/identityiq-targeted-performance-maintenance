# Perform Targeted Maintenance Rule
This task runs for all the options where ids are provided.

Two files:

- `TargetedMaintenanceRule.xml` is the rule **Targeted Maintenance Rule**
- `TargetedMaintenanceRuleTask.xml` is the task **Perform Targeted Maintenance Rule**

## Import

Import the rule first, then the task, from the IdentityIQ console:

```
import "TargetedMaintenanceRule.xml"
import "TargetedMaintenanceRuleTask.xml"
```

## Open the task

Go to **Setup > Tasks**. Open **Perform Targeted Maintenance Rule**, provide relevant ids for the option you want to use and choose **Save and Execute**.

Each field name matches one option on the product task **Perform Maintenance**.

More than one field can be filled in the same run. Each id field takes one id, or several ids separated by commas.

```
7f0000018a1b1c2d018a1b1c2d010001
7f0000018a1b1c2d018a1b1c2d010001,7f0000018a1b1c2d018a1b1c2d010002
```

The three number fields are not id lists. Set them only when you want to change the thread count or the workflow thread timeout for this run.

This task does not read the checkboxes on the scheduled **Perform Maintenance** task. An option runs here when its field has ids.

| Field on this task | Perform Maintenance option |
|---|---|
| Identity snapshot ids (Prune identity snapshots) | Prune identity snapshots |
| Task result ids (Prune task results) | Prune task results |
| Request ids (Prune requests) | Prune requests |
| Provisioning transaction ids (Prune provisioning transactions) | Prune provisioning transactions |
| Certification ids to archive or prune (Archive and prune certifications) | Archive and prune certifications |
| Certification archive ids to prune (Archive and prune certifications) | Archive and prune certifications |
| Certification ids to close (Automatically close certifications) | Automatically close certifications |
| Certification ids to finish (Finish certifications) | Finish certifications |
| Number of finisher threads | Number of finisher threads |
| Certification ids to phase (Transition certifications phases) | Transition certifications phases |
| Certification ids to scan for revocations (Scan for completed revocations) | Scan for completed revocations |
| Inactive owner work item ids (Forward inactive user work items) | Forward inactive user work items |
| Batch request ids (Prune batch requests) | Prune batch requests |
| Syslog event ids (Prune syslog events) | Prune syslog events |
| Workflow event work item ids (Process background workflow events) | Process background workflow events |
| Number of background workflow threads | Number of background workflow threads |
| Workflow thread timeout (seconds) | Workflow thread timeout (seconds) |
| Attachment ids (Prune Attachments) | Prune Attachments |
| Pending attachment ids (Prune Pending Attachments) | Prune Pending Attachments |

**Denormalize scopes** and **Enable Partitioning** are not on this task because they do not take an id list.

After you execute the task, the result attribute **Targeted partitions launched** is the number of requests that were queued. Stdout lines from the rule start with `[Custom Performance Maintenance]`.

A missing id is reported on the task result. That field is not queued. The other fields in the same run are still queued.

## Find the ids

```sql
USE identityiq;

SET @now_ms = UNIX_TIMESTAMP(NOW()) * 1000;

-- Each variable is one SystemConfiguration key. The names are not the same.

-- Custom Variable for query                          -- Variable name in SystemConfiguration
SET @identity_snapshot_max_age_days = 0;              -- identitySnapshotMaxAge
SET @task_result_max_age_days = 0;                    -- taskResultMaxAge
SET @request_max_age_days = 0;                        -- requestMaxAge
SET @provisioning_transaction_log_prune_age_days = 0; -- provisioningTransactionLogPruneAge
SET @certification_max_age_days = 0;                  -- certificationMaxAge
SET @certification_archive_max_age_days = 0;          -- certificationArchiveMaxAge
SET @syslog_purge_age_days = 0;                       -- syslogPurgeAge
SET @pending_attachment_prune_age_hours = 12;         -- pendingAttachmentPruneAge

SET @snapshot_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @identity_snapshot_max_age_days DAY)) * 1000;
SET @task_result_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @task_result_max_age_days DAY)) * 1000;
SET @request_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @request_max_age_days DAY)) * 1000;
SET @provisioning_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @provisioning_transaction_log_prune_age_days DAY)) * 1000;
SET @certification_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @certification_max_age_days DAY)) * 1000;
SET @archive_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @certification_archive_max_age_days DAY)) * 1000;
SET @syslog_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @syslog_purge_age_days DAY)) * 1000;
SET @pending_threshold_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL @pending_attachment_prune_age_hours HOUR)) * 1000;
SET @thirty_days_ms = UNIX_TIMESTAMP(DATE_SUB(NOW(), INTERVAL 30 DAY)) * 1000;
```

### Identity snapshot ids (Prune identity snapshots)

Skip when `@identity_snapshot_max_age_days` is `0` or less.

```sql
SELECT id
FROM spt_identity_snapshot
WHERE created <= @snapshot_threshold_ms;
```

### Task result ids (Prune task results)

Run the first query only when `@task_result_max_age_days` is greater than `0`. Run the second query every time. Paste ids from either result into the same field.

```sql
SELECT id
FROM spt_task_result
WHERE completed IS NOT NULL
  AND pending_signoffs = 0
  AND (expiration IS NULL OR expiration = 0)
  AND created <= @task_result_threshold_ms;
```

```sql
SELECT id
FROM spt_task_result
WHERE completed IS NOT NULL
  AND expiration > 0
  AND expiration <= @now_ms;
```

### Request ids (Prune requests)

Run the first query only when `@request_max_age_days` is greater than `0`. Run the second query every time. Paste ids from either result into the same field.

```sql
SELECT id
FROM spt_request
WHERE completed IS NOT NULL
  AND expiration IS NULL
  AND created <= @request_threshold_ms;
```

```sql
SELECT id
FROM spt_request
WHERE completed IS NOT NULL
  AND expiration IS NOT NULL
  AND expiration <= @now_ms;
```

### Provisioning transaction ids (Prune provisioning transactions)

Skip when `@provisioning_transaction_log_prune_age_days` is `0` or less.

```sql
SELECT id
FROM spt_provisioning_transaction
WHERE created <= @provisioning_threshold_ms
  AND status <> 'Pending';
```

### Certification ids to archive or prune (Archive and prune certifications)

Skip when `@certification_max_age_days` is `0` or less. Add `AND immutable = 0` only when `@certification_archive_max_age_days` is negative. The product can still skip an id after this query when the certification cannot be archived.

```sql
SELECT id
FROM spt_certification
WHERE signed <= @certification_threshold_ms;
```

### Certification archive ids to prune (Archive and prune certifications)

Skip when `@certification_archive_max_age_days` is `0` or less. Paste these ids into the archive field, not the certification field.

```sql
SELECT id
FROM spt_certification_archive
WHERE created <= @archive_threshold_ms
  AND immutable = 0;
```

### Certification ids to close (Automatically close certifications)

```sql
SELECT id
FROM spt_certification
WHERE automatic_closing_date < @now_ms
  AND signed IS NULL;
```

### Certification ids to finish (Finish certifications)

```sql
SELECT id
FROM spt_certification
WHERE finished IS NULL
  AND signed IS NOT NULL
  AND phase <> 'Staged'
ORDER BY signed;
```

### Certification ids to phase (Transition certifications phases)

Paste the certification ids. The rule loads the due certification items for those ids. Do not paste certification item ids into this field.

```sql
SELECT id
FROM spt_certification
WHERE next_phase_transition < @now_ms;
```

### Certification ids to scan for revocations (Scan for completed revocations)

```sql
SELECT id
FROM spt_certification
WHERE next_remediation_scan IS NOT NULL
  AND next_remediation_scan < @now_ms;
```

### Inactive owner work item ids (Forward inactive user work items)

```sql
SELECT w.id
FROM spt_work_item w
JOIN spt_identity i ON i.id = w.owner
WHERE i.inactive = 1;
```

### Batch request ids (Prune batch requests)

The 30 days are fixed in the product.

```sql
SELECT id
FROM spt_batch_request
WHERE created <= @thirty_days_ms;
```

### Syslog event ids (Prune syslog events)

Skip when `@syslog_purge_age_days` is `0` or less.

```sql
SELECT id
FROM spt_syslog_event
WHERE created <= @syslog_threshold_ms;
```

### Workflow event work item ids (Process background workflow events)

```sql
SELECT id
FROM spt_work_item
WHERE type = 'Event'
  AND iiqlock IS NULL
  AND (expiration IS NULL OR expiration <= @now_ms);
```

### Attachment ids (Prune Attachments)

The 30 days are fixed in the product. Attachments that are still linked from an identity request item are excluded.

```sql
SELECT id
FROM spt_attachment
WHERE created <= @thirty_days_ms
  AND id NOT IN (
    SELECT DISTINCT attachment_id
    FROM spt_identity_req_item_attach
    WHERE attachment_id IS NOT NULL
  );
```

### Pending attachment ids (Prune Pending Attachments)

```sql
SELECT id
FROM spt_pending_req_attach
WHERE created <= @pending_threshold_ms;
```
