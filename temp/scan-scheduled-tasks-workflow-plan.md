# Workflow Plan: scan-scheduled-tasks.yml

This document outlines the plan for creating a GitHub Actions workflow to audit scheduled tasks on a server.

## 1. Workflow Name
- Set the name of the workflow to "Scheduled Tasks Audit".

## 2. Trigger
- Configure the workflow to run on `push` events to the `development` branch.
  ```yaml
  on:
    push:
      branches: [development]
  ```

## 3. Job: `audit-scheduled-tasks`
- Define a single job named `audit-scheduled-tasks`.
- **Runner:** Use `ubuntu-latest`.
- **Timeout:** Set a timeout of `10` minutes.
- **Steps:**
    1.  **Checkout Repository:**
        - Use the `actions/checkout@v4` action to allow the workflow to access the repository's content.
        ```yaml
        - name: Checkout repository
          uses: actions/checkout@v4
        ```
    2.  **Run Scheduled Tasks Audit Script:**
        - Use the `appleboy/ssh-action@master` action to execute the bash script on a remote server.
        - Assign an `id` to this step (e.g., `run_audit_script_step`) to reference its outputs.
        - Configure `with` parameters:
            - `host`: `\${{ secrets.SERVER_HOST }}`
            - `username`: `\${{ secrets.SERVER_USER }}`
            - `password`: `\${{ secrets.SERVER_PASSWORD }}`
            - `script`: (The multi-line "Complete Scheduled Tasks Audit Script" provided in the initial request)
        ```yaml
        - name: Run Scheduled Tasks Audit Script
          id: run_audit_script_step
          uses: appleboy/ssh-action@master
          with:
            host: \${{ secrets.SERVER_HOST }}
            username: \${{ secrets.SERVER_USER }}
            password: \${{ secrets.SERVER_PASSWORD }}
            script: |
              #!/bin/bash
              # Complete Scheduled Tasks Audit Script
              # Shows all cron jobs, systemd timers, and other scheduled tasks

              echo "=========================================="
              echo "    COMPLETE SCHEDULED TASKS AUDIT"
              echo "=========================================="
              echo "Generated at: $(date)"
              echo

              # Function to show file content with header
              show_file() {
                  local file="$1"
                  local description="$2"
                  
                  if [ -f "$file" ] && [ -s "$file" ]; then
                      echo "=== $description ==="
                      echo "File: $file"
                      echo "Last modified: $(stat -c %y "$file" 2>/dev/null)"
                      echo "--- Content ---"
                      cat "$file"
                      echo
                  fi
              }

              # Function to show directory contents
              show_cron_dir() {
                  local dir="$1"
                  local description="$2"
                  
                  if [ -d "$dir" ] && [ "$(ls -A "$dir" 2>/dev/null)" ]; then
                      echo "=== $description ==="
                      echo "Directory: $dir"
                      for file in "$dir"/*; do
                          if [ -f "$file" ]; then
                              echo
                              echo "--- File: $(basename "$file") ---"
                              echo "Last modified: $(stat -c %y "$file" 2>/dev/null)"
                              cat "$file"
                              echo
                          fi
                      done
                  fi
              }

              echo "=========================================="
              echo "           SYSTEMD TIMERS"
              echo "=========================================="

              echo "=== ACTIVE SYSTEMD TIMERS ==="
              systemctl list-timers --all --no-pager
              echo

              echo "=== SYSTEMD TIMER DETAILS ==="
              systemctl list-timers --all --no-pager | tail -n +2 | head -n -3 | awk '{print $1}' | while read timer; do
                  if [ -n "$timer" ] && [ "$timer" != "NEXT" ]; then
                      echo "--- Timer: $timer ---"
                      systemctl show "$timer" --property=Id,Description,NextElapseUSecRealtime,LastTriggerUSec,ActiveState,UnitFileState 2>/dev/null
                      echo
                  fi
              done

              echo "=========================================="
              echo "           SYSTEM CRON JOBS"
              echo "=========================================="

              # Main system crontab
              show_file "/etc/crontab" "MAIN SYSTEM CRONTAB"

              # System cron directories
              show_cron_dir "/etc/cron.d" "SYSTEM CRON.D JOBS"
              show_cron_dir "/etc/cron.hourly" "HOURLY CRON JOBS"
              show_cron_dir "/etc/cron.daily" "DAILY CRON JOBS" 
              show_cron_dir "/etc/cron.weekly" "WEEKLY CRON JOBS"
              show_cron_dir "/etc/cron.monthly" "MONTHLY CRON JOBS"

              echo "=========================================="
              echo "            USER CRON JOBS"
              echo "=========================================="

              echo "=== CURRENT USER CRON JOBS ==="
              echo "User: $(whoami)"
              crontab -l 2>/dev/null || echo "No crontab for current user"
              echo

              echo "=== ALL USER CRON JOBS ==="
              # Check for other users with crontabs
              if [ -d "/var/spool/cron/crontabs" ]; then
                  for user_cron in /var/spool/cron/crontabs/*; do
                      if [ -f "$user_cron" ]; then
                          username=$(basename "$user_cron")
                          echo "--- User: $username ---"
                          echo "File: $user_cron"
                          echo "Last modified: $(stat -c %y "$user_cron" 2>/dev/null)"
                          sudo cat "$user_cron" 2>/dev/null || echo "Cannot read crontab (permission denied)"
                          echo
                      fi
                  done
              else
                  echo "No user crontabs directory found"
              fi

              echo "=========================================="
              echo "             ANACRON JOBS"
              echo "=========================================="

              show_file "/etc/anacrontab" "ANACRON CONFIGURATION"

              echo "=========================================="
              echo "              AT JOBS"
              echo "=========================================="

              echo "=== AT QUEUE ==="
              atq 2>/dev/null || echo "No at jobs scheduled or at command not available"
              echo

              echo "=== AT JOBS DETAILS ==="
              atq 2>/dev/null | while read job_id queue user date time; do
                  if [ -n "$job_id" ] && [ "$job_id" != "no" ]; then
                      echo "--- Job ID: $job_id (User: $user, Date: $date $time) ---"
                      at -c "$job_id" 2>/dev/null | tail -n 20 || echo "Cannot read job details"
                      echo
                  fi
              done

              echo "=========================================="
              echo "           SYSTEMD SERVICES"
              echo "         (with Timer dependency)"
              echo "=========================================="

              echo "=== SERVICES TRIGGERED BY TIMERS ==="
              systemctl list-timers --all --no-pager | tail -n +2 | head -n -3 | awk '{print $NF}' | while read service; do
                  if [ -n "$service" ] && [ "$service" != "ACTIVATES" ]; then
                      echo "--- Service: $service ---"
                      systemctl show "$service" --property=Id,Description,Type,ExecStart --no-pager 2>/dev/null
                      echo
                  fi
              done

              echo "=========================================="
              echo "      SCHEDULED TASKS SUMMARY"
              echo "=========================================="

              echo "=== COUNTS ==="
              echo "Active systemd timers: $(systemctl list-timers --state=active --no-legend | wc -l)"
              echo "System cron files: $(find /etc/cron.d /etc/cron.{hourly,daily,weekly,monthly} -type f 2>/dev/null | wc -l)"
              echo "User crontabs: $(find /var/spool/cron/crontabs -type f 2>/dev/null | wc -l)"
              echo "At jobs: $(atq 2>/dev/null | wc -l)"

              echo
              echo "=== NEXT 5 SCHEDULED EVENTS ==="
              {
                  # Get systemd timer next runs
                  systemctl list-timers --no-legend 2>/dev/null | head -5 | awk '{print $1 " " $2 " (systemd timer)"}'
                  
                  # Get at jobs
                  atq 2>/dev/null | head -5 | awk '{print $2 " " $3 " " $4 " " $5 " (at job)"}'
              } | sort

              echo
              echo "=========================================="
              echo "Scheduled tasks audit completed at $(date)"
              echo "=========================================="
        ```
    3.  **Save Script Output:**
        - This step will run `if: always()` to ensure output is saved even if the script step fails.
        - Use a `run` block to:
            - Save the `stdout` of the script step to `scheduled_tasks_audit_output.txt`.
            - Save the `stderr` of the script step to `scheduled_tasks_audit_error.txt`.
            - Ensure both files exist for artifact upload: `touch scheduled_tasks_audit_output.txt scheduled_tasks_audit_error.txt`.
        ```yaml
        - name: Save Script Output
          if: always()
          run: |
            echo "Saving script stdout to scheduled_tasks_audit_output.txt"
            echo "\${{ steps.run_audit_script_step.outputs.stdout }}" > scheduled_tasks_audit_output.txt
            echo "Saving script stderr to scheduled_tasks_audit_error.txt"
            echo "\${{ steps.run_audit_script_step.outputs.stderr }}" > scheduled_tasks_audit_error.txt
            touch scheduled_tasks_audit_output.txt scheduled_tasks_audit_error.txt
        ```
    4.  **Upload Audit Results:**
        - This step will run `if: always()`.
        - Use the `actions/upload-artifact@v4` action.
        - Configure `with` parameters:
            - `name`: `scheduled-tasks-audit-results-\${{ github.run_number }}`
            - `path`:
              ```
              scheduled_tasks_audit_output.txt
              scheduled_tasks_audit_error.txt
              ```
            - `retention-days`: `30`.
        ```yaml
        - name: Upload Audit Results
          if: always()
          uses: actions/upload-artifact@v4
          with:
            name: scheduled-tasks-audit-results-\${{ github.run_number }}
            path: |
              scheduled_tasks_audit_output.txt
              scheduled_tasks_audit_error.txt
            retention-days: 30
        ```
    5.  **Notify on Failure:**
        - This step will run `if: failure()`.
        - Use a `run` block to echo error messages.
        ```yaml
        - name: Notify on failure
          if: failure()
          run: |
            echo "Scheduled Tasks Audit scan failed. Check the logs for details."
            echo "::error::Scheduled Tasks Audit workflow failed"