# FEC FLR estimation support in SONiC #

## Table of Content 
- [Revision](#revision)
- [Scope](#scope)
- [Definitions/Abbreviations](#abbreviations)
- [1 Overview](#1-overview)
- [2 Requirements](#2-requirements)
  - [2.1 Functional Requirements](#2.1-functional-requirements)
  - [2.2 CLI Requirements](#2.2-cli-requirements)
- [3 Architecture Design](#3-architecture-design)
- [4 High level design](#4-high-level-design)
  - [4.1 Assumptions](#41-assumptions)
  - [4.2 SAI counters used](#42-sai-counters-used)
  - [4.3 SAI API](#43-sai-api)
  - [4.4 Calculation formulas](#44-calculation-formulas)
- [5 Sample output](#5-sample-output)

### Revision  

  | Rev |     Date    |       Author           | Change Description                |
  |:---:|:-----------:|:----------------------:|-----------------------------------|
  | 0.1 |             | Vinod Kumar Reddy Jammala| Initial version                   |

### Scope  

This document provides the information about the implementation of Port Forward Error Correction (FEC) Frame Loss Ratio (FLR) estimation in SONiC.

### Definitions/Abbreviations 

 | Term    |  Definition / Abbreviation                                            |
 |---------|-----------------------------------------------------------------------|
 | FEC     | Forward Error Correction  |
 | FLR     | Frame Loss Ratio  |
 | Frame   | Size of each FEC block.  
 | Symbol  | Part of the FEC structure which the error detection and correction base on.  
 | RS-FEC  | Reed Solomon Forward Error correction, RS-544 = 5440 total size , RS-528 = 5280 total size  
 | NRZ     | Non Return to Zero encoding  
 | PAM4    | Pulse Amplitude Modulation 4 level encoding  

### 1 Overview
Frame Loss Ratio (FLR) is a key performance metric used to measure the percentage of lost frames relative to the total transmitted frames over a network link. Since receiver device won't have access to sender-side statistics, using Forward Error Correction (FEC) FLR which estimates frame loss based on the number of uncorrected FEC codewords is the best alternative.

## 2 Requirements
### 2.1 Functional Requirements
  This HLD is to   
  - enhance the current "show interfaces counters fec-stats" to include observed and predicted FEC FLR statistics as new columns.
  - Add FEC FLR per interface into Redis DB for telemetry streaming.
  - Calculate the FEC FLR at the same interval as the PORT_STAT poll rate which is 1 sec.

### 2.2 CLI Requirements

The existing "show interfaces counters fec-stats" will be enhanced to include two additional columns for FLR.  
 - OBSERVED_FEC_FLR
 - PREDICTED_FEC_FLR (Phase 2)

## 3 Architecture Design

There are no changes to the current SONiC Architecture.

## 4 High-Level Design

 * SWSS changes:
   + port_rates.lua

      Enhance to collect and compute the FEC FLR on each port at the same port state collection interval. It is currently at 1 second.

     - Access the COUNTER_DB for counters for SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES, SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES, and SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0.
     - Store the computed FEC FLR and previous redis counter values back to the redis DB.

 * Utilities Common changes:

   + portstat.py:

     The portstat command with -f, which representing the cli "show interfaces counters fec-stats" will be enhanced to add two new columns, OBSERVED_FEC_FLR and PREDICTED_FEC_FLR. 


### 4.1 Assumptions

SAI provide access to each interface the following attributes
- SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES 
  - return not support if its not working for an interface
- SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES 
  - return not support if its not working for an interface
- SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0
  - return not support if its not working for an interface


### 4.2 Sai Counters Used

The following redis DB entries will be accessed for the FEC FLR calculations

|Redis DB |Table|Entries|New, RW| Format | Descriptions|   
|--------------|-------------|------------------|--------|----------------|----------------|  
|COUNTER_DB |COUNTERS |SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES |R |number |Total number of uncorrected codewords |
|COUNTER_DB |COUNTERS |SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES |R |number |Total number of corrected codewords |
|COUNTER_DB |COUNTERS |SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0 |R |number |Total number of codewordswith 0 symbol errors |
|COUNTER_DB |COUNTERS_PORT_NAME_MAP | name & oid  |R |name  |Oid to name mapping |  
|COUNTER_DB |RATES |SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES_last |NEW, RW |number |Total number of uncorrected codewords |
|COUNTER_DB |RATES |SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES_last |NEW, RW |number |Total number of corrected codewords |
|COUNTER_DB |RATES |SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0_last |NEW, RW |number |Total number of codewordswith 0 symbol errors |
|COUNTER_DB |RATES |OBSERVED_FEC_FLR |New, RW| floating |calculated observed FEC FLR |  
|COUNTER_DB |RATES |PREDICTED_FEC_FLR |New, RW| floating |calulated predicted FEC FLR |  


### 4.3 SAI API

No change in the SAI API. No new SAI object accessed.


### 4.4 Calculation Formulas

```
Step 1: calcuate observed FLR
    OBSERVED_FEC_FLR = Uncorrectable FEC codewords / (Uncorrectable FEC codewords + Correctable FEC codewords)

    where, Uncorrectable FEC codewords = SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES - SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES_last
	   Correctable FEC codewords = SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES - SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES_last +
				       SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0 - SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0_last

Step 2: the following data will be updated and its latest value stored in the COUNTER_DB, RATES table after each iteraction

    OBSERVED_FEC_FLR, SAI_PORT_STAT_IF_IN_FEC_NOT_CORRECTABLE_FRAMES_last, SAI_PORT_STAT_IF_IN_FEC_CORRECTABLE_FRAMES_last and SAI_PORT_STAT_IF_IN_FEC_CODEWORD_ERRORS_S0_last

```
## 5 Sample Output
```
root@ctd615:/usr/local/lib/python3.11/dist-packages/utilities_common#  portstat -f
      IFACE    STATE    FEC_CORR    FEC_UNCORR    FEC_SYMBOL_ERR    FEC_PRE_BER    FEC_POST_BER    OBSERVED_FEC_FLR
-----------  -------  ----------  ------------  ----------------  -------------  --------------  ------------------
  Ethernet0        U           0             0                 0    1.48e-20       0.00e+00
  Ethernet8        U           0             0                 0    1.98e-19       0.00e+00
 Ethernet16        U           0             0                 0    1.77e-20       0.00e+00
 Ethernet24        U           0             0                 0    4.36e-19       0.00e+00
 Ethernet32        U           0             0                 0    1.93e-19       0.00e+00
 Ethernet40        U           1             0                 1    2.77e-18       0.00e+00
 Ethernet48        U           0             0                 0    8.33e-23       0.00e+00
 Ethernet56        U           0             0                 0    1.48e-55       0.00e+00
 Ethernet64        U           0             0                 0    9.88e-32       0.00e+00
 Ethernet72        U           0             0                 0    4.97e-22       0.00e+00
 Ethernet80        U           0             0                 0    4.10e-19       0.00e+00
 Ethernet88        U           0             0                 0    3.84e-19       0.00e+00
 Ethernet96        U           0             0                 0    4.77e-20       0.00e+00
```

## 6 Unit Test cases

## 7 System Test cases

## 8 Open/Action items - if any

