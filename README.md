## Important Note About Certificate Management in TrueNAS

### The midclt Limitation

While using `midclt` to update certificates appears to work based on success messages, there is a known issue in TrueNAS SCALE where the certificate database may not properly update metadata (like issuance dates) even when the certificate content itself is updated. This leads to a discrepancy where:

1. The actual certificate files used by the web UI and services are updated
2. The certificate dates shown in the TrueNAS UI remain unchanged 
3. The browser correctly shows the new certificate dates

### Direct File Update: A More Reliable Approach

After extensive testing, a more reliable method is to directly update the certificate files in the system and restart the web service. This approach bypasses the TrueNAS certificate database limitations.

The improved script (below) features:
- Direct file updates to `/etc/certificates/`
- Certificate age checking to force renewal after a specified number of days
- Proper certificate comparison to avoid unnecessary updates
- Log rotation for better troubleshooting

While this method won't update the dates shown in the TrueNAS UI, it ensures that the actual certificates used by the system are properly updated and recognized by browsers.

### Optimized Certificate Update Script

The script below replaces the original script with the improved direct file approach:

```bash
#!/usr/bin/env bash
set -euo pipefail

########################################
# CONFIGURATION / PATHS
########################################

# Script directory for logging
SCRIPT_DIR="$(dirname "$(realpath "$0")")"
LOG_FILE="${SCRIPT_DIR}/acme-update.log"
LOG_MAX_SIZE=1048576  # 1MB in bytes
LOG_KEEP_FILES=3

# Days threshold for renewal
RENEWAL_DAYS_THRESHOLD=50

GD_ENV="./.godaddy.env"
CERT_DOMAIN="yourdomain.com"  # Change to your domain

ACME_BASE="/root/.acme.sh/${CERT_DOMAIN}_ecc"
CERT_PATH="${ACME_BASE}/fullchain.cer"
KEY_PATH="${ACME_BASE}/${CERT_DOMAIN}.key"

PERSIST_BASE="/mnt/tank/acme/certs"  # Change to your pool/dataset
PERSIST_CERT="${PERSIST_BASE}/${CERT_DOMAIN}-fullchain.cer"
PERSIST_KEY="${PERSIST_BASE}/${CERT_DOMAIN}.key"

# System certificate locations
SYS_CERT_PATH="/etc/certificates/letsencrypt.crt"
SYS_KEY_PATH="/etc/certificates/letsencrypt.key"

########################################
# LOGGING SETUP
########################################
setup_logging() {
    # Create log directory if it doesn't exist
    mkdir -p "$(dirname "$LOG_FILE")"
    
    # Check if log file exceeds max size
    if [ -f "$LOG_FILE" ]; then
        LOG_SIZE=$(stat -c %s "$LOG_FILE" 2>/dev/null || stat -f %z "$LOG_FILE" 2>/dev/null)
        if [ "$LOG_SIZE" -gt "$LOG_MAX_SIZE" ]; then
            # Rotate logs
            for i in $(seq $((LOG_KEEP_FILES-1)) -1 1); do
                if [ -f "${LOG_FILE}.$i" ]; then
                    mv "${LOG_FILE}.$i" "${LOG_FILE}.$((i+1))"
                fi
            done
            mv "$LOG_FILE" "${LOG_FILE}.1"
            touch "$LOG_FILE"
        fi
    else
        touch "$LOG_FILE"
    fi
    
    # Redirect stdout and stderr to both console and log file
    exec > >(tee -a "$LOG_FILE") 2>&1
    
    echo "====== Certificate Update Script Started $(date) ======"
}

########################################
# CLI FLAGS: --force, --cp
########################################
FORCE=false
COPY_ONLY=false

for arg in "$@"; do
  case "$arg" in
    --force|-f)
      FORCE=true
      shift
      ;;
    --cp|-c)
      COPY_ONLY=true
      shift
      ;;
    *)
      echo "Unknown argument: $arg" >&2
      echo "Usage: $0 [--force] [--cp]" >&2
      exit 1
      ;;
  esac
done

########################################
# SETUP LOGGING
########################################
setup_logging

########################################
# FUNCTION: CHECK CERTIFICATE AGE
########################################
check_cert_age() {
    if [ ! -f "$CERT_PATH" ]; then
        echo "Certificate file not found. Will force renewal."
        FORCE=true
        return 0
    fi
    
    # Get certificate dates
    local CERT_START_DATE=$(openssl x509 -in "$CERT_PATH" -noout -startdate 2>/dev/null | cut -d= -f2-)
    local CERT_END_DATE=$(openssl x509 -in "$CERT_PATH" -noout -enddate 2>/dev/null | cut -d= -f2-)
    
    echo "Certificate start date: $CERT_START_DATE"
    echo "Certificate end date: $CERT_END_DATE"
    
    # Get certificate file modification time
    local CERT_MOD_TIME=$(stat -c %Y "$CERT_PATH" 2>/dev/null || stat -f %m "$CERT_PATH" 2>/dev/null)
    local CURRENT_TIME=$(date +%s)
    
    # Calculate age in days
    local CERT_AGE_SECONDS=$((CURRENT_TIME - CERT_MOD_TIME))
    local CERT_AGE_DAYS=$((CERT_AGE_SECONDS / 86400))
    
    echo "Certificate file age: $CERT_AGE_DAYS days"
    
    if [ $CERT_AGE_DAYS -lt $RENEWAL_DAYS_THRESHOLD ]; then
        echo "Certificate is only $CERT_AGE_DAYS days old (threshold: $RENEWAL_DAYS_THRESHOLD days)."
        if [ "$FORCE" = "true" ]; then
            echo "Force flag is set, will proceed with forced renewal anyway."
            return 0
        fi
        echo "Skipping renewal process. Use --force to override."
        return 1
    fi
    
    echo "Certificate is $CERT_AGE_DAYS days old, which exceeds the threshold of $RENEWAL_DAYS_THRESHOLD days."
    echo "Will force renewal automatically."
    # Set the FORCE flag to true when certificate is older than threshold
    FORCE=true
    return 0
}

########################################
# LOAD DNS ENV VARIABLES
########################################
if [ "$COPY_ONLY" = false ]; then
  if [ -f "$GD_ENV" ]; then
    . "$GD_ENV"
  else
    echo "ERROR: DNS provider env file not found at $GD_ENV"
    exit 1
  fi

  if [ -z "${GD_Key:-}" ] || [ -z "${GD_Secret:-}" ]; then
    echo "ERROR: GD_Key or GD_Secret not set. Check $GD_ENV."
    exit 1
  fi
fi

########################################
# FUNCTION: COPY TO TRUENAS
########################################
copy_to_truenas() {
  # Ensure persistence directory exists and copy files there.
  mkdir -p "$PERSIST_BASE" || { echo "ERROR: mkdir failed"; return 1; }
  
  echo "Copying certificate from ACME to persistence location..."
  cp -f "$CERT_PATH" "$PERSIST_CERT" || { echo "ERROR: cp cert failed"; return 1; }
  
  # Brief pause between file operations
  sleep 1
  
  cp -f "$KEY_PATH" "$PERSIST_KEY" || { echo "ERROR: cp key failed"; return 1; }

  # Verify copy
  if [ ! -f "$PERSIST_CERT" ] || [ ! -f "$PERSIST_KEY" ]; then
      echo "ERROR: Persisted cert/key not found after copy."
      return 1
  fi

  # Brief pause before system file updates
  sleep 2
  
  # DIRECT FILE UPDATE - Copy certificates directly to system locations
  echo "Directly updating system certificate files..."
  cp -f "$PERSIST_CERT" "$SYS_CERT_PATH" || { echo "ERROR: cp to system cert path failed"; return 1; }
  
  # Brief pause between copying cert and key
  sleep 1
  
  cp -f "$PERSIST_KEY" "$SYS_KEY_PATH" || { echo "ERROR: cp to system key path failed"; return 1; }
  echo "System certificate files updated successfully."
  
  # Allow filesystem to settle
  sync
  sleep 3
  
  # Restart the web service
  echo "Restarting web service..."
  midclt call service.restart "http" >/dev/null 2>&1
  RESTART_STATUS=$?
  if [ $RESTART_STATUS -ne 0 ]; then
      echo "WARNING: Failed to restart web service (exit code: $RESTART_STATUS)."
      echo "You may need to manually restart the web service from the TrueNAS UI."
  else
      echo "Web service restarted successfully."
  fi
  
  # Allow web service to fully restart
  sleep 5
  
  # Verify certificate dates
  echo "Verifying certificate dates:"
  echo "System certificate file date:" 
  openssl x509 -in "$SYS_CERT_PATH" -noout -dates

  return 0
}

########################################
# COPY ONLY MODE
########################################
if [ "$COPY_ONLY" = true ]; then
  echo "Running in COPY-ONLY mode."
  if [ ! -f "$CERT_PATH" ] || [ ! -f "$KEY_PATH" ]; then
    echo "ERROR: Source certificate or key file not found."
    exit 1
  fi
  copy_to_truenas || { echo "ERROR: Copy failed."; exit 1; }
  exit 0
fi

########################################
# CHECK CERTIFICATE AGE BEFORE RENEWAL
########################################
if ! check_cert_age; then
    echo "Certificate renewal skipped based on age check."
    echo "====== Certificate Update Script Finished $(date) ======"
    exit 0
fi

########################################
# NORMAL / FORCED RENEWAL
########################################
ACME_COMMAND="/root/.acme.sh/acme.sh --dns dns_gd"
ACME_EXIT_STATUS=0

if [ "$FORCE" = true ]; then
  echo "Forcing a new certificate from Let's Encrypt..."
  if ! $ACME_COMMAND --issue --force -d "$CERT_DOMAIN" -d "*.$CERT_DOMAIN"; then
      echo "ERROR: acme.sh --issue --force encountered an error."
      exit 1
  fi
  
  # Allow time for file operations to complete
  sleep 3
  
else
  echo "Attempting a normal renewal..."
  $ACME_COMMAND --renew -d "$CERT_DOMAIN" -d "*.$CERT_DOMAIN" || ACME_EXIT_STATUS=$?
  if [ $ACME_EXIT_STATUS -ne 0 ] && [ $ACME_EXIT_STATUS -ne 2 ]; then
      echo "WARNING: acme.sh --renew exited with status $ACME_EXIT_STATUS."
  fi
  
  # Allow time for file operations to complete
  sleep 2
fi

if [ ! -f "$CERT_PATH" ] || [ ! -f "$KEY_PATH" ]; then
  echo "ERROR: Certificate or key file not found after acme.sh run."
  exit 1
fi

# Compare ACME certificate with system certificate
# Only update if they differ or if forced
SHOULD_IMPORT=false

if [ $ACME_EXIT_STATUS -eq 0 ]; then
  echo "acme.sh completed successfully with a new certificate."
  SHOULD_IMPORT=true
elif [ "$FORCE" = true ]; then
  echo "Force flag used - will update certificate."
  SHOULD_IMPORT=true
else
  # Check if system cert exists
  if [ ! -f "$SYS_CERT_PATH" ]; then
    echo "System certificate does not exist. Will copy the certificate."
    SHOULD_IMPORT=true
  else
    # Compare certificates
    echo "Comparing certificates..."
    ACME_FINGERPRINT=$(openssl x509 -in "$CERT_PATH" -noout -fingerprint -sha256 2>/dev/null)
    # Brief pause between operations
    sleep 1
    SYS_FINGERPRINT=$(openssl x509 -in "$SYS_CERT_PATH" -noout -fingerprint -sha256 2>/dev/null)
    
    if [ "$ACME_FINGERPRINT" != "$SYS_FINGERPRINT" ]; then
      echo "Certificates are different. Will update system certificate."
      echo "ACME fingerprint: $ACME_FINGERPRINT"
      echo "System fingerprint: $SYS_FINGERPRINT"
      SHOULD_IMPORT=true
    else
      echo "Certificates are identical. No update needed."
      SHOULD_IMPORT=false
    fi
  fi
fi

# Brief pause before proceeding
sleep 2

if [ "$SHOULD_IMPORT" = true ]; then
  echo "Proceeding with copy to TrueNAS..."
  copy_to_truenas || { echo "ERROR: Copy failed."; exit 1; }
else
  echo "No certificate update needed."
fi

echo "====== Certificate Update Script Finished $(date) ======"
exit 0