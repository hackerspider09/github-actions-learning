# Chapter 11: Advanced Bash Operations

The default expression syntax within GitHub Actions lacks native capabilities for certain complex operations. Advanced operations within `run` steps require standard shell capabilities. 

## 1. Modifying the System Path (`$GITHUB_PATH`)

Custom executables downloaded during a workflow run require path modification to be accessible globally across subsequent steps. Directories are appended to the system PATH by writing to the `$GITHUB_PATH` environment file.

```yaml
steps:
  - name: Install Executable
    run: |
      mkdir -p $HOME/custom-tools
      wget https://example.com/binary-tool -O $HOME/custom-tools/binary-tool
      chmod +x $HOME/custom-tools/binary-tool
      
      echo "$HOME/custom-tools" >> $GITHUB_PATH

  - name: Execute Tool
    run: binary-tool --version
```

## 2. Generating Markdown Summaries (`$GITHUB_STEP_SUMMARY`)

Custom markdown data can be rendered on the GitHub Actions workflow run summary page by writing content to the `$GITHUB_STEP_SUMMARY` environment file. This is utilized to aggregate test results, coverage statistics, or deployment statuses natively.

```yaml
steps:
  - name: Generate Report
    run: |
      echo "## Deployment Status" >> $GITHUB_STEP_SUMMARY
      echo "Target environment updated successfully." >> $GITHUB_STEP_SUMMARY
      echo "" >> $GITHUB_STEP_SUMMARY
      echo "| Component | Status |" >> $GITHUB_STEP_SUMMARY
      echo "| --- | --- |" >> $GITHUB_STEP_SUMMARY
      echo "| API Gateway | Online |" >> $GITHUB_STEP_SUMMARY
      echo "| Database | Online |" >> $GITHUB_STEP_SUMMARY
```
*(Reference: `10-test_summary.yaml`)*

## 3. SSH Debugging

During persistent failure analysis, a workflow execution can be paused to facilitate interactive debugging via SSH. This is accomplished using community actions such as `mxschmitt/action-tmate`.

```yaml
steps:
  - name: Initialize Debug Session
    uses: mxschmitt/action-tmate@v3
```
*Security Note: Interactive debugging actions suspend execution and should not be merged into production workflows, as they consume execution minutes and expose runner access.*
