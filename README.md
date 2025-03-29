# TrueNAS SCALE Certificate Management with acme.sh

This guide explains how to automate SSL/TLS certificate management on TrueNAS SCALE Electric Eel (24.10.2) using acme.sh and DNS validation.

## Overview

This solution allows you to:

1. Obtain wildcard certificates via Let's Encrypt or ZeroSSL using DNS validation
2. Automatically renew certificates before expiration
3. Update TrueNAS web UI and services to use the renewed certificate

## Prerequisites

- TrueNAS SCALE Electric Eel (24.10.2) or later
- Root shell access to your TrueNAS system
- Domain name with DNS hosted at a supported provider (this guide uses GoDaddy)
- API credentials for your DNS provider

## Installation Steps

### 1. Install acme.sh

Access your TrueNAS shell as root and install acme.sh:

```bash
curl https://get.acme.sh | sh
```

This installs acme.sh to `/root/.acme.sh/` and adds a cron job for automatic renewals.

### 2. Set up DNS API credentials

Create an environment file for your DNS provider. For GoDaddy:

```bash
mkdir -p /mnt/tank/acme  # Adjust the pool name as needed
cd /mnt/tank/acme
cat > .godaddy.env << EOF
export GD_Key="your_api_key"
export GD_Secret="your_api_secret"
EOF
chmod 600 .godaddy.env
```

### 3. Issue your first certificate

Run the following command to issue a wildcard certificate:

```bash
source ./.godaddy.env
/root/.acme.sh/acme.sh --issue --dns dns_gd -d yourdomain.com -d "*.yourdomain.com"
```

This creates certificate files in `/root/.acme.sh/yourdomain.com_ecc/`.

### 4. Create the update script

Create a file named `update-cert.sh` in your persistence directory:

```bash
vi /mnt/tank/acme/update-cert.sh
```

Copy the script from the next section into this file, then make it executable:

```bash
chmod +x /mnt/tank/acme/update-cert.sh
```

### 5. Create initial certificate in TrueNAS UI

1. Go to Credentials → Certificates
2. Click "Add"
3. Choose "Import Certificate"
4. Enter a name (e.g., "letsencrypt")
5. Import any temporary certificate (you can use the TrueNAS default)
6. Save the certificate

### 6. Run the update script

Execute the script with the `--cp` flag to update the certificate in TrueNAS:

```bash
cd /mnt/tank/acme
./update-cert.sh --cp
```

### 7. Set up automated renewal

Edit the root crontab:

```bash
crontab -e
```

Add this line to run the script monthly:

```
0 0 1 * * /mnt/tank/acme/update-cert.sh > /mnt/tank/acme/renewal.log 2>&1
```

## The Certificate Update Script

Save this as `update-cert.sh` and customize the variables at the top:

```bash
#!/usr/bin/env bash
set -euo pipefail

########################################
# CONFIGURATION / PATHS
########################################

GD_ENV="./.godaddy.env"
CERT_DOMAIN="yourdomain.com"  # Change to your domain

ACME_BASE="/root/.acme.sh/${CERT_DOMAIN}_ecc"
CERT_PATH="${ACME_BASE}/fullchain.cer"
KEY_PATH="${ACME_BASE}/${CERT_DOMAIN}.key"

PERSIST_BASE="/mnt/tank/acme/certs"  # Change to your pool/dataset
PERSIST_CERT="${PERSIST_BASE}/${CERT_DOMAIN}-fullchain.cer"
PERSIST_KEY="${PERSIST_BASE}/${CERT_DOMAIN}.key"

# Certificate name in TrueNAS (must match name in UI)
TRUENAS_CERT_NAME="letsencrypt"

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
# LOAD DNS ENV VARIABLES
########################################
if [ "$COPY_ONLY" = false ]; then
  if [ -f "$GD_ENV" ]; then
    . "$GD_ENV"
  else
    echo "ERROR: DNS provider env file not found at $GD_ENV" >&2
    exit 1
  fi

  if [ -z "${GD_Key:-}" ] || [ -z "${GD_Secret:-}" ]; then
    echo "ERROR: GD_Key or GD_Secret not set. Check $GD_ENV." >&2
    exit 1
  fi
fi

########################################
# FUNCTION: COPY & IMPORT TO TRUENAS
########################################
copy_and_import_to_truenas() {
  # Ensure persistence directory exists and copy files there
  mkdir -p "$PERSIST_BASE" || { echo "ERROR: mkdir failed" >&2; return 1; }
  cp -f "$CERT_PATH" "$PERSIST_CERT" || { echo "ERROR: cp cert failed" >&2; return 1; }
  cp -f "$KEY_PATH" "$PERSIST_KEY"   || { echo "ERROR: cp key failed" >&2; return 1; }

  if [ ! -f "$PERSIST_CERT" ] || [ ! -f "$PERSIST_KEY" ]; then
      echo "ERROR: Persisted cert/key not found after copy." >&2
      return 1
  fi

  if ! command -v jq >/dev/null; then
      echo "ERROR: jq command not found. Please install jq." >&2
      return 1
  fi

  # Read the certificate and key files
  CERT_CONTENT=$(cat "$PERSIST_CERT") || { echo "ERROR: Failed to read certificate file." >&2; return 1; }
  KEY_CONTENT=$(cat "$PERSIST_KEY")   || { echo "ERROR: Failed to read key file." >&2; return 1; }

  if [ -z "$CERT_CONTENT" ] || [ -z "$KEY_CONTENT" ]; then
      echo "ERROR: Certificate or key content is empty." >&2
      return 1
  fi

  # Find the existing certificate by name
  echo "Looking for existing certificate with name '$TRUENAS_CERT_NAME'..."
  EXISTING_CERT=$(midclt call certificate.query '[["name","=","'"$TRUENAS_CERT_NAME"'"]]' 2>&1)
  EXISTING_CERT_ID=$(echo "$EXISTING_CERT" | jq -r '.[0].id // empty' 2>/dev/null)
  
  if [ -z "$EXISTING_CERT_ID" ]; then
      echo "ERROR: Certificate with name '$TRUENAS_CERT_NAME' not found in TrueNAS." >&2
      echo "Please create a certificate with name '$TRUENAS_CERT_NAME' in the TrueNAS web interface first." >&2
      echo "Available certificates:"
      midclt call certificate.query | jq -r '.[] | "ID: \(.id), Name: \(.name)"'
      return 1
  fi
  
  echo "Found certificate with ID: $EXISTING_CERT_ID"
  
  # Update the existing certificate
  echo "Updating existing certificate..."
  JSON_DATA=$(jq -n --arg cert "$CERT_CONTENT" --arg key "$KEY_CONTENT" '{
    "certificate": $cert,
    "privatekey": $key
  }') || { echo "ERROR: Failed to create JSON payload." >&2; return 1; }
  
  UPDATE_OUTPUT=$(midclt call certificate.update "$EXISTING_CERT_ID" "$JSON_DATA" 2>&1)
  UPDATE_STATUS=$?
  
  if [ $UPDATE_STATUS -ne 0 ]; then
      echo "ERROR: Failed to update certificate: $UPDATE_OUTPUT" >&2
      return 1
  fi
  
  echo "Certificate updated successfully."
  
  # Update the UI certificate
  echo "Setting certificate as the UI certificate..."
  UPDATE_OUTPUT=$(midclt call system.general.update "{\"ui_certificate\":$EXISTING_CERT_ID}" 2>&1)
  UPDATE_EXIT_STATUS=$?
  
  if [ $UPDATE_EXIT_STATUS -ne 0 ]; then
      echo "WARNING: Failed to set the certificate as the UI certificate: $UPDATE_OUTPUT" >&2
      echo "You may need to manually set this certificate as the UI certificate in the TrueNAS web interface."
  else
      echo "UI certificate updated successfully."
  fi
  
  return 0
}

########################################
# COPY ONLY MODE
########################################
if [ "$COPY_ONLY" = true ]; then
  echo "Running in COPY-ONLY mode."
  if [ ! -f "$CERT_PATH" ] || [ ! -f "$KEY_PATH" ]; then
    echo "ERROR: Source certificate or key file not found." >&2
    exit 1
  fi
  copy_and_import_to_truenas || { echo "ERROR: Import failed." >&2; exit 1; }
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
      echo "ERROR: acme.sh --issue --force encountered an error." >&2
      exit 1
  fi
else
  echo "Attempting a normal renewal..."
  $ACME_COMMAND --renew -d "$CERT_DOMAIN" -d "*.$CERT_DOMAIN" || ACME_EXIT_STATUS=$?
  if [ $ACME_EXIT_STATUS -ne 0 ] && [ $ACME_EXIT_STATUS -ne 2 ]; then
      echo "WARNING: acme.sh --renew exited with status $ACME_EXIT_STATUS." >&2
  fi
fi

if [ ! -f "$CERT_PATH" ] || [ ! -f "$KEY_PATH" ]; then
  echo "ERROR: Certificate or key file not found after acme.sh run." >&2
  exit 1
fi

MODIFICATION_THRESHOLD_SECONDS=$((60 * 60 * 24))
CURRENT_TIME=$(date +%s)
CERT_MTIME=$(stat -c %Y "$CERT_PATH" 2>/dev/null || date -r "$CERT_PATH" +%s 2>/dev/null || echo 0)
TIME_DIFF=$((CURRENT_TIME - CERT_MTIME))

SHOULD_IMPORT=false
if [ $ACME_EXIT_STATUS -eq 0 ]; then
    echo "acme.sh completed successfully."
    SHOULD_IMPORT=true
elif [ $TIME_DIFF -lt $MODIFICATION_THRESHOLD_SECONDS ]; then
    echo "Certificate file appears recently modified."
    SHOULD_IMPORT=true
fi

if [ "$SHOULD_IMPORT" = true ]; then
  echo "Proceeding with import into TrueNAS..."
  copy_and_import_to_truenas || { echo "ERROR: Import failed." >&2; exit 1; }
else
  if [ "$FORCE" = true ]; then
    echo "WARNING: --force was used, but certificate file does not seem updated." >&2
  else
    echo "No new certificate issued. Skipping import."
  fi
fi

echo "Script finished."
exit 0
```

