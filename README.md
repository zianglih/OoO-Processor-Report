# EECS470_Report

## Group Info

Group 17

|      name      | uniqname |
| :-------------: | :------: |
|    Ziang Li    | ziangli |
|    Zihao Ye    | zihaoye |
|  Yuxiang Chen  | alaricyx |
|   Yuewen Hou   | isaachyw |
| Mingchun Zhuang | mczhuang |
|   Xueqing Wu   |  bradwu  |

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

## Advanced Feature Chart

|            Advanced Features            | Integrated in the final submission | Implemented and working but discarded | Partially Implemented and not integrated |                                                 Comment                                                 |
| :-------------------------------------: | :--------------------------------: | :-----------------------------------: | :--------------------------------------: | :------------------------------------------------------------------------------------------------------: |
|            N way superscalar            |                yes                |                                      |                                          |                                                                                                          |
|                   LSQ                   |                yes                |                                      |                                          |                                                                                                          |
|     LSQ Data Merging and Forwarding     |                                    |                  yes                  |                                          |                        Optional via macro. Turn off for making synthesis faster.                        |
|         Early Branch Resolution         |                yes                |                                      |                                          |                Head retires older instructions, tail rollback one instruction per cycle                |
|                                        |                                    |                                      |                                          |                                                                                                          |
|        Fully Associative D-Cache        |                yes                |                                      |                                          |                                                                                                          |
| Dual Ported D-Cache via Dual Cache Bank |                                    |                                      |                   yes                   | All components but the race condition when two banks compete for mem are implemented. Ease debug burden. |
|        Dcache Writeback Trigger        |                                    |                  yes                  |                                          |          Optional via macro. Not sure if this counts, but it works and it's interesting to do.          |
|             Visual Debugger             |                                    |                                      |                                          |                                               TODO for yzh                                               |

## Implementation Details

### N Way Superscalar
Our staged pipeline supports N-way superscalar in each stages:
+ Fetch: We designed a N-port Icache to support N-way superscalar fetch bandwidth. 
+ Dispatch: We replicated N decoders and utilized N-wide bus to deliver fetched instructions
to later stages of the pipeline. We also designed the map table to handle internal forwarding
of new tags and old tags. The reversation station and reorder buffer implements N-wide interface
for incoming instructions.
+ Issue: We created a dedicated function unit mananger to issue at most N ready instructiosn from reservation
station per cycle.
+ Complete: We designed designated CDB broadcast channel to deliver ready bits and register value
for the issue stage.
+ Retire: The reorder buffer attempts to retire at most N instructions per cycle. Considering
the blocking behavior of the cache, the reorder buffer can at most retire one store per cycle.
The reorder buffer needs to handle complex edge cases during retire such as it cannot retire younger
instruction than the branch currently being rollbacked. It also has replicated prediction for 
next head position considering N-way superscalar feature.
### LSQ

We implement a N-way intergrated load store queue with byte-level forwarding.

#### Initial Approach

We initially implemented a LSQ with following features:

1. issue check for load
   Only when all previous store are successfully issued with address, the incoming load issue request is responsed with issue_approved = 1
2. non-blocking dispatch for load and store
   To minimize waiting time for memory operations, the size of LSQ is the same as ROB size. Therefore all dispatch requests from ROB are guarenteed non-blocking.

#### Problems in Initial Approach

1. Forwarding is rare
   When the size of ROB is 8 and N is 2, the forward success rate across all load request in most test cases are less than 5%
2. byte-level forwarding in an intergrated LSQ is very costly
   For each byte of the $i^{th}$ entry, there are $i - 1$ potential sources. Making the logic complexity grows quickly as N and ROB size grows.
3. In-cycle issue approval response to FU manager is critical path
   In initial approach, the FU manager is required to wait for in-cycle issue approval from load store queue to proceed, which is our critical path.

#### Fix in Final Version For Problems in Initial Approach

1. non-blocking load store request issue
   Accept load request even when there is not issued store before the requested load in load store queue. Now both load and store issue requests are non-blocking and guaranteed to successfully issued once given to load store queue. To accommodate this change, we moved the check to the logic where load store queue handles forwarding or dcache request to avoid potential issues with RAW hazard. After this change to decouple the original dependent logic, we are able to reduce clock period by 20%.
