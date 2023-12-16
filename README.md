# EECS470_Report

## Group Info

Group 17

| name            | uniqname |
|:---------------:|:--------:|
| Ziang Li        | ziangli  |
| Zihao Ye        | zihaoye  |
| Yuxiang Chen    | alaricyx |
| Yuewen Hou      | isaachyw |
| Mingchun Zhuang | mczhuang |
| Xueqing Wu      | bradwu   |

## Base Design

- R10K style dynamic scheduling and register renaming
  - ROB
  - RS
  - Map table
  - Free list
- "Frontend"
  - Instruction Buffer
  - Branch Predictor
  - Branch Target Buffer
  - Instruction Cache
- "Execution sub-system"
  - Integer ALU
  - Pipelined multiplier
  - CDB
  - Issue logic
- "Memory sub-system"
  - LSQ
  - D cache

## Advanced Feature Chart

| Advanced Features                       | Integrated in the final submission | Implemented and working but discarded | Partially Implemented and not integrated | Comment                                                                                                  |
|:---------------------------------------:|:----------------------------------:|:-------------------------------------:|:----------------------------------------:|:--------------------------------------------------------------------------------------------------------:|
| N way superscalar                       | yes                                |                                       |                                          |                                                                                                          |
| LSQ                                     | yes                                |                                       |                                          |                                                                                                          |
| LSQ Data Merging and Forwarding         |                                    | yes                                   |                                          | Optional via macro. Turn off for making synthesis faster.                                                |
| Early Branch Resolution                 | yes                                |                                       |                                          | Head retires older instructions, tail rollback one instruction per cycle                                 |
| Fully Associative D-Cache               | yes                                |                                       |                                          |                                                                                                          |
| Dual Ported D-Cache via Dual Cache Bank |                                    |                                       | yes                                      | All components but the race condition when two banks compete for mem are implemented. Ease debug burden. |
| Dcache Writeback Trigger                |                                    | yes                                   |                                          | Optional via macro. Not sure if this counts, but it works and it's interesting to do.                    |
| Visual Debugger                         |                                    |                                       |                                          | Need cooperative use of P3's wb file, could be included inside the `run.sh` file.                                                                                            |

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

We implemented a N-way intergrated load store queue with byte-level forwarding. The load store queue is the single united interface for our memory system, which encapsulated DCache.

#### Initial Approach

We initially implemented a LSQ with following features:

1. issue check for load
   Only when all previous stores in queue are successfully issued with address, the incoming load issue request is responsed with issue_approved = 1
2. non-blocking dispatch for load and store
   To minimize waiting time for memory operations, the size of LSQ is the same as ROB size. Therefore all dispatch requests from ROB are guarenteed non-blocking.

#### Problems in Initial Approach

1. Forwarding is rare
   When the size of ROB is 8 and N is 2, the forward success rate across all load request in most test cases are less than 5%
2. Byte-level forwarding in an intergrated LSQ is very costly
   For each byte of the $i^{th}$ entry, there are $i - 1$ potential sources. Taking into account that each entry contains at most 4 bytes, there are $4 \cdot (i - 1)$ potential sources in total. Making the logic complexity grows quickly as load store queue size grows.
3. In-cycle issue approval response to FU manager is critical path
   In initial approach, the FU manager is required to wait for in-cycle issue approval from load store queue to proceed, which is our critical path.

#### Final Version with Fix For Problems in Initial Approach

1. non-blocking load store request issue
   In the final version, load store queue will accept load request even when there is store without issued address before the requested load in the load store queue. Now both load and store issue requests are non-blocking and guaranteed to successfully issued once given to load store queue. To accommodate this change, we moved the check to the logic where load store queue handles forwarding or dcache request to avoid potential issues with RAW hazard. After this change to decouple the original dependent logic, we are able to reduce clock period by 20%.
2. Restricted forward range in load store queue
   We observed that among the rare forwarding hit cases, most forwarding sources are at the very beginning of the queue head. To reduce the cost of queue wide forwarding, we add a configurable parameter to restrict the forward source range such that we could make trade-off between the logic complexity and overall performance.  To achieve dynamic forward range while maintaining correctness, we only allowed first $range\_forward$ entries as forward sources and issue memory request if the entries failed to forward and are within $range\_forward $ distance from the queue head. We ran the profiling under the final configuration parameter (ROB size: 8, N: 2). After profiling the outcome between different configuarations of forward range, we found that even with a very restricted forward range ($range\_forward  = n$), the hit rate change in most cases is within 10% and the impact on CPI is within 2%, which is completely acceptable considering that this change reduce 70% of the forward logic, where the forwarding logic takes up more than 30% of the total mutexes in the synthesis process.
3. Disable forwarding
   Since we implemented byte-level forwarding based on a unified load store queue, even with a great improvements in #2 that greatly reduced its logic complexity, it's still the most time consuming process in the system. In the final design if we enable the forwarding, systhesis time will be long due to its complicated logic. Considering the rare hit rate of forwarding and poor CPI improvements in many test cases, we disable forwarding in the final version.

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

### Multiplier

We modified from the original pipelined multiplier and still need a way for signed multiplication. The approach we adopted is to do sign-extension at input time, extending all 32-bit input to 64 bits.

In this approach, it is closer to real-world multiplier that perform binary multiplication and is more technically elegant.

This approach may not be the optimal one as by sign extension a lot of compute throughput is on wasted bits.

### RS

In our implementation, RS module is not in charge of select entries to output. Instead, it exposes all its entries to fu_manager, let fu_manager select what to issue and takes an feedback from fu_manager as an input.

The motivation and benefit is this creates a clear segragation between the register renaming part and the execution part in the pipeline, which makes future encapsulation much easier.

