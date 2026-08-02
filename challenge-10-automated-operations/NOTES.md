# Challenge 10: Automated Operations - Scheduled Tasks

## What was done
- Created srijan-asset-sync CronJob (every 5 minutes) - simulates daily asset
  index updates
- Created srijan-analytics-report CronJob (every 10 minutes) - connects to
  Challenge 9's analytics service by curling srijan-analytics-svc directly
- Created srijan-db-cleanup CronJob (daily at 2 AM) - connects to Challenge 4's
  database, queries the asset_catalog table using credentials from Challenge 5's
  srijan-db-secret
- All three configured with successfulJobsHistoryLimit: 3, failedJobsHistoryLimit: 2
  (bounded job history, prevents unbounded Job object accumulation) and
  backoffLimit: 2 with restartPolicy: OnFailure (failure handling)

## Verification
- Triggered each CronJob manually via `kubectl create job --from=cronjob/...`
  to test immediately without waiting for the real schedule
- srijan-asset-sync: completed successfully, log confirmed timestamp output
- Confirmed the CronJob scheduler itself is working independently of manual
  triggers - observed real scheduled runs firing on their own
  (srijan-asset-sync-29761320, srijan-analytics-report-29761320), both Complete,
  proving the */5 and */10 minute schedules are genuinely active
- srijan-analytics-report and srijan-db-cleanup manual test jobs verified
  Complete with expected log output (curl response from analytics service;
  row count query result from MySQL respectively)
