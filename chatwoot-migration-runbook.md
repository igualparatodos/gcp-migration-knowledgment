# Chatwoot Migration Runbook: AWS → GCP

**Purpose**: Migrate Chatwoot from AWS EC2/RDS/S3 to GCP Compute Engine/CloudSQL/GCS

**Duration**: ~4-6 hours (database: 30-60min, storage: 3-5hrs depending on network)

**Prerequisites**: 
- GCP infrastructure already provisioned via Terragrunt (VPC, CloudSQL, Compute Engine, GCS bucket)
- AWS Chatwoot instance **MUST be stopped** before migration to ensure data consistency

---

## Infrastructure Overview

### AWS (Source)
- **EC2**: `chatwoot.igual.com` (sa-east-1)
- **RDS**: `chatwoot-prod.cluster-clqie6c28evp.sa-east-1.rds.amazonaws.com`
  - Database: `chatwoot_production`
  - User: `postgres`
- **S3**: `s3://chatwoot-s3-prod` (sa-east-1)
  - ~710,234 files, 196.7 GiB

### GCP (Destination)
- **Compute Engine**: `chatwoot` (southamerica-east1-a)
  - External IP: `34.39.149.165`
  - Machine: n2-standard-8
- **CloudSQL**: `chatwoot-db` (southamerica-east1)
  - Private IP: `10.103.0.3`
  - Database: `chatwoot`
  - User: `chatwoot`
- **GCS**: `gs://igualparatodos-chatwoot` (southamerica-east1)

---

## Migration Steps

### 1. Deploy the latest to GCP

```bash
cd cw
cd chatwoot-igual
git pull
docker build -t chatwoot:igual-4.1.0 -f docker/Dockerfile .
cd ..
docker-compose down
docker-compose up -d
```

### 2. Authenticate to GCP (DRY-RUN)

```bash
# Authenticate with your Google account
gcloud auth login

# Set default project
gcloud config set project igual-production

# Verify authentication
gcloud auth list
gcloud config list
```

### 3. Stop AWS Chatwoot Instance

#### 3.1 Stop lambda
https://sa-east-1.console.aws.amazon.com/lambda/home?region=sa-east-1#/functions/n8n-waha-messages-prod?subtab=triggers&tab=configure

**CRITICAL**: Stop the AWS instance to prevent data changes during migration.

```bash
# SSH to AWS EC2
ssh ec2-user@chatwoot.igual.com

# Stop Chatwoot services
cd /home/ec2-user/cw
docker compose down

# Verify all containers stopped
docker ps

# Exit AWS instance
exit
```

### 4. Connect to GCP Compute Instance (DRY-RUN)

```bash
# SSH via IAP tunnel (no public SSH key needed)
gcloud compute ssh chatwoot \
  --project=igual-production \
  --zone=southamerica-east1-a \
  --tunnel-through-iap
```

### 5. Migrate Database (RDS → CloudSQL) (DRY-RUN)

**On GCE VM:**

```bash
# Run database migration script
sudo /opt/chatwoot/migrate-db-rds-to-cloudsql.sh
```

- You can move to step 6 and run in parallel while dump is running.

**What this does:**
1. Dumps PostgreSQL from AWS RDS (`chatwoot_production` database)
2. Restores to GCP CloudSQL (`chatwoot` database)
3. Verifies migration with row counts

**Expected output:**
- Dump file: ~367 MB
- Tables: 78
- Messages: ~3.2M
- Conversations: ~115K
- Duration: 30-60 minutes

**Troubleshooting:**
- If you see `pg_restore: warning: errors ignored on restore: XXX` - these are typically duplicate index warnings and are **safe to ignore**
- If pg_dump/pg_restore commands not found: PostgreSQL 15 client tools should already be installed

### 6. Migrate Storage (S3 → GCS) — Async (DRY-RUN)

**On GCE VM:**

```bash
# Run storage sync script
sudo /opt/chatwoot/sync-s3-to-gcs.sh
```

**What this does:**
1. Uses `gsutil rsync` to copy all files from S3 to GCS
2. Parallel multi-threaded transfer for speed
3. Checksum-based comparison (only copies changed files)
4. **Does NOT delete source data**

