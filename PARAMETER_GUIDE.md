# Dataset Parameters Guide for CPU Temperature-Based Task Scheduling

## Overview

This document explains the parameters in `merged_dataset_detailed.csv` and how they relate to building a temperature-aware CPU task scheduling algorithm.

---

## 1. Process Parameters

### 1.1 Process Identification
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `Timestamp_Round` | Timestamp (5-sec intervals) | Time-series correlation with temperature |
| `ProcessId` | Unique process identifier | Track specific process behavior over time |
| `ProcessName` | Name of the executable | Identify resource-intensive applications |
| `ProcessType` | Category: System/Application/UWP/UserApp | Different types have different thermal profiles |

### 1.2 CPU Usage Metrics
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `CPU_Percent` | Current CPU usage (%) | **PRIMARY** - Directly correlates with temperature (+0.71 to +0.78) |
| `TotalProcessorTime_Sec` | Cumulative CPU time used | Identifies long-running CPU-intensive processes |
| `UserProcessorTime_Sec` | Time in user mode | User-mode processing generates more heat |
| `PrivilegedProcessorTime_Sec` | Time in kernel mode | Kernel operations are often I/O bound |

### 1.3 Memory Metrics
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `WorkingSet_MB` | Physical RAM used | Memory operations affect power consumption |
| `PrivateMemory_MB` | Private memory allocation | High memory = more cache activity |
| `PeakWorkingSet_MB` | Maximum RAM used | Indicates potential memory spikes |
| `VirtualMemory_MB` | Virtual address space | Large virtual memory = potential swapping |

### 1.4 Thread and Handle Metrics
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `ThreadCount` | Number of active threads | Multi-threaded = load distributed across cores |
| `HandleCount` | OS resource handles | High handles = I/O intensive process |

### 1.5 Priority and Affinity
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `BasePriority` | Numeric priority (0-31) | Higher priority = more CPU time allocation |
| `PriorityClass` | Priority class name | Normal/High/RealTime affect scheduling |
| `AffinityMask` | Bitmask of allowed cores | Determines which cores can run the process |
| `AllowedCores` | Human-readable core list | Direct core assignment information |

### 1.6 I/O Metrics
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `ReadOperationCount` | Number of read operations | I/O-bound processes generate less heat |
| `WriteOperationCount` | Number of write operations | Disk I/O doesn't heat CPU much |
| `ReadTransferCount_KB` | Data read volume | Large reads = memory/disk activity |
| `WriteTransferCount_KB` | Data written volume | Large writes = disk activity |

---

## 2. CPU Temperature Parameters

### 2.1 Overall Temperature
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `CPU (Tctl/Tdie) [°C]` | Main CPU temperature sensor | **PRIMARY** - Overall thermal state |
| `Core Temperatures (avg) [°C]` | Average of all core temps | System-wide thermal indicator |

### 2.2 Per-Core Temperature
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `Core0 [°C]` | Core 0 temperature | **CRITICAL** - Per-core scheduling decision |
| `Core1 [°C]` | Core 1 temperature | Identifies hottest cores to avoid |
| `Core2 [°C]` | Core 2 temperature | Enables load migration to cooler cores |
| `Core3 [°C]` | Core 3 temperature | Temperature-aware core selection |

---

## 3. CPU Usage Parameters

### 3.1 Per-Thread Usage (T0 = Thread 0, T1 = Thread 1 for SMT/Hyperthreading)
| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `Core Usage (avg) [%]` | Average usage across cores | Overall CPU load indicator |
| `Total CPU Usage [%]` | Total system CPU usage | System-wide thermal predictor |
| `Core 0 T0 Usage [%]` | Core 0, Thread 0 usage | Per-thread load distribution |
| `Core 0 T1 Usage [%]` | Core 0, Thread 1 usage | SMT thread utilization |
| `Core 1-3 T0/T1 Usage` | Other cores/threads | Complete per-thread visibility |

---

## 4. Power Parameters

| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `CPU Package Power [W]` | Total CPU power draw | Power = Heat generation |
| `Core Powers (avg) [W]` | Average per-core power | Power distribution insight |
| `Core 0-3 Power [W]` | Individual core power | Correlates with per-core temperature |

