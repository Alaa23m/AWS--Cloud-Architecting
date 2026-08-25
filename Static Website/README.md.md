# Static Website Hosting on Amazon S3

Host a static website on S3 and apply data protection, lifecycle management, and cross-region replication to the underlying bucket.

## Services Used

- Amazon S3 (static website hosting, versioning, lifecycle rules, cross-region replication)
- IAM (CafeRole, used for replication permissions)

## What Was Built

- S3 bucket created in `us-east-1`, configured for static website hosting with `index.html` as the index document.
- Block Public Access disabled and ACLs enabled on the bucket to allow public object access.
- Bucket policy granting `s3:GetObject` (public read-only) to all principals, so newly uploaded objects (e.g. pastry images) are publicly readable without manual per-object ACL changes.
- Versioning enabled on the source bucket to protect website files from accidental overwrite or deletion.
- Lifecycle configuration with two separate rules on the source bucket:
  - Rule 1: transition noncurrent (previous) object versions to S3 Standard-IA after 30 days.
  - Rule 2: permanently delete noncurrent object versions after 365 days.
- Cross-region replication (CRR) from the source bucket to a destination bucket in a second AWS Region:
  - Versioning enabled on both source and destination buckets (required for CRR).
  - Replication rule uses `CafeRole`, an IAM role with the following permissions on all resources (`*`): `s3:ListBucket`, `s3:ReplicateObject`, `s3:ReplicateDelete`, `s3:ReplicateTags`, `s3:Get*`.
  - Existing objects not replicated retroactively; rule applies only to objects created after the rule was set up.

## Key Technical Decisions

- Used a bucket policy instead of per-object ACLs for public read access, since new objects need to be public on upload without manual configuration each time.
- Configured the lifecycle policy as two separate rules rather than one rule with two transitions, so each rule handles a single action (transition or expiration) independently.
- Declined to replicate existing objects when creating the CRR rule, limiting replication scope to newly created objects going forward.

## Problems Encountered

- Creating the replication rule returned the warning: "The replication rule is saved, but it might not work." This is expected when the rule is created before the bucket has objects matching the replication criteria. No action needed; replication began working once new objects were uploaded.

## Notes

- `CafeRole`'s policy grants replication permissions on all S3 resources (`Resource: '*'`). In a production setup this should be scoped to the source and destination bucket ARNs only.
