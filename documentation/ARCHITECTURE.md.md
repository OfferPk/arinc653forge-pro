# ARINC 653 Part 1/2 Avionics RTOS Partitioning Architecture

## 1\. Executive Summary & Specification Scope

ARINC 653 defines an Application Executive (APEX) interface between safety-critical avionics application software and the underlying Real-Time Operating System (RTOS) in Integrated Modular Avionics (IMA) architectures.

### Key Architectural Concepts:

1. **Robust Spatial Partitioning**:  
   - Memory Management Units (MMU) isolate address spaces so that application memory faults in lower-criticality partitions (e.g. DAL-D/E) cannot corrupt flight-critical partitions (DAL-A/B).  
2. **Deterministic Temporal Partitioning**:  
   - Static cyclic scheduling repeats within a predefined Major Time Frame (MTF).  
   - Partitions are allocated dedicated, non-preemptible execution windows.  
3. **APEX Inter-Partition Communication (IPC)**:  
   - **Sampling Ports**: Connection-oriented, fixed message size, unqueued (latest value overwrites earlier values), freshness verification.  
   - **Queuing Ports**: Connection-oriented, queued (FIFO or Priority discipline), lossless message delivery.  
4. **Three-Level Health Monitoring (HM)**:  
   - System error handling decomposed into:  
     - Process Level: Handled by application error handler.  
     - Partition Level: Partition restart (`WARM_START` or `COLD_START`).  
     - Module Level: System-wide hardware reset or fallback to secondary core.