**Expected output:**
- Files: ~710,234
- Size: ~196.7 GiB
- Duration: 3-5 hours (depends on network speed)

**Monitor progress:**
```bash
# In another terminal, check GCS bucket size
gsutil du -sh gs://igualparatodos-chatwoot

# Check sync process
ps aux | grep gsutil
```

**Re-run sync anytime:**
The script is idempotent - run it multiple times for incremental updates without re-copying existing files.

> Note (Async): This step is long-running (3–5 hours). You can start it and move forward with the next steps (service verification, DNS planning, etc.). If desired, run it in the background:

```bash
# Start sync detached and log to file
nohup sudo /opt/chatwoot/sync-s3-to-gcs.sh > /opt/chatwoot/s3-to-gcs-sync.log 2>&1 &
tail -f /opt/chatwoot/s3-to-gcs-sync.log
```

### 7. Restart and verify Services (DRY-RUN)

**On GCE VM:**

```bash
# Restart service
sudo docker compose -f /opt/chatwoot/docker-compose.yaml restart

# Check container status
sudo docker compose -f /opt/chatwoot/docker-compose.yaml ps

# Expected output - all containers "Up":
# chatwoot-rails-1     Up
# chatwoot-sidekiq-1   Up
# chatwoot-redis-1     Up

# Check application logs
sudo docker compose -f /opt/chatwoot/docker-compose.yaml logs --tail=50 rails | grep -i error

# Test local endpoint
curl -I http://127.0.0.1:3000/api/v1/profile
# Expected: HTTP/1.1 401 Unauthorized (auth required, but app is responding)
```

### 8. Test Application via Nginx (DRY-RUN)

**On GCE VM:**

```bash
# Check Nginx configuration
sudo nginx -t

# Test public endpoint
curl -I http://34.39.149.165
# Expected: HTTP/1.1 401 Unauthorized

# Check Nginx logs
sudo tail -50 /var/log/nginx/access.log
```

### 9. DNS Cutover (When Ready)

**Update DNS record at Cloudflare:**
- Domain: `chatwoot.igual.com`
- Type: A
- Value: `34.39.149.165` (GCE external IP)
- TTL: 300 (5 minutes for quick rollback if needed)

**Wait for DNS propagation:**
```bash
# Check DNS resolution
dig chatwoot.igual.com +short

# Test from different location
curl -I https://chatwoot.igual.com
```

### 10. SSL Certificate (After DNS Points to GCP)

**On GCE VM:**

```bash
# Update .env to use production domain
sudo sed -i 's|chatwoot-gcp.igual.com|chatwoot.igual.com|g' /opt/chatwoot/.env

# Restart application
cd /opt/chatwoot
sudo docker compose restart rails sidekiq

# Install SSL certificate
sudo certbot --nginx \
  -d chatwoot.igual.com \
  --non-interactive \
  --agree-tos \
  -m guilherme.barczyszyn@gmail.com \
  --redirect

# Verify SSL
curl -I https://chatwoot.igual.com
```

### 11. Enable Lambda
https://sa-east-1.console.aws.amazon.com/lambda/home?region=sa-east-1#/functions/n8n-waha-messages-prod?subtab=triggers&tab=configure

### 13. Post-Migration Validation

**Functional Tests:**

```bash
# 1. Login to Chatwoot web interface
open https://chatwoot.igual.com

# 2. Verify key functionality:
# - Login works
# - Can view conversations
# - Can send/receive messages
# - File uploads work (tests GCS)
# - Sidekiq jobs processing (check Background Jobs in Settings)

# 3. Check database connectivity
# On GCE VM:
sudo docker compose exec rails bundle exec rails c
# In Rails console:
# > Account.count
# > Conversation.count
# > Message.count
# Expected: Should show migrated data counts
```

---

## Rollback Procedure

If issues occur, rollback to AWS:

1. **Update DNS** to point back to AWS EC2 IP
2. **Start AWS instance:**
   ```bash
   ssh ec2-user@chatwoot.igual.com
   cd /home/ec2-user/cw
   docker compose up -d
   ```
