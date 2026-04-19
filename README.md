COSC 312  
HW 8b: Turing Machine Project  
19 April 2026  

# Territorial Battle Simulation

## Contributors
Vincent Guo  
Minjae Bae

## Problem Description

The goal of this project was to develop a Turing machine that could simulate a territorial battle between two animals in a crowded environment (a 1D tape), where the user can initalize the starting board configuration, some rules, and the number of iterations. 

The turing machine then iteratively applies the rules, scanning from left to right, applying the user-defined rules on each animal. 

### The input string
The program takes input in the following format:  
```[.|.|.$.|.|.]S#```

- where ```.``` and ```S``` are sequences of ```A```'s and ```B```'s, in any order. 
- *NOTE: The length of ```S``` should be **strictly greater** than the length of any defined ```.```. We do not check for this, and doing so may lead to undefined behavior.*

We will refer to the ```[.|.|.$.|.|.]``` segment as the **RULES** and the ```S#``` segment as the **BOARD**.

### Understanding the rules
```[.|.|.$.|.|.]``` is where the user defines the rules for the simulation.

The rules for animal ```A``` are defined in the first ```[.|.|.$``` chunk. The rules for animal ```B``` are defined in the second ```$.|.|.]``` chunk.

Animal rules are interpreted as CONSUME|REPRODUCE|TRANSFORM. That might not make much sense, so let us give an example.

E.g. ```[AA|AA|AB$A|BB|]BBAAABA#```  
First, we look at the rules for animal A in the ```[BB|A|AB$``` chunk.  

Interpreting this chunk:  
- ```[AA|```: if ```A``` is adjacent to a sequence of **AT LEAST** 2 ```A```'s, the 2 adjacent ```A```'s are erased.
- ```|AA|```: if ```A``` **meets** the CONSUME condition, **REPLACE** ```A``` with ```AA```. Note that if the CONSUME section contains no letters, ```A``` will always execute this path.
- ```|AB$```: if ```A``` **fails to meet** the CONSUME condition, **REPLACE** ```A``` with ```AB```.

For ```$A|BB|]```, we interpret things the same way, just from ```B```'s point of view. 

*Note that if 2 delimiters have no A's or B's in between, e.g. ```|]```, it is assumed that the rule is empty.*

### Parsing Precedence

*The parser always evaluates left precedence over right precedence. An animal only attempts to consume other animals to the right if it cannot find the animals it needs to consume on the left.* 

*E.g. if we were currently reading the ```A``` using the same rules, ```BBABB``` evaluates to ```AB BB``` **not** ```BB ABB``` (spaces are not allowed in the board, but relevant characters are grouped together for clarity in this example) after the CONSUME step. This prevents ambiguity.*

*Also, animals that have already been marked as prey by one animal cannot be marked as prey again by another animal. Animals that likewise have satisfied the conditions for CONSUMING cannot be CONSUMED by another animal. This will make more sense in the example.*

### Approach

#### Step 1: Copying Rules

Before parsing the board, we copy the RULES section to the end. We have to do this at some point, so doing it at the beginning was what we elected to do. This is done via a duplicator.

![Duplicator Image](./RuleDuplicator.png)

Nodes 0 sets the turing pointer to the first element of the input string. We then loop (yellow nodes), marking elements as we go and copying characters to the first instance of whitespace, before looping back and using the mark to find the next element we need to copy. This is done until we reach the end of the rules, marked by ```]```. 

After this step:
- ```[AA|AA|AB$A|BB|]BBAAABA#``` becomes ```[AA|AA|AB$A|BB|]BBAAABA#[AA|AA|AB$A|BB|]```.  
- ```[||$||]BBABBA#``` becomes ```[||$||]BBABBA#[||$||]```.

#### Step 2: Parsing the board
Continuing with ```[AA|AA|AB$A|BB|]BBAAABA#```.  

(```[AA|AA|AB$A|BB|]BBAAABA#[AA|AA|AB$A|BB|]``` after step 1)  

We read the board ```BBAAABA#```.  

**(i):**  
We read ```B```. This means use ```B```'s rules (```$A|BB|]```). 

We first keep track of our current char by marking ```B``` temporarily as ```y```, and then scan left. We read ```[```.  

Since ```B```'s consume rule is ```A```, ```B``` does not meet the requirements to CONSUME and then REPRODUCE.

(current state: ```[AA|AA|AB$A|BB|]yBAAABA#[AA|AA|AB$A|BB|]```) 

We then scan right. We see that to the right of ```y``` (temporarily representing scanned ```B```) is a ```B```. But the CONSUME rule for ```B``` specifies that the sequence starts with an ```A```. ```B``` does not match that, so we revert ```y``` back to ```B```, indicating that the animal could not find food to consume. Then, we start the scanning process over again for the next element. 

**(ii):**
The next element we scan is also a ```B``` so it gets converted to a ```y```.

(current state: ```[AA|AA|AB$A|BB|]ByAAABA#[AA|AA|AB$A|BB|]```) 

We now repeat **(i)**. Scanning left, we find a ```B```. That does not match an ```A```, so we continue by scanning right. 

Scanning right, we find ```A```. That matches the CONSUME rule for ```B```. We mark ```A``` as ```B```'s prey by writing ```A``` in lowercase and continue to the next element. ```B``` also gets converted to an ```&``` to signify that it will reproduce.

(current state: ```[AA|AA|AB$A|BB|]B&aAABA#[AA|AA|AB$A|BB|]```) 

**(iii):**

We scan ```a``` next. But ```a``` has already been marked as prey. So we skip ```a``` and scan right to the next non-lowercase element (the ```A```). Mark ```A``` as ```x``` to denote that it is the current element we are tracking. 

(current state: ```[AA|AA|AB$A|BB|]B&axABA#[AA|AA|AB$A|BB|]```) 

To the left of ```x``` is an ```a```. However, it is in lowercase, so it has already been marked as prey by another animal. ```x``` cannot feast on it. 

We then scan right. We see an ```A``` to the right of ```x```, which is part of what we need for the CONSUME rule of ```A``` (```AA``` if you have forgotten).

However, to the right of ```x``` is ```AB``` not ```AA```. ```x``` does not meet the CONSUME rule, so we scan right.

**(iv):**