2. 

### Early Branch Resolution

#### Hazard branch detection

Whenever a branch is marked as hazard, i.e, the branch predictor mispredicted, the ROB will mark
that branch as hazard and prompt the reorder buffer into rollback state next cycle. Since ROB supports N-way superscalar,
it will accept at most N branch outcomes and choose the oldest one to rollback if there are multiple
hazards inside one cycle.

#### Cycle-by-cycle rollback

During rollback stage, the reorder buffer will undo the instruction on the tail of the ROB and stall the pipeline from dispatching new instructions.
It will broadcast its new destination register to freelist and map table, index in reservation reservation station and function unit mananger.
The cycle-by-cycle rollback saved the storage overhead for introducing architected map table and freelist yet
enables the reorder buffer to retire instructions older than the hazard branch. One edge case is that
younger branch gets hazard first, while during rollback older branches found new hazards. The reorder buffer
will set up new rollback index to continue the rollback to the oldest branch rather than ping-pong from rollback stage
and normal stage.

#### Complete rollback transition

When the next tail pointer is the hazard branch itself, the reorder buffer will transit to normal
stage where it allows new instructions' dispatch. We also handles the edge case where the branch to rollback
is exactly at index of tail pointer, where there is no instructions to rollback.

### D-Cache

We implemented a write-on-allocate, write-back, fully-associative data cache, that also supports a self clean up trigger.

#### Design Choice

##### Write-on-Allocate, Write-Back

As the main memory is single ported and that excessive d-cache requests may stall i-cache requests, we want to minimize the total request times and improve bandwidth utilization. By a write-on-allocate and write-back policy we only have traffic on a miss and eviction.

##### Fully Associativity

Higher associativity is generally a good thing to have as it increases hit rate effectively dispite higher clock period and power consumption. In our processor, we are aware that the d-cache is not on the critical path so it is reasonable to adopt technology that trade clock period for performance. Furthermore, as our d-cache is single ported and blocking, miss penalty is significant as it completely stall further loads and stores. Overall it's a very good deal to select a full associativity as we nearly sacrifice no loss for a better hit rate and CPI.

##### Semi-Random Eviction

We use ``request.PC[6:3] ^ request.add[7:4]`` as a semi random index for eviction. When there still exsits empty or invalid line, the empty or invalid line is preferred.

##### Aggresive Internal Forwarding

Some forwarding techniques include send memory request as soon as a miss happens within the same cycle, read and write on the new data as soon as it's first ready.

Applying more aggresive internal forwarding trades latency for less cycles. Again as d-cache is not on critical path, by applying aggressive internal forwarding we are sacrificing nothing for less cycles.

#### Internal FSM Design

We define an internal FSM for structuring d-cache behavior:

- DC_READY
- DC_WAITING_WB_RES
- DC_WAITING_RD_ACK
- DC_WAITING_RD_RES
- DC_WAITING_CLEANING_RES

DC_WAITING_CLEANING_RES is just a stage for post-execution cleaning up trigger and is not involved in normal walk-through.

On a hit, the cache stays in DC_READY. On a miss that requires replacing an invalid or clean line, the cache does not need to write back anything so it jump to DC_WAITING_RD_ACK and walk further. On a miss that requires replacing a dirty line, the cache have to write back the dirty result first so it jumps to DC_WAITING_WB_RES.

### Multiplier Design Choice

#### Input Time Sign Extension

We modified from the original pipelined multiplier and still need a way for signed multiplication. The approach we adopted is to do sign-extension at input time, extending all 32-bit input to 64 bits.

#### Motivation

## Interesting Design Ideas

### Notion of "Sub-systems" and Encapsulation

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

### Unit Test

We have written separate module level test benches for the the following:

- branch_predictor
- dcache
- decoder
- fetch
- free_list
- fu_manager
- icache
- inst_buffer
- lsq
- map_table
- mem_issue
- mult
- regfile
- rob
- rs