3. **Verify AWS instance** is working
4. **Investigate GCP issues** without time pressure

---

## Migration Scripts Reference

Both scripts are located on GCE VM at `/opt/chatwoot/`:

### Database Migration Script
```bash
sudo /opt/chatwoot/migrate-db-rds-to-cloudsql.sh
```
- **Source**: AWS RDS PostgreSQL
- **Destination**: GCP CloudSQL PostgreSQL
- **Safety**: Uses `--clean --if-exists` (safe to re-run)

### Storage Migration Script
```bash
sudo /opt/chatwoot/sync-s3-to-gcs.sh
```
- **Source**: S3 bucket `chatwoot-s3-prod`
- **Destination**: GCS bucket `igualparatodos-chatwoot`
- **Safety**: Sync only, never deletes source data
- **Idempotent**: Safe to run multiple times

---

## Key Credentials

**RDS (Source)**
- Host: `chatwoot-prod.cluster-clqie6c28evp.sa-east-1.rds.amazonaws.com`
- User: `postgres`
- Password: In `/opt/chatwoot/migrate-db-rds-to-cloudsql.sh`

**CloudSQL (Destination)**
- Host: `10.103.0.3` (private IP)
- User: `chatwoot`
- Password: In `/opt/chatwoot/migrate-db-rds-to-cloudsql.sh`

**S3 (Source)**
- Bucket: `chatwoot-s3-prod`
- Credentials: In `/opt/chatwoot/sync-s3-to-gcs.sh`

**GCS (Destination)**
- Bucket: `igualparatodos-chatwoot`
- Auth: Application Default Credentials (VM service account)

---

## Timeline Estimate

| Phase | Duration | Can Run In Parallel |
|-------|----------|---------------------|
| Stop AWS Instance | 2 min | - |
| Database Migration | 30-60 min | No |
| Storage Migration | 3-5 hrs | Yes (can start while DB is running) |
| Deploy Code | 3-4 min | After DB done |
| Verify & Test | 15-30 min | - |
| DNS Cutover (Cloudflare) | 5-15 min (propagation) | - |
| SSL Setup | 5 min | After DNS |
| **Total** | **4-6 hours** | - |

**Recommended execution time**: Off-peak hours or scheduled maintenance window

---

## Support & Troubleshooting

**Common Issues:**

1. **pg_restore warnings about duplicate indexes**
   - **Solution**: Safe to ignore - these occur when restoring into an existing schema

2. **Storage sync seems stuck**
   - **Solution**: It's building file list (710K files takes time). Wait for "At source listing..." messages

3. **GCS permission denied**
   - **Solution**: Verify VM service account has `roles/storage.objectAdmin` on bucket

4. **Application won't start**
   - **Solution**: Check `.env` file, verify POSTGRES_HOST=10.103.0.3 and GCS_BUCKET set correctly

5. **502 Bad Gateway from Nginx**
   - **Solution**: Rails container may still be starting. Wait 30s and retry

**Get Help:**
- Check logs: `sudo docker compose logs rails sidekiq`
- Verify database: `psql -h 10.103.0.3 -U chatwoot -d chatwoot`
- Test GCS: `gsutil ls gs://igualparatodos-chatwoot | head`

---

## Post-Migration Cleanup (Optional)

After confirming GCP migration is stable for 1-2 weeks:

1. **Archive AWS resources** (do not delete immediately):
   - Take final RDS snapshot
   - Export S3 bucket inventory
   - Stop EC2 instance (keep for 30 days)

2. **Monitor costs** on both platforms

3. **Document any issues** encountered for future migrations

---

**Last Updated**: 2026-03-12  
**Maintained By**: SRE Team  
**Related Documentation**: 
- [Chatwoot GCS Configuration](https://developers.chatwoot.com/self-hosted/deployment/storage/gcs-bucket)
- [GCP Compute Instance Terragrunt Config](../iac-gcp/live/production/southamerica-east1/compute/chatwoot/)
- [GitHub Actions Deploy Workflow](../chatwoot/.github/workflows/deploy-gce.yml)