## High Level Design Guidlines

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

We initially used the `$display` inside our module to print out the suspect values. However the sigal in combinational logic is not stable,and the displaying scontent is also too huge for us to easily locate the buggy time and place,  making the debugging process time-comsuming and sometimes went wrong by wasting time on the by-product of the real bug source.

We then choose `verdi` as the primary debugging tool, it starts well by solving several "hard-to-detect" bugs which we couldn't find by displaying singals out. However, when instructions nunber grows, finding the exact time where the bug exposed becomes a real problem. Since `verdi` does not support a specific value search in one signal through time, we have to locate the approximate timeline of the bug, then using `verdi` to make inspection. More than that, when the buggy instruction has several layers of dependency, the backtracking becomes a real problem: one might easily get lost trying to backtrack the source resgiter value throughout the whole program.

To solve this problem, we wrote a visual-debugger. It uses the `display` message from retire stage and merge them together since the retirement is in-order. Everyline of this input will contain `time`, `pc`, `dest_reg_idx` and `writeback value`, which corresponds to the `.wb` file generated from project 3. In this way we obtain the instruction flow and can easily use `diff` to get the first wrong line, which drastically improve our debugging speed. The source file of this program is stored in `unique.py` and `script.py`.

### Functionality Debugging
Now our debugging flow is highly condense and efficient. We recommend firstly use our visual debugger to generate the instructions flow and do the `diff` to find the timeline.

We then choose to use `verdi` or the displaying content inside the `result.log` file. Since the the buggy instruction is found and the backtracking could be easily done, we can choose the rest debugging strategy according to the speculative type of the bug:
- memory bug: `verdi` usually works better.
- frontend bug: `display` might saves time.


The `display` content should also be considered, here is what we includes:
- `ROB CONTENT`
- `RS CONTENT`
- `RETIRED INST`
- `FETCH RESULT`
- `MEM BUS`
- ...

### Synthesis Debugging

TODO for whoever understands

We also find the ```-xprop=tmerge``` flag helpful.

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

### Sigle Width Rollback as a Bottleneck

#### Behavior

We find that on a mis-speculated branch, it takes many cycles to rollback. Furthermore, as the ILP increases, the CPI does not improve much and by manually tracing for some assembly test cases over half of the cycles are spent on rolling back.

#### Benchmark

Sweep result on `graph.c`

| CPI  | ROB Size | RS Size |
| ---- | -------- | ------- |
| 5.48 | 64       | 64      |
| 5.46 | 32       | 32      |
| 5.29 | 16       | 16      |
| 5.01 | 8        | 8       |

Sweep result on `mergesort.c`

| CPI  | ROB Size | RS Size |
| ---- | -------- | ------- |
| 5.61 | 64       | 64      |
| 5.14 | 32       | 32      |
| 4.29 | 16       | 16      |
| 4.01 | 8        | 8       |

As shown above, inceasing ROB size and RS size which allows higher ILP leads to an even worse CPI, which is in align with our analysis.

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

#### Future Optimization

There are indeed some other possible optimizations on our wishlist that we do not have time for implementing:

- An issue buffer
  - Allow deeper pipeline, trade latency for throughput and clock
  - Give a unified interface for rollback cancellation detection and trigerring
- Use the provided parallel priority selector instead of huge for loops
- Split Unified Load Store Queue to further reduce its size and thus the forward logic complexity

## Social Impact

The most special feature of our processor design is that we keep the notion of "sub-systems" and encapsulation in the first place, so the highest level pipeline is clean and organized and the sub-system modules are relatively indepent. The chip can be built in a "chipset"-like manner, which create several benefits to the commertialization process and the society.

- Reduce waste in wafer in production. The sub-systems can be separately producted so we do not need a huge chip area in production time. This will reduce waste on edge and reduce defective rate. Furthermore, this will lead to lower-costs chips for human that use fewer resources in the production process.
- The core register renaming logic can be selected as a base motherboard, while the execution and memory sub-systems could be assembled with customized configurations as add ons. This will create huge flexibility for different compute workload. For example an embedded device may configure a minimal execution sub-system to reduce power consumption. A data center could customize different configuration for different services, such as one with more multiplier for HPC workload and one with a more advanced memory sub-system for IO intensive service.

## Credit and Contribution

TODO for everyone

| Name            | Credits                                                       | Comments                                                                                |
| --------------- | ------------------------------------------------------------- | --------------------------------------------------------------------------------------- |
| Ziang Li        | Whole FU Manager, Whole Data Cache, MMU, Performance Tweaking | Very high code quality                                                                  |
| Zihao Ye        | Pipeline, Dispatch, Regfile, visual debugger, Unit Test Bench, memory debugging                 | Very solid debugging, speeding up the whole project progress, main solver to Xueqing's broken modules                                                          |
| Yuxiang Chen    | ROB, Map Table, Dispatch, Early Branch, frontend debugging                        | Main solver to Xueqing's broken modules, spend most time and work towards it                                                                                         |
| Yuewen Hou      | Whole Frontend, BP, RS, Free List                             |                                                                                         |
| Mingchun Zhuang | Load Store Queue, Instruction Buffer                                                 | Thorough unit tests covering extreme cases. Almost bug-free after intergated into pipelines                                                                        |
| Xueqing Wu      | Some module interface but discarded later, decoder and maptable but can't even compile                     | No working modules, innocent about any internal design, not cooperative, rarely show up |

## Corectness Summary

| test | Simulation | Synthesis | CPI |
| :---: | :---: | :---: | :---: |
| mult_no_lsq | passed | passed |
| rv32 | passed | passed |
| alexnet.c |||
