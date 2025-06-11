# Plan to Add New GitHub Actions Workflow: Complete Package Inventory

This plan outlines the steps to create a new GitHub Actions workflow file, `scan-packages.yml`, in the `.github/workflows/` directory. This workflow will execute the provided "Complete Package Inventory Script".

**1. Define the New Workflow File:**
   - **File Name:** `scan-packages.yml`
   - **Location:** `./.github/workflows/`

**2. Structure of `scan-packages.yml`:**
   The new workflow file will be structured similarly to the existing scan workflows.

   - **`name`**: `Complete Package Inventory`
   - **`on` trigger**:
     ```yaml
     on:
       push:
         branches: [development]
     ```
   - **`jobs`**:
     - **`package-inventory-audit`**:
       - `runs-on: ubuntu-latest`
       - `timeout-minutes: 20` (allowing extra time for `apt update` and extensive queries)
       - **`steps`**:
         1.  **Checkout repository**: Standard step to checkout the code.
             ```yaml
             - name: Checkout repository
               uses: actions/checkout@v4
             ```
         2.  **Run Package Inventory Script**: This step will execute the provided bash script on the target server via SSH.
             ```yaml
             - name: Run Package Inventory Script
               id: scan_packages_step
               uses: appleboy/ssh-action@master
               with:
                 host: ${{ secrets.SERVER_HOST }}
                 username: ${{ secrets.SERVER_USER }}
                 password: ${{ secrets.SERVER_PASSWORD }}
                 script: |
                   #!/bin/bash
                   # Complete Package Inventory Script
                   # Shows all installed packages with current version, available versions, and sizes

                   echo "=========================================="
                   echo "    COMPLETE PACKAGE INVENTORY AUDIT"
                   echo "=========================================="
                   echo "Generated at: $(date)"
                   echo "System: $(lsb_release -d 2>/dev/null | cut -f2 || cat /etc/debian_version)"
                   echo

                   # Update package cache first
                   echo "Updating package cache..."
                   sudo apt update >/dev/null 2>&1
                   echo "Cache updated."
                   echo

                   # Function to get human readable size
                   human_size() {
                       local size=$1
                       if [ "$size" -gt 1073741824 ]; then
                           echo "$(( size / 1073741824 ))GB"
                       elif [ "$size" -gt 1048576 ]; then
                           echo "$(( size / 1048576 ))MB"
                       elif [ "$size" -gt 1024 ]; then
                           echo "$(( size / 1024 ))KB"
                       else
                           echo "${size}B"
                       fi
                   }

                   # Function to check if version is newer
                   is_newer_version() {
                       local current="$1"
                       local available="$2"
                       # Use dpkg to compare versions
                       dpkg --compare-versions "$available" gt "$current" 2>/dev/null
                   }

                   echo "=========================================="
                   echo "         PACKAGE SUMMARY STATISTICS"
                   echo "=========================================="

                   echo "=== PACKAGE COUNTS ==="
                   echo "Total installed packages: $(dpkg -l | grep -c '^ii')"
                   echo "Upgradable packages: $(apt list --upgradable 2>/dev/null | grep -c upgradable)"
                   echo "Automatically installed: $(apt-mark showauto | wc -l)"
                   echo "Manually installed: $(apt-mark showmanual | wc -l)"
                   echo

                   echo "=== TOTAL DISK USAGE ==="
                   total_size=$(dpkg-query -Wf '${Installed-Size}\n' | awk '{sum+=$1} END {print sum*1024}')
                   echo "Total installed size: $(human_size $total_size)"
                   echo

                   echo "=== LARGEST PACKAGES (Top 20) ==="
                   dpkg-query -Wf '${Installed-Size}\t${Package}\t${Version}\n' | sort -nr | head -20 | while read size package version; do
                       size_bytes=$((size * 1024))
                       printf "%-15s %-40s %s\n" "$(human_size $size_bytes)" "$package" "$version"
                   done
                   echo

                   echo "=========================================="
                   echo "           UPGRADABLE PACKAGES"
                   echo "=========================================="

                   echo "=== PACKAGES WITH AVAILABLE UPDATES ==="
                   apt list --upgradable 2>/dev/null | grep -v "^Listing" | while IFS='/' read package rest; do
                       if [ -n "$package" ]; then
                           current_version=$(dpkg-query -Wf '${Version}' "$package" 2>/dev/null)
                           available_info=$(apt-cache policy "$package" 2>/dev/null | grep "Candidate:" | awk '{print $2}')
                           installed_size=$(dpkg-query -Wf '${Installed-Size}' "$package" 2>/dev/null)
                           size_human=$(human_size $((installed_size * 1024)))
                           
                           printf "%-30s %-20s -> %-20s %s\n" "$package" "$current_version" "$available_info" "$size_human"
                       fi
                   done | head -50
                   echo "... (showing first 50 upgradable packages)"
                   echo

                   echo "=========================================="
                   echo "         COMPLETE PACKAGE LISTING"
                   echo "=========================================="

                   echo "Creating comprehensive package report..."
                   echo "Format: PACKAGE | INSTALLED_VERSION | LATEST_AVAILABLE | SIZE | STATUS"
                   echo "========================================================================"

                   # Create temporary file for processing
                   temp_file=$(mktemp)

                   # Get all installed packages
                   dpkg -l | grep '^ii' | awk '{print $2}' | while read package; do
                       # Get current installed version
                       current_version=$(dpkg-query -Wf '${Version}' "$package" 2>/dev/null)
                       
                       # Get installed size
                       installed_size=$(dpkg-query -Wf '${Installed-Size}' "$package" 2>/dev/null)
                       size_human=$(human_size $((installed_size * 1024)))
                       
                       # Get available versions from apt
                       policy_output=$(apt-cache policy "$package" 2>/dev/null)
                       candidate_version=$(echo "$policy_output" | grep "Candidate:" | awk '{print $2}')
                       
                       # Determine status
                       status="CURRENT"
                       if [ "$candidate_version" != "$current_version" ] && [ -n "$candidate_version" ] && [ "$candidate_version" != "(none)" ]; then
                           if is_newer_version "$current_version" "$candidate_version"; then
                               status="UPDATE_AVAILABLE"
                           fi
                       fi
                       
                       # Get all available versions
                       all_versions=$(echo "$policy_output" | grep -A 20 "Version table:" | grep "^ \*\*\*\|^     " | awk '{print $1}' | grep -v "^\*\*\*" | head -3 | tr '\n' ' ')
                       
                       printf "%-35s | %-25s | %-25s | %-10s | %s\n" \
                           "$package" \
                           "$current_version" \
                           "${candidate_version:-N/A}" \
                           "$size_human" \
                           "$status"
                           
                   done | tee "$temp_file"

                   echo
                   echo "=========================================="
                   echo "         SECURITY UPDATE ANALYSIS"
                   echo "=========================================="

                   echo "=== SECURITY UPDATES AVAILABLE ==="
                   apt list --upgradable 2>/dev/null | grep -i security | head -20 || echo "No obvious security updates found in upgrade list"
                   echo

                   echo "=== PACKAGES FROM SECURITY REPOSITORY ==="
                   apt-cache policy | grep -A 1 security | head -20 || echo "No security repositories configured"
                   echo

                   echo "=========================================="
                   echo "         PACKAGE SOURCE ANALYSIS"
                   echo "=========================================="

                   echo "=== PACKAGES BY REPOSITORY ==="
                   dpkg -l | grep '^ii' | awk '{print $2}' | head -50 | while read package; do
                       origin=$(apt-cache policy "$package" 2>/dev/null | grep "^ \*\*\*" -A 1 | tail -1 | awk '{print $2}' | head -1)
                       echo "$package: $origin"
                   done | sort -k2 | uniq -f1 -c | sort -nr | head -10
                   echo "... (showing top 10 repository sources for first 50 packages)"
                   echo

                   echo "=== NON-OFFICIAL PACKAGES ==="
                   dpkg -l | grep '^ii' | awk '{print $2}' | while read package; do
                       policy=$(apt-cache policy "$package" 2>/dev/null)
                       if echo "$policy" | grep -q "now.*local\|now.*unknown"; then
                           version=$(dpkg-query -Wf '${Version}' "$package" 2>/dev/null)
                           echo "$package ($version) - locally installed or unknown source"
                       fi
                   done | head -20
                   echo

                   echo "=========================================="
                   echo "         PACKAGE MAINTENANCE INFO"
                   echo "=========================================="

                   echo "=== PACKAGES THAT CAN BE REMOVED ==="
                   echo "--- Automatically installed and no longer needed ---"
                   apt autoremove --dry-run 2>/dev/null | grep "^Remv" | head -10 || echo "No packages marked for autoremoval"
                   echo

                   echo "--- Orphaned packages (deborphan if available) ---"
                   if command -v deborphan >/dev/null 2>&1; then
                       deborphan | head -10 || echo "No orphaned packages found"
                   else
                       echo "deborphan not installed (install with: sudo apt install deborphan)"
                   fi
                   echo

                   echo "=== RECENTLY INSTALLED PACKAGES ==="
                   grep "install " /var/log/dpkg.log 2>/dev/null | tail -10 | while read date time action package version; do
                       echo "$date $time: $package ($version)"
                   done || echo "Cannot access dpkg log"
                   echo

                   echo "=== RECENTLY UPGRADED PACKAGES ==="
                   grep "upgrade " /var/log/dpkg.log 2>/dev/null | tail -10 | while read date time action package old_version new_version; do
                       echo "$date $time: $package ($old_version -> $new_version)"
                   done || echo "Cannot access dpkg log"
                   echo

                   echo "=========================================="
                   echo "         PACKAGE ANALYSIS SUMMARY"
                   echo "=========================================="

                   # Analysis of the temp file
                   if [ -f "$temp_file" ]; then
                       echo "=== PACKAGE STATUS BREAKDOWN ==="
                       echo "Packages needing updates: $(grep "UPDATE_AVAILABLE" "$temp_file" | wc -l)"
                       echo "Packages current: $(grep "CURRENT" "$temp_file" | wc -l)"
                       echo
                       
                       echo "=== SIZE ANALYSIS ==="
                       echo "Packages > 100MB:"
                       awk -F'|' '{print $4}' "$temp_file" | grep -c "GB\|[0-9][0-9][0-9]MB"
                       echo "Packages > 10MB:"
                       awk -F'|' '{print $4}' "$temp_file" | grep -c "GB\|[0-9][0-9]MB\|[0-9][0-9][0-9]MB"
                       
                       rm "$temp_file"
                   fi

                   echo
                   echo "=== MAINTENANCE RECOMMENDATIONS ==="
                   echo "1. Run 'sudo apt upgrade' to update $(apt list --upgradable 2>/dev/null | grep -c upgradable) packages"
                   echo "2. Run 'sudo apt autoremove' to clean up unneeded packages"
                   echo "3. Run 'sudo apt autoclean' to clear package cache"
                   echo "4. Consider reviewing large packages for necessity"
                   echo "5. Check for security updates regularly"
                   echo

                   echo "=========================================="
                   echo "Package inventory completed at $(date)"
                   echo "=========================================="

                   echo
                   echo "=== USEFUL FOLLOW-UP COMMANDS ==="
                   echo "# List all packages sorted by size:"
                   echo "dpkg-query -Wf '\${Installed-Size}\\t\${Package}\\t\${Version}\\n' | sort -nr"
                   echo
                   echo "# Check specific package details:"
                   echo "apt show <package-name>"
                   echo
                   echo "# See what files a package installed:"
                   echo "dpkg -L <package-name>"
                   echo
                   echo "# Find which package owns a file:"
                   echo "dpkg -S /path/to/file"
                 ```
         3.  **Save Script Output**: Saves `stdout` and `stderr` from the script to files.
             ```yaml
             - name: Save Script Output
               if: always()
               run: |
                 echo "Saving script stdout to package_scan_output.txt"
                 echo "${{ steps.scan_packages_step.outputs.stdout }}" > package_scan_output.txt
                 echo "Saving script stderr to package_scan_error.txt"
                 echo "${{ steps.scan_packages_step.outputs.stderr }}" > package_scan_error.txt
                 touch package_scan_output.txt package_scan_error.txt # Ensure files exist for artifact upload
                 echo "Contents of package_scan_output.txt (first 5 lines):"
                 head -n 5 package_scan_output.txt
                 echo "Contents of package_scan_error.txt (first 5 lines):"
                 head -n 5 package_scan_error.txt
             ```
         4.  **Upload Package Scan Results**: Uploads the output files as build artifacts.
             ```yaml
             - name: Upload Package Scan Results
               if: always()
               uses: actions/upload-artifact@v4
               with:
                 name: package-scan-results-${{ github.run_number }}
                 path: |
                   package_scan_output.txt
                   package_scan_error.txt
                 retention-days: 30
             ```
         5.  **Notify on failure**: Sends a notification if the workflow step fails.
             ```yaml
             - name: Notify on failure
               if: failure()
               run: |
                 echo "Complete Package Inventory scan failed. Check the logs for details."
                 echo "::error::Complete Package Inventory workflow failed"