## Understanding the midclt Command

The `midclt` command is TrueNAS SCALE's middleware client tool for interacting with the backend API. Here's how to use it effectively in scripts:

### Basic Syntax

```bash
midclt call <namespace>.<method> [arguments]
```

### Common Certificate Operations

1. **List all certificates**:
   ```bash
   midclt call certificate.query
   ```

2. **Get a specific certificate by name**:
   ```bash
   midclt call certificate.query '[["name","=","letsencrypt"]]'
   ```

3. **Update a certificate**:
   ```bash
   midclt call certificate.update <id> '{"certificate":"<cert-content>","privatekey":"<key-content>"}'
   ```

4. **Set a certificate as the UI certificate**:
   ```bash
   midclt call system.general.update '{"ui_certificate":<cert-id>}'
   ```

### Tips for Using midclt in Scripts

1. **Always capture output and exit status**:
   ```bash
   OUTPUT=$(midclt call some.command 2>&1)
   STATUS=$?
   if [ $STATUS -ne 0 ]; then
       echo "Error: $OUTPUT" >&2
   fi
   ```

2. **Use jq to parse JSON responses**:
   ```bash
   CERT_ID=$(midclt call certificate.query '[["name","=","letsencrypt"]]' | jq -r '.[0].id // empty')
   ```

3. **Handle errors gracefully**:
   ```bash
   if [ -z "$CERT_ID" ]; then
       echo "Certificate not found" >&2
       # Provide fallback behavior
   fi
   ```

4. **Testing midclt commands**:
   You can test commands directly in the shell before adding them to scripts.

## Troubleshooting

### Certificate Not Found

If the script can't find your certificate:
```
ERROR: Certificate with name 'letsencrypt' not found in TrueNAS.
```

Make sure you've created a certificate with the exact name specified in `TRUENAS_CERT_NAME` in the TrueNAS UI.

### acme.sh Errors

If acme.sh reports rate limiting:
```
The retryafter=86400 value is too large (> 600), will not retry anymore.
```

This is normal if you've recently requested a certificate. Use `--cp` to update without renewal.

### UI Certificate Update Failure

If setting the UI certificate fails, you can manually set it in the TrueNAS UI:
1. Go to System Settings → General
2. Select your certificate in the "GUI SSL Certificate" dropdown
3. Save changes

## License

This project is licensed under the MIT License - see the LICENSE file for details.
