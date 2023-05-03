---
layout: default
title: Computer Organization
subscription: Chapter 4
---

Text can be **bold**, _italic_, or ~~strikethrough~~.

# Data Hazards
## Forwarding
Let’s look at a sequence with dependences:
```
sub $2, $1, $3
and $12,$2, $5  # 1st operand($2) depends on sub
or  $13,$6, $2  # 2nd operand($2) depends on sub
add $14,$2, $2  # both operands depend on sub
sw  $15,100($2) # base ($2) depends on sub
```
If we don’t do something, the value of $2 will effect the later instruction.  
As shown below, $2 is written in clock cycle 5, so the proper value is unavailable before clock cycle 5.  
(There’s a potential hazard, if the register write in the first half of the clock cycle and read in the second, this hazard will be solved.)

![4.52](https://i.imgur.com/zejkHki.png)

As mentioned, _forwarding_ can solve this problem.  
We needn’t necessarily wait untill the WB stage, since the result is available at the end of the EX stage(namely, clock cycle 3).  
Notice that we have pipeline register to perserve the result, so we can simply forward the content in pipeline regitster:

![4.53](https://i.imgur.com/IjEdbE9.png)

However, some instructions don’t write registers, e.g. (```sw```).  
We can simply see whether the RegWrite signal in WB control field of the pipeline registers is asserted or not to decide when to forward.  
But there’s still a bug here. Consider a sequence of hacking code:
```
add $0, $1, $2 
add $3, $0, $0 # shall it forwarding?
```
If we forwarding the result of $1+$2, the content in $3 isn’t correct.  
To solve this special problem, we need to modify forwarding condition: forwarding when RegWrite asserted and **EX/MEM.RegisterRd isn’t 0 and MEM/WB.RegisterRd isn’t 0**.

For now, we will assume the only instruction we need to forward are the four R-type instructions: (```add```,```sub```,```and```,```or```).  
The figure below shows the partial pipelined datapath with forwarding.
![4.54](https://i.imgur.com/fQFpbRN.png)

The forwarding unit will function in the EX stage, because the ALU forwarding multiplexors are in this stage.

![4.55](https://i.imgur.com/uk38lxA.png)

Now we write the conditions for detecting hazards and the forwarding control signal based on above figure:

_EX hazard_:
```c
if (\
   EX/MEM.RegWrite and \
   (EX/MEM.RegisterRd != 0) and \
   (EX/MEM.RegisterRd == ID/EX.RegisterRs)\
   ) 
    ForwardA = 0b10;  
if (\
   EX/MEM.RegWrite and \
   (EX/MEM.RegisterRd != 0) and \
   (EX/MEM.RegisterRd == ID/EX.RegisterRt)\
   )
    ForwardB = 0b10;
```
This case forwards the result from the previous instrcution to either input of ALU.

_MEM hazard_:
```c
if (\
   MEM/WB.RegWrite and \
   (MEM/WB.RegisterRd != 0) and \
   (MEM/WB.RegisterRd == ID/EX.RegisterRs)\
   )
    ForwardA = 0b01;
if (\
   MEM/WB.RegWrite and \
   (MEM/WB.RegisterRd != 0) and \
   (MEM/WB.RegisterRd == ID/EX.RegisterRt)\
   )
    ForwardB = 0b01;
```
This case forwads the result from data memory or an earlier ALU result.

However, consider:
```
add $1, $1, $2
add $1, $1, $3
add $1, $1, $4
```
The condition above will have a conflict, ForwardA shall be 0b10(EX hazard) but also 0b01(MEM hazard).  
Obviously, the forwarding from MEM/WB is obsolete, so what we need is the forwarding from EX/MEM (if exists). We modify the MEM hazard condition:
```c
if (\
   MEM/WB.RegWrite and \
   (MEM/WB.RegisterRd != 0) and \
   
   !(\
   EX/MEM.RegWrite and \
   (EX/MEM.RegisterRd != 0) and \
   (EX/MEM.RegisterRd == ID/EX.RegisterRs)\
   ) and \
   
   (MEM/WB.RegisterRd == ID/EX.RegisterRs)\
   )
    ForwardA = 0b01;
    
if (\
   MEM/WB.RegWrite and \
   (MEM/WB.RegisterRd != 0) and \
   
   !(\
   EX/MEM.RegWrite and \
   (EX/MEM.RegisterRd != 0) and \
   (EX/MEM.RegisterRd == ID/EX.RegisterRt)\
   ) and \
   
   (MEM/WB.RegisterRd == ID/EX.RegisterRt)\
   )
    ForwardB = 0b01;
```
Figure below shows the hardware necessary to support forwarding for operations that use results during the EX stage. 
![4.56](https://i.imgur.com/8PtYa15.png)
> Forwarding can also help with hazards when store instructions are dependent on other instructions. and it’s actually easy to implement.
> I-type instruction, e.g. (```addi```), (```muli```), etc…, can be solve by adding one more multiplexer. As shown below.  
![4.57](https://i.imgur.com/4dMOsBZ.png)

## Stalling
As we mentioned, there’s one case where forwarding can’t handle. it happens when an instruction tries to read a register following a load instruction that writes the same register. as shown below.
![4.58](https://i.imgur.com/n4dUDKd.png)
To solve the problem, we need to “stall” a pipeline, which means we do nothing and delay the following instruction by 1 clock.

Hence, we add one more unit, called hazard detection unit. it operates during ID stage so that it can insert stall betweem the load and its ues.
the condition of hazard detection unit:
```c
if (\
   ID/EX.MemRead and \
   (\
     (ID/EX.RegisterRt == IF/ID.RegisterRs) or \
     (ID/EX.RegisterRt == IF/ID.RegisterRt)\
   )\
   )
    stall the pipeline
```
After this 1-cycle stall, the forwarding logic will take care the rest part.

Let’s talk about how to implement “stall”, i.e. do nothing.  
If we stall a pipeline, we expect that every thing won’t change, including PC. (so that we can repeat the instruction)  
How can we achieve that? Recall the control line involve in the changing of state:  
1.  Branch
2.  MemWrite
3.  RegWrite

If these control lines are deasserted during ID stage, none of state will be changed. To implement easier, we simply deasserted all the nine control lines.  
But PC and IF/ID? they will both be written just after the IF stage, while the hazard detection unit work on ID stage.  
Well, we need to connect PC and IF/ID with a write signal (unlike before, we assume they will be updated every clock cycle.) which is usually asserted but when the hazard occurs, it deasserted.  
We call the instruction that doesn’t change any state a _nop_.

The figure shows what happens when stalling a pipeline:
![4.59](https://i.imgur.com/LFrSwm5.png)
Finally, put the pieces of idea into implementation:
![4.60](https://i.imgur.com/416Zt0q.png)
> The control line, Branch, isn’t matter asserted or not after we add a PCWrite signal.
# Control Hazards
As mentioned, there’re also pipeline hazards involving branches.  
As shown below:
![4.61](https://i.imgur.com/X1HxnS7.png)
The decision about whether to branch doesn’t occur till the MEM pipeline stage. This delay in determining the proper instruction to fetch is called _control hazard_ or _branch hazard_.

Branch is a major problem when the pipeline being more deeper. If our prediction is wrong, all the pipeline will waste. Eventually, it effect the performance of pipeline significantly. So branch prediction is an important topic we need to study.  
(To take advantage of long pipeline, some program in super computers will deliberately decrease the amount of branch.)
## Assume branch not taken
If the branch is not taken, the instruction will continue; if it is, the instruction that are being fetched and decoded must be discarded. and continue execution at the branch target.
to discard (flush) instructions, we merely change the original control value to 0s, much as we did to stall a pipeline.  
The difference is, in stalling, we need to preserve the info in IF/ID and PC, but in flushing, we need to change the three instructions in the IF, ID, EXE stages.

## reducing the delay of branches
Before enter the section of branch prediction, we first introduce a simple technique to reduce the delay of branch.  
This technique is moving the branch execution earlier in the pipeline, then fewer instructions need to be flushed.  
Moving the branch decision up requires two actions to occur earlier:

1. computing the branch target address  
for the target address, since we already have the PC value and the immediate field in the IF/ID pipeline register, so we just move the branch adder from the EXE stage to the ID stage.
2. evaluating the branch decision  
For branch equal, we would compare the two registers read during the ID stage to see if they are equal.  
Equality can be tested by first XOR their respective bits and then OR all the results.
Moving the branch test to the ID stage implies additional forwarding and hazard detection hardware.  
e.g. one of, or both, the operands in equality comparison is needed from fowarding.  
However, it is possible that a data hazard can occur and a stall will be needed.  

## dynamic prediction
### branch history table (BHT)
A small memory that is indexed by the lower portion of the address of the branch instruction and that contains one or more bits indicating whether the branch was recently taken or not.

## Exceptions Handling in MIPS
The basic action that the processor must perform when an exception occurs is to save the address of the offending instruction in the exception program counter (EPC) and then transfer control to the operating system at some specified address.