Note as we have a notion of sub-systems, a lot of modules are actually hierarchical. We wrote test bench and debug such modules from bottom up.

### Integrate Test

The integrate test is done by running provided test programs on the assembled pipeline.

## Debugging Adventure

### Tools

TODO for yzh

### Functionality Debugging

### Synthesis Debugging

### Interesting Bugs

TODO for everyone

- Index Overflow
  - depends on use case ``[$clog2(`N)-1:0]`` may not be enough
- Overlooking Invalid BP Checking
- Overlooking On-Flight Rollback Cancellation
  - Pipelined FUs, RS
- D-Cache Robbing I-Cache's Mem Request
  - Be careful with aggresive d-cache forwarding
- Register Zero Forwarding/Ready Bit/Valid Bit/...
- Load Store Queue Rollback entry with in-flight dcache request

## Analysis

We should have done benchmarking for each of the following but we do not have time as only five people actually work in out team.

For each point, cover reason-effect-optimization

### Sigle Width Rollback as a Bottleneck

#### Behavior

We find that on a mis-speculated branch, it takes many cycles to rollback. Furthermore, as the ILP increases, the CPI does not improve much and by manually tracing for some assembly test cases over half of the cycles are spent on rolling back.

#### Benchmark

TODO for hyw

#### Reason

The reason is superscalar speculative executaion but single-width rollback. Furthermore as the ILP window increases we are more likely to falsely execute. As a result the benefit from ILP is offset by the big penalty from mis-predict and rollback.

#### Implemented Solution

Our idea is straight forward, if the penaly for pursuing higher ILP is big then just trade ILP for something else, majorly a better clock frquency. To do that we performed finer tunning on the pipeline configuration and finalized a set up with 8 ROB entries, 4 RS entries, 2 superscalar width, 1 multiplier.

#### Future Optimization

There are indeed some other possible optimizations on our wishlist that we do not have time for implementing:

- Superscalar Rollback
- Checkpoint
- Improved BP Avoiding Mis Predict

### RS Critical Path

#### Behavior

During synthesis, we found RS is our critical path and that the size of RS has very big impact to clock frequency.

#### Reason

The design of our issue & complete logic is inspired by purely following the behavior of the example from class: check RS entry line by line and examine whether there is an available FU that can take it, untill reaching a superscalar width.

During implementation, the logic is majorly divided into two chunks, FU availability conclusion and RS entry scanning.

The FU availability conclusion does not only involve checking FU status itself, but also requires information about whether CDB can complete its result so that the FU can proceed to accept a new input this cycle or it has to stall and hold the value, which is basically the complete selection logic.

The RS entry scanning part use the info concluded and examine each RS entry. The two parts cannot be done in parallel so their time accumulate on the critical path.

Furthermore, as we have to detect whether a certain instruction need to be rollbacked, such rollback detection and cacellation logic is spread across many differen places making the combinational logic even more complicated.

#### Implemented Solution

#### Future Optimization

There are indeed some other possible optimizations on our wishlist that we do not have time for implementing:

- An issue buffer
  - Allow deeper pipeline, trade latency for throughput and clock
  - Give a unified interface for rollback cancellation detection and trigerring
- Use the provided parallel priority selector instead of huge for loops

## Social Impact

TODO for ziangli

## Credit and Contribution

TODO for everyone

| Name            | Credits                                                       | Comments                                                                                |
| --------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Ziang Li        | Whole FU Manager, Whole Data Cache, MMU, Performance Tweaking | Very high code quality                                                                  |
| Zihao Ye        | Map Table, Dispatch, Regfile, Unit Test Bench                 | Very very very solid debugging                                                          |
| Yuxiang Chen    | ROB, Map Table, Dispatch, Early Branch                        |                                                                                         |
| Yuewen Hou      | Whole Frontend, BP, RS, Free List                             |                                                                                         |
| Mingchun Zhuang | LSQ, I Buffer                                                 | Almost bug-free                                                                         |
| Xueqing Wu      | Some module interface but discarded later                     | No working modules, innocent about any internal design, not cooperative, rarely show up |
