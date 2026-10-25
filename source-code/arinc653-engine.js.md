/\*\*  
 \* ARINC 653 Part 1/2 Core Partitioning & APEX IPC Engine  
 \* Implements Major Time Frame (MTF) cyclic scheduling,  
 \* Sampling & Queuing Port buffer queues, process state transitions, and Health Monitoring.  
 \*/

// Partition Operational Modes (ARINC 653 Section 2.3)  
const OPERATING\_MODES \= {  
  IDLE: 'IDLE',  
  COLD\_START: 'COLD\_START',  
  WARM\_START: 'WARM\_START',  
  NORMAL: 'NORMAL'  
};

// Process States (ARINC 653 Section 3.2)  
const PROCESS\_STATES \= {  
  DORMANT: 'DORMANT',  
  READY: 'READY',  
  RUNNING: 'RUNNING',  
  WAITING: 'WAITING'  
};

// Port Discipline  
const PORT\_DISCIPLINES \= {  
  FIFO: 'FIFO',  
  PRIORITY: 'PRIORITY'  
};

class ARINC653Engine {  
  constructor(majorTimeFrameMs \= 100\) {  
    this.majorTimeFrameMs \= majorTimeFrameMs;  
    this.partitions \= \[\];  
    this.samplingPorts \= new Map();  
    this.queuingPorts \= new Map();  
    this.healthMonitoringEvents \= \[\];  
  }

  /\*\*  
   \* Adds a Partition to the Major Time Frame schedule  
   \*/  
  addPartition(id, name, dalLevel, durationMs, periodMs) {  
    const partition \= {  
      id: id,  
      name: name,  
      dal: dalLevel, // 'DAL\_A', 'DAL\_B', etc.  
      durationMs: durationMs,  
      periodMs: periodMs,  
      mode: OPERATING\_MODES.COLD\_START,  
      processes: \[\],  
      allocatedMemoryKb: 2048  
    };  
    this.partitions.push(partition);  
    return partition;  
  }

  /\*\*  
   \* Validates Major Time Frame schedule harmonic constraints  
   \*/  
  validateSchedule() {  
    let totalAllocatedMs \= 0;  
    for (let p of this.partitions) {  
      totalAllocatedMs \+= p.durationMs;  
      if (this.majorTimeFrameMs % p.periodMs \!== 0\) {  
        return {  
          isValid: false,  
          error: \`Partition \${p.name} period (\${p.periodMs}ms) does not divide Major Time Frame (\${this.majorTimeFrameMs}ms)\`  
        };  
      }  
    }

    if (totalAllocatedMs \> this.majorTimeFrameMs) {  
      return {  
        isValid: false,  
        error: \`Total partition duration (\${totalAllocatedMs}ms) exceeds Major Time Frame (\${this.majorTimeFrameMs}ms)\`  
      };  
    }

    const cpuUtilizationPct \= (totalAllocatedMs / this.majorTimeFrameMs) \* 100;  
    return {  
      isValid: true,  
      totalDurationMs: totalAllocatedMs,  
      mtfDurationMs: this.majorTimeFrameMs,  
      cpuUtilizationPct: cpuUtilizationPct.toFixed(1)  
    };  
  }

  /\*\*  
   \* Creates an APEX Sampling Port (latest-value overwrite)  
   \*/  
  createSamplingPort(name, maxMessageSize, refreshPeriodMs) {  
    const port \= {  
      name: name,  
      type: 'SAMPLING',  
      maxSize: maxMessageSize,  
      refreshPeriodMs: refreshPeriodMs,  
      lastValue: null,  
      lastUpdatedTimeMs: 0  
    };  
    this.samplingPorts.set(name, port);  
    return port;  
  }

  /\*\*  
   \* Writes to an APEX Sampling Port  
   \*/  
  writeSamplingMessage(portName, messageBytes, currentTimeMs) {  
    const port \= this.samplingPorts.get(portName);  
    if (\!port) throw new Error(\`Sampling port not found: \${portName}\`);  
    if (messageBytes.length \> port.maxSize) throw new Error(\`Message exceeds max size (\${port.maxSize})\`);

    port.lastValue \= messageBytes;  
    port.lastUpdatedTimeMs \= currentTimeMs;  
    return { status: 'NO\_ERROR', port: portName, size: messageBytes.length };  
  }

  /\*\*  
   \* Reads from an APEX Sampling Port with validity check  
   \*/  
  readSamplingMessage(portName, currentTimeMs) {  
    const port \= this.samplingPorts.get(portName);  
    if (\!port) throw new Error(\`Sampling port not found: \${portName}\`);  
    if (\!port.lastValue) return { status: 'NO\_ACTION', value: null };

    const age \= currentTimeMs \- port.lastUpdatedTimeMs;  
    const isFresh \= age \<= port.refreshPeriodMs;  
    return {  
      status: 'NO\_ERROR',  
      value: port.lastValue,  
      ageMs: age,  
      validity: isFresh ? 'VALID' : 'STALE'  
    };  
  }

  /\*\*  
   \* Creates an APEX Queuing Port (FIFO/Priority buffer)  
   \*/  
  createQueuingPort(name, maxMessageSize, maxNbMessages, discipline \= PORT\_DISCIPLINES.FIFO) {  
    const port \= {  
      name: name,  
      type: 'QUEUING',  
      maxSize: maxMessageSize,  
      maxMessages: maxNbMessages,  
      discipline: discipline,  
      queue: \[\]  
    };  
    this.queuingPorts.set(name, port);  
    return port;  
  }

  /\*\*  
   \* Sends a message to an APEX Queuing Port  
   \*/  
  sendQueuingMessage(portName, messageBytes) {  
    const port \= this.queuingPorts.get(portName);  
    if (\!port) throw new Error(\`Queuing port not found: \${portName}\`);  
    if (messageBytes.length \> port.maxSize) throw new Error(\`Message exceeds max size (\${port.maxSize})\`);  
    if (port.queue.length \>= port.maxMessages) {  
      return { status: 'NOT\_AVAILABLE', error: 'Buffer full' };  
    }

    port.queue.push({ data: messageBytes, timestamp: Date.now() });  
    return { status: 'NO\_ERROR', queueDepth: port.queue.length };  
  }

  /\*\*  
   \* Receives a message from an APEX Queuing Port  
   \*/  
  receiveQueuingMessage(portName) {  
    const port \= this.queuingPorts.get(portName);  
    if (\!port) throw new Error(\`Queuing port not found: \${portName}\`);  
    if (port.queue.length \=== 0\) {  
      return { status: 'NO\_ACTION', message: null };  
    }

    const item \= port.queue.shift();  
    return { status: 'NO\_ERROR', data: item.data, remaining: port.queue.length };  
  }

  /\*\*  
   \* Simulates a Health Monitoring Event and Action Lookup  
   \*/  
  triggerHealthMonitoring(partitionId, errorCode, recoveryAction) {  
    const event \= {  
      timestamp: new Date().toISOString(),  
      partitionId: partitionId,  
      errorCode: errorCode, // 'DEADLINE\_MISSED', 'MEMORY\_VIOLATION', etc.  
      action: recoveryAction // 'IGNORE', 'RESTART', 'COLD\_RESET'  
    };  
    this.healthMonitoringEvents.push(event);  
    return event;  
  }  
}

if (typeof module \!== 'undefined' && module.exports) {  
  module.exports \= {  
    ARINC653Engine,  
    OPERATING\_MODES,  
    PROCESS\_STATES,  
    PORT\_DISCIPLINES  
  };  
}  
