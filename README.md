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

- where ```.``` and ```S``` are sequences of only ```A```'s and ```B```'s, no spaces, in any order. 
- *NOTE: The length of ```S``` should be **strictly greater** than the length of any defined ```.```. We do not check for this, and inputting strings longer than the program is expecting will lead to undefined behavior.*

We will refer to the ```[.|.|.$.|.|.]``` segment as the **RULES** and the ```S#``` segment as the **BOARD**.

#### Example Inputs:

**GOOD:**
- [||$||]ABABABABAB#
- [B|AA|$A|BB|]BAAABBB#

**BADE:**
- [AAAAA||$||]#
- [B|AA|$A|BB|]BAAABBB
- [b|aa|$a|bb|]BBBBBBB#


### Understanding the rules
```[.|.|.$.|.|.]``` is where the user defines the rules for the simulation.

The rules for species ```A``` are defined in the first ```[.|.|.$``` chunk. The rules for species ```B``` are defined in the second ```$.|.|.]``` chunk.

Species rules are interpreted as HUNT|REPRODUCE|TRANSFORM. That might not make much sense, so let us give an example.

E.g. ```[AA|AA|AB$A|BB|]BBAAABA#```  
First, we look at the rules for species A in the ```[BB|A|AB$``` chunk.  

Interpreting this chunk:  
- ```[AA|```: this defines what species A can hunt. If ```A``` is adjacent to a sequence of **AT LEAST** 2 ```A```'s, the 2 adjacent ```A```'s are erased.
Order also matters. Specifying that species A can hunt ```BA``` does not allow them to hunt ```AB```.

- ```|AA|```: if ```A``` **meets** the hunting condition, **REPLACE** ```A``` with ```AA```. Note that if the hunting section contains no letters, ```A``` will always execute this path.
- ```|AB$```: if ```A``` **fails to meet** the hunting condition, **REPLACE** ```A``` with ```AB```.

For ```$A|BB|]```, we interpret things the same way, just from ```B```'s point of view. 

*Note that if 2 delimiters have no A's or B's in between, e.g. ```|]```, it is assumed that the rule is empty.*

### Parsing Precedence

*The parser always evaluates left precedence over right precedence. An animal only attempts to hunt other animals to the right if it cannot find the animals it needs to hunt on the left.* 

*E.g. if we were currently reading the ```A``` using the same rules, ```BBABB``` evaluates to ```AB BB``` **not** ```BB ABB``` (spaces are not allowed in the board, but relevant characters are grouped together for clarity in this example) after hunting. This prevents ambiguity.*

*Also, animals that have already been marked as prey by one animal cannot be marked as prey again by another animal. Animals that likewise have sucessfully hunted other animals cannot be hunted by another animal (these animals have already established themselves as apex predators, and it would make little sense to hunt them). Everyone else? Fair game. This will make more sense in the example.*

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

Since ```B```'s can only hunt is ```A```, ```B``` does not meet the requirements to hunt and then REPRODUCE.

(current state: ```[AA|AA|AB$A|BB|]yBAAABA#[AA|AA|AB$A|BB|]```) 

We then scan right. We see that to the right of ```y``` (temporarily representing scanned ```B```) is a ```B```. But ```B``` can only hunt ```A```, not ```B```. We revert ```y``` back to ```B```, indicating that the animal could not hunt for food. Then, we start the scanning process over again for the next element. 

**(ii):**
The next element we scan is also a ```B```, so it gets converted to a ```y```.

(current state: ```[AA|AA|AB$A|BB|]ByAAABA#[AA|AA|AB$A|BB|]```) 

We now repeat **(i)**. Scanning left, we find a ```B```. That does not match an ```A```, so we continue by scanning right. 

Scanning right, we find ```A```. That matches what ```B``` can hunt. We mark ```A``` as ```B```'s prey by writing ```A``` in lowercase and continue to the next element. ```B``` also gets converted to an ```&``` to signify that it will reproduce.

(current state: ```[AA|AA|AB$A|BB|]B&aAABA#[AA|AA|AB$A|BB|]```) 

**(iii):**

We scan ```a``` next. But ```a``` has already been marked as prey. So we skip ```a``` and scan right to the next non-lowercase element (the ```A```). Mark ```A``` as ```x``` to denote that it is the current element we are tracking. 

(current state: ```[AA|AA|AB$A|BB|]B&axABA#[AA|AA|AB$A|BB|]```) 

To the left of ```x``` is an ```a```. However, it is in lowercase, so it has already been marked as prey by another animal. ```x``` cannot feast on it. 

We then scan right. To the right of ```x``` is ```AB``` not ```AA```, so ```x``` cannot hunt. We revert ```x``` back to ```A``` and continue.

**(iv):**

You should get the hang of this by now. Replace the current scanned animal with a temporary variable (we use ```x``` for ```A```), then scan left and right.

(current state: ```[AA|AA|AB$A|BB|]B&aAxBA#[AA|AA|AB$A|BB|]```) 

To the left, there is ```aA```, but ```a``` doesn't count, since that food source has already been taken by someone else.

Scanning to the right, we see ```BA```. That doesn't match ```AAA```, so our current marked animal gets to transform. For now, our ```x``` will remain an ```A```. 

**(v)**

Again, replace the current scanned variable with a fitting alias. We use ```y``` since this is an animal of species ```B```.

(current state: ```[AA|AA|AB$A|BB|]B&aAAyA#[AA|AA|AB$A|BB|]```) 

We scan left first. To the left of ```y``` is ```A```, so our current animal has found his prey.  

We modify the ```A``` left of the the ```y``` to ```a```, to show that the ```A``` has been preyed upon.  

Since ```y``` had a nice meal, it can now reproduce, so we replace ```y``` with ```&``` and then move on to the final animal. Note that since the animal got it's food from hunting creatures on the left, it sees no need to hunt animals to the right of it. 

(current state: ```[AA|AA|AB$A|BB|]B&aAa&A#[AA|AA|AB$A|BB|]```) 

**(vi)**

Now we look at the ```A``` right before the ```#```. To the left of it is an ```&```, meaning an animal who has already hunted. You cannot hunt an animal who has already hunted (why hunt something so dangerous)?

To the right is #. That means the end of the board. Since neither scans bring up ```AAA```, the last animal we see does not successifully hunt, so it transforms. 

#### Step 3: Expanding

So now we have a string like this: 
```[AA|AA|AB$A|BB|]B&aAa&A#[AA|AA|AB$A|BB|]```  

We want our final output to be like so: 
```[.|.|.$.|.|.]S#```  

That way, the output of this program can be cycled over and over again.  

The last thing to do is to copy over the new board so we get another string in the following format (```[.|.|.$.|.|.]S#```) without the ```@``` and ```&```.  

This is done via a very simple copy:
- Capital letters (```A```, ```B```) are animals who were unable to hunt but were lucky enough to not be preyed upon. This means copying over the corresponding TRANSFORM rules when one of these is encountered.
- ```@``` and ```&``` are used for encoding animals for species ```A``` and ```B``` respectively who were able to hunt. 
- Lowercase letters (```A```, ```B```) are the unlucky ones. They were preyed upon and are now presumed to be dead. We will not keep records of them in the next board.  

- B &aAa&A  
- & aAa&A  
- BB a Aa&A  
- BB A a&A  
- BBAB a &A  
- BBAB & A  
- BBABBB A  
- BBABBBAB
- BBABBBAB#

When done to completion, ```[AA|AA|AB$A|BB|]B&aAa&A#[AA|AA|AB$A|BB|]``` becomes ```[AA|AA|AB$A|BB|]BBABBBAB#```