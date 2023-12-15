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
TODO for ziangli
### Parameterization
TODO for ziangli
## Testing Methodology
### "Progressive" Unit Test
### Integrate Test
## Debugging Adventure
### Tools
### Functionality Debugging
### Synthesis Debugging

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

