# Plan for New GitHub Actions Workflow: Process and Service Audit

This plan outlines the steps to create a new GitHub Actions workflow for performing a complete process and service audit on a target server.

## 1. Workflow File Creation

*   **File Name**: `.github/workflows/scan-processes-services.yml`
*   **Purpose**: To automate the execution of a comprehensive audit script that checks running processes, services, and potential security concerns.

## 2. Workflow Configuration

*   **Name**: `Complete Process and Service Audit`
*   **Trigger**: The workflow will be triggered on `push` events to the `development` branch.
    ```yaml
    on:
      push:
        branches: [development]
    ```

## 3. Job Details

The workflow will contain a single job:

*   **Job ID**: `process-service-audit`
*   **Runner**: `ubuntu-latest`
*   **Timeout**: `15 minutes` (to accommodate the potentially lengthy script execution)

### 3.1. Job Steps

The `process-service-audit` job will consist of the following steps:

1.  **Checkout Repository**:
    *   Uses `actions/checkout@v4` to checkout the repository code.
    ```yaml
    - name: Checkout repository
      uses: actions/checkout@v4
    ```

2.  **Run Process and Service Audit Script**:
    *   Uses `appleboy/ssh-action@master` to connect to the target server and execute the audit script.
    *   **ID**: `run_audit_script_step` (for referencing output).
    *   **Secrets**: Requires `SERVER_HOST`, `SERVER_USER`, and `SERVER_PASSWORD` secrets to be configured in GitHub repository settings.
    *   **Script**: The multi-line bash script provided by the user will be embedded directly.
    ```yaml
    - name: Run Process and Service Audit Script
      id: run_audit_script_step
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.SERVER_HOST }}
        username: ${{ secrets.SERVER_USER }}
        password: ${{ secrets.SERVER_PASSWORD }}
        script: |
          #!/bin/bash
          # Complete Process and Service Audit Script
          # Shows all running processes, services, and identifies potential security concerns
          # ... (full script content will be placed here) ...
          echo "=========================================="
          echo "Process and service audit completed at $(date)"
          echo "=========================================="
    ```

3.  **Save Script Output**:
    *   This step runs `if: always()` to ensure output is saved even if the script step fails.
    *   Saves the standard output of the script to `process_service_audit_output.txt`.
    *   Saves the standard error of the script to `process_service_audit_error.txt`.
    *   Ensures both files exist using `touch`.
    ```yaml
    - name: Save Script Output
      if: always()
      run: |
        echo "Saving script stdout to process_service_audit_output.txt"
        echo "${{ steps.run_audit_script_step.outputs.stdout }}" > process_service_audit_output.txt
        echo "Saving script stderr to process_service_audit_error.txt"
        echo "${{ steps.run_audit_script_step.outputs.stderr }}" > process_service_audit_error.txt
        touch process_service_audit_output.txt process_service_audit_error.txt
    ```

4.  **Upload Audit Results**:
    *   This step runs `if: always()`.
    *   Uses `actions/upload-artifact@v4` to upload the output files.
    *   **Artifact Name**: `process-service-audit-results-${{ github.run_number }}`
    *   **Files Uploaded**: `process_service_audit_output.txt`, `process_service_audit_error.txt`
    *   **Retention**: `30 days`
    ```yaml
    - name: Upload Audit Results
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: process-service-audit-results-${{ github.run_number }}
        path: |
          process_service_audit_output.txt
          process_service_audit_error.txt
        retention-days: 30
    ```

5.  **Notify on Failure**:
    *   This step runs `if: failure()`.
    *   Prints an error message to the workflow log if the job fails.
    ```yaml
    - name: Notify on failure
      if: failure()
      run: |
        echo "Process and Service Audit scan failed. Check the logs for details."
        echo "::error::Process and Service Audit workflow failed"
    ```

## 4. Next Steps
1.  This plan will be saved to `temp/scan-processes-services-workflow-plan.md`.
2.  Upon your confirmation, I will proceed to create the `.github/workflows/scan-processes-services.yml` file with the content described.
3.  After creating the file, I will suggest switching to "Code" mode for any further actions or review.