# ARINC653Forge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your web browser.  
2. In the "IMA Partition Matrix Config" panel, adjust the Major Time Frame (MTF) duration in milliseconds.  
3. Review or customize the Partition Schedule Slots (e.g. Flight Control, Navigation, Telemetry).  
4. Select the APEX Port Type (`Sampling Port` or `Queuing Port`).  
5. Click **Validate MTF Schedule** to verify harmonic period constraints and compute total CPU utilization.  
6. Click **Simulate HM Fault** to trigger a mock partition deadline miss or memory violation and inspect the Health Monitoring recovery action.  
7. Click **Run Normative Audit** to evaluate all 10 ARINC 653 Part 1/2 compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression tests and compliance verification:

python3 arinc653-cli.py \--audit \--mtf 100