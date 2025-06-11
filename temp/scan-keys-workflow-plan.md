# Plan for System Key Inventory GitHub Actions Workflow

This document outlines the plan to create a new GitHub Actions workflow for performing a system key inventory.

**1. Workflow File:**
*   **Name:** `scan-keys.yml`
*   **Location:** `.github/workflows/`

**2. Workflow Configuration:**
*   **Display Name:** "System Key Inventory"
*   **Trigger:** On `push` to the `development` branch.
*   **Runner:** `ubuntu-latest`.
*   **Timeout:** 10 minutes.

**3. Workflow Steps:**
    *   **Checkout Repository:** Uses `actions/checkout@v4`.
    *   **Run Key Inventory Script:**
        *   Uses `appleboy/ssh-action@master` to connect to the server.
        *   Authentication will use `host: ${{ secrets.SERVER_HOST }}`, `username: ${{ secrets.SERVER_USER }}`, and `password: ${{ secrets.SERVER_PASSWORD }}`.
        *   The full Bash script (as provided by the user, with minor robustness adjustments) will be embedded in the `script` parameter.
        *   This step will be given an `id` (e.g., `scan_keys_step`) to access its outputs (stdout/stderr).
    *   **Save Script Output:**
        *   A new step will run `if: always()` to ensure it executes even if the script step has issues.
        *   It will take `steps.scan_keys_step.outputs.stdout` and save it to a file named `key_scan_output.txt` in the runner's workspace.
        *   It will take `steps.scan_keys_step.outputs.stderr` and save it to a file named `key_scan_error.txt` in the runner's workspace.
        *   Includes `touch` commands to ensure these files are created, even if empty, to prevent issues with the artifact upload step.
    *   **Upload Key Scan Results:**
        *   Uses `actions/upload-artifact@v4`.
        *   Runs `if: always()`.
        *   Artifact name: `key-scan-results-${{ github.run_number }}`.
        *   Uploads `key_scan_output.txt` and `key_scan_error.txt`.
        *   Retention period: 30 days.
    *   **Notify on Failure:**
        *   Runs `if: failure()`.
        *   Prints an error message to the logs and uses `echo "::error::..."` to mark the workflow as failed prominently.

**Proposed `scan-keys.yml` content:**

```yaml
name: System Key Inventory

on:
  push:
    branches: [development]

jobs:
  key-inventory:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Run Key Inventory Script
        id: scan_keys_step
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          password: ${{ secrets.SERVER_PASSWORD }}
          script: |
            #!/bin/bash
            # System Key Inventory Script
            # Provides overview of cryptographic keys on the system

            echo "=========================================="
            echo "         SYSTEM KEY INVENTORY"
            echo "=========================================="
            echo

            # SSH Host Keys
            echo "=== SSH HOST KEYS ==="
            if [ -d "/etc/ssh" ]; then
                for key_file in /etc/ssh/ssh_host_*.pub; do
                    if [ -f "$key_file" ]; then
                        echo "Key: $(basename "$key_file")"
                        ssh-keygen -l -f "$key_file"
                        echo
                    fi
                done
            else
                echo "No SSH host keys found"
            fi

            echo "=== USER SSH KEYS ==="
            if [ -d "$HOME/.ssh" ]; then
                echo "Directory: $HOME/.ssh"
                ls -la ~/.ssh/
                echo
                
                for key_file in ~/.ssh/id_*.pub; do
                    if [ -f "$key_file" ]; then
                        echo "Public key: $(basename "$key_file")"
                        cat "$key_file"
                        echo
                    fi
                done
                
                if [ -f "$HOME/.ssh/authorized_keys" ]; then
                    echo "Authorized keys count: $(wc -l < ~/.ssh/authorized_keys)"
                fi
            else
                echo "No user SSH directory found"
            fi

            echo
            echo "=== APT PACKAGE SIGNING KEYS ==="
            if [ -d "/etc/apt/trusted.gpg.d" ]; then
                echo "Trusted keyrings:"
                ls -la /etc/apt/trusted.gpg.d/ | grep -v "^total"
                echo
            fi

            if [ -d "/usr/share/keyrings" ]; then
                echo "System keyrings:"
                ls -la /usr/share/keyrings/ | grep -v "^total" | head -10
                echo
            fi

            echo "=== GPG KEYS ==="
            echo "Personal GPG keys:"
            gpg --list-keys 2>/dev/null | head -20 || echo "No personal GPG keys found"

            echo
            echo "=== SYSTEM IDENTIFIERS ==="
            echo "Machine ID: $(cat /etc/machine-id 2>/dev/null || echo 'Not found')"
            echo "Hostname: $(hostname)"
            echo "Boot ID: $(cat /proc/sys/kernel/random/boot_id 2>/dev/null || echo 'Not found')"

            echo
            echo "=== RUNNING KERNEL KEY TYPES ==="
            echo "Loaded asymmetric keys:"
            sudo cat /proc/keys 2>/dev/null | grep -i asymmetric | wc -l || echo "Cannot access /proc/keys or no asymmetric keys found"

            echo
            echo "=== POTENTIAL SENSITIVE FILES ==="
            echo "Searching for key/certificate files in /etc (first 10):"
            sudo find /etc -name "*.key" -o -name "*.pem" -o -name "*_key" -o -name "*.crt" 2>/dev/null | head -10 || echo "None found or error searching"

            echo
            echo "=== SSL/TLS SERVICE KEYS ==="
            for dir in "/etc/ssl/private" "/etc/pki/tls/private" "/etc/apache2/ssl" "/etc/nginx/ssl"; do
                if [ -d "$dir" ]; then
                    echo "Found SSL directory: $dir"
                    sudo ls -la "$dir" 2>/dev/null | head -5
                fi
            done

            echo
            echo "=========================================="
            echo "Key inventory completed at $(date)"
            echo "=========================================="

      - name: Save Script Output
        if: always()
        run: |
          echo "Saving script stdout to key_scan_output.txt"
          echo "${{ steps.scan_keys_step.outputs.stdout }}" > key_scan_output.txt
          echo "Saving script stderr to key_scan_error.txt"
          echo "${{ steps.scan_keys_step.outputs.stderr }}" > key_scan_error.txt
          
          # Ensure files exist for artifact upload, even if empty
          touch key_scan_output.txt key_scan_error.txt
          
          echo "Contents of key_scan_output.txt (first 5 lines):"
          head -n 5 key_scan_output.txt
          echo "Contents of key_scan_error.txt (first 5 lines):"
          head -n 5 key_scan_error.txt


      - name: Upload Key Scan Results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: key-scan-results-${{ github.run_number }}
          path: |
            key_scan_output.txt
            key_scan_error.txt
          retention-days: 30

      - name: Notify on failure
        if: failure()
        run: |
          echo "System Key Inventory scan failed. Check the logs for details."
          echo "::error::System Key Inventory workflow failed"