---

## 5. Clock Speed Parameters

| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `Core Clocks (avg) [MHz]` | Average clock speed | Higher clocks = more heat |
| `Core 0-3 Clock [MHz]` | Per-core clock speeds | Dynamic frequency affects temperature |

---

## 6. Throttling Parameter

| Parameter | Description | Scheduling Relevance |
|-----------|-------------|---------------------|
| `Thermal Throttling (HTC) [Yes/No]` | CPU is thermally throttling | **CRITICAL** - Indicates thermal emergency |

---

## 7. Parameter Relationships for Scheduling

### 7.1 Primary Correlations (Strong: r > 0.7)

```
CPU_Percent ────────────┬──────────> Core Temperature (+0.71 to +0.78)
                        │
Total CPU Usage ────────┤
                        │
Max CPU (per process) ──┘

CPU Package Power ─────────────────> Core Temperature (+0.65 to +0.75)

Core Clock Speed ──────────────────> Core Temperature (+0.55 to +0.65)
```

### 7.2 Secondary Correlations (Moderate: 0.4 < r < 0.7)

```
Thread Count ───────────────────────> CPU Distribution
                                      (More threads = spread across cores)

Memory Working Set ─────────────────> Temperature (+0.3)
                                      (Memory ops generate some heat)
```

### 7.3 Key Relationship: CPU Usage → Temperature Lag

There is a **time lag** between high CPU usage and temperature increase:
- High CPU usage at time T
- Temperature peaks at T+5 to T+15 seconds
- This lag enables **predictive scheduling**

---

## 8. Building a Temperature-Aware Scheduling Algorithm

### 8.1 Decision Parameters (Ordered by Importance)

1. **Per-Core Temperature** (`Core0-3 [°C]`) - Choose coolest core
2. **Process CPU Usage** (`CPU_Percent`) - Predict heat generation
3. **Per-Core Usage** (`Core X TX Usage [%]`) - Find least loaded core
4. **Process Priority** (`BasePriority`) - Respect priority levels
5. **Thread Count** - Distribute multi-threaded workloads
6. **Affinity Mask** - Respect process constraints

### 8.2 Algorithm Logic Flow

```
1. READ current per-core temperatures
2. IDENTIFY cores below threshold (e.g., < 75°C)
3. FOR each new/migrating process:
   a. GET process CPU_Percent (predicted heat)
   b. GET process AffinityMask (allowed cores)
   c. FILTER cores: allowed AND below threshold
   d. SELECT core with lowest temperature
   e. ASSIGN process to selected core
4. IF all cores above threshold:
   a. ENABLE throttling mode
   b. REDUCE clock speeds
   c. DELAY non-critical processes
```

### 8.3 Temperature Thresholds (Suggested)

| Threshold | Temperature | Action |
|-----------|-------------|--------|
| Normal | < 65°C | Normal scheduling |
| Warm | 65-75°C | Prefer cooler cores |
| Hot | 75-85°C | Migrate tasks away |
| Critical | > 85°C | Throttle/pause non-essential |

### 8.4 Process Classification for Scheduling

| ProcessType | Thermal Impact | Scheduling Priority |
|-------------|----------------|---------------------|
| System | Low-Medium | High (system stability) |
| Application | High | Normal |
| UserApp | Medium-High | Normal |
| UWP | Low-Medium | Background |

---

## 9. Summary: Key Parameters for Your Algorithm

### Must-Have Parameters:
1. `Core0-3 [°C]` - Per-core temperature
2. `CPU_Percent` - Process CPU usage
3. `AffinityMask` / `AllowedCores` - Core constraints
4. `BasePriority` - Process importance
5. `ProcessType` - Process classification

### Nice-to-Have Parameters:
6. `ThreadCount` - Load distribution hints
7. `Core X TX Usage [%]` - Per-thread utilization
8. `CPU Package Power [W]` - Power-based decisions
9. `Thermal Throttling (HTC)` - Emergency detection
10. `RunningTime_Min` - Long-running process identification
