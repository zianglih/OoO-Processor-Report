# EECS470_Report

## Group Info
Group 17

| name | uniqname |
| :---: | :---: |
| Ziang Li | ziangli |
| Zihao Ye | zihaoye |
| Yuxiang Chen | alaricyx |
| Yuewen Hou | isaachyw |
| Mingchun Zhuang | mczhuang |
| Xueqing Wu | bradwu |

## Base Design
- R10K style dynamic scheduling and register renaming
  - ROB
  - RS
  - Map table
  - Free list
- "Forntend"
  - TODO for hyw
- "Execution sub-system"
  - Integer ALU
  - Pipelined multiplier
  - CDB
  - Issue logic
- "Memory sub-system"
  - LSQ
  - D cache

## Advanced Feature

### The Chart

| Advanced Features | Integrated in the final submission | Implemented and working but discarded | Partially Implemented and not integrated | Comment |
| :---: | :---: | :---: | :---: | :---: |
|N way superscalar|yes|||
|LSQ|yes|||
|LSQ Data Merging and Forwarding||yes||Optional via macro. Turn off for making synthesis faster.|
|Early Branch Resolution|yes|||
|Fully Associative D-Cache|yes|||
|Dual Ported D-Cache via Dual Cache Bank|||yes| All components but the race condition when two banks compete for mem are implemented. Ease debug burden.|
|Dcache Writeback Trigger||yes|| Optional via macro. Not sure if this counts, but it works and it's interesting to do. |

### Details
#### N Way Superscalar
#### LSQ
TODO for MC
#### Early Branch Resolution
TODO for cyx
#### D-Cache
TODO for ziangli

## Interesting Design Ideas

### "Sub-systems"
#### Motivation
Modern processors, especially SoCs or chipset systems usually have the major functionalities encapsulated into standalone sub-modules. This creates many benefits, such as easier integration with other company's propriertary IPs, easier early-stage design, allowing separate verfication etc.
#### Our Implementation
In our design, we consider the idea of sub-system and encapsulation in the first place. We have three major sub-systems, exectution sub-system called "fu-manager" which manages issu, execution, and complete, data memory sub-system which is essentially LSQ wrapping around write-back d-cache, and a frontend.

Structures supporting register renaming are not encapsulated as we want to preserve the characteristics of the R10K pipeline taught in class.
#### Results
Benefits:
- Top level pipeline is more organized.
- Sub-systems can each be fully tested before assembled to pipeline.
- Smoother work distribution across team members.

Drawbacks:
- When different sub-systems interact with each other, severe errors may occur due to unclear assumptions.
- Each sub-module is more complicated internally. When wanting to add support for a new feature on a developed sub-system, it's usaually hard to come up with a bug-free and elegant approach. We usually find ourselves making compromises just to make things work.

### Parameterization
#### Motivation
From a very early stage in the project, we concluded that if we could parameterize everything, we would have much larger flexibility for performance tweaking in the end.
#### Challenges
Writing HDL code consists of for loops, counters, and feedbacks (though should be avoided) while still keeping its synthesziability is very challenging.
#### Results
List of configurable parameter:
- ROB_SZ
- RS_SZ
- N
- CDB_W
- NUM_ALU
- NUM_MULT
- MULT_STAGES
- REG_PORT
- BHT_SZ
- BTB_SZ
- LSQ forward range

Specifically, our implementation supports arbitrary complete number that could be different from N, which does not add much value in this project but provides high flexibility for tweaking the pipeline.
## Testing Methodology
### "Progressive" Unit Test
### Integrate Test
## Debugging Adventure
### Tools
### Functionality Debugging
### Synthesis Debugging
### Interesting Bugs
- Index Overflow
- Overseen Invalid BP Checking
- Overseen On-Flight Rollback Cancellation
- D-Cache Robbing I-Cache's Mem Request
- Register Zero Forwarding/Ready Bit/Valid Bit/...

## Analysis
We should have done benchmarking for each of the following but we do not have time as only five people actually work in out team.

For each point, cover reason-effect-optimization
### ILP Roofline
TODO for ziangli
### RS Critical Path
TODO for ziangli
### Data Cache Design Choice and Tradeoff
TODO for ziangli
### Multiplier Design Choice
TODO for ziangli

## Social Impact
TODO for ziangli
## Credit and Contribution

