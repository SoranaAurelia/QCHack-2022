<b>The solution is implemented in the QuantumCrew_Grovers_Challange.ipynb</b>

Team name: QuantumCrew

## Summary

1. ### Introduction
2. ### Solution
    1. Analyzing the connected edges
    2. Modeling idea
3. ### Implementation
4. ### Optimization
5. ### Simulations
6. ### Discussions



## Introduction
Challenge: find a Hamiltonian cycle in the given graph

<img width="615" alt="Screenshot 2022-04-10 at 06 13 54" src="https://user-images.githubusercontent.com/93521998/162608726-4403ee8d-7a50-4cf2-a646-a0e70acf31c5.png">


A Hamiltonian cycle is a path visiting all vertices, starting and ending on the same vertex.

In order to implement the Grover Search for this problem, we needed to figure out a few things, the most important one beeing: how do we encode the data?

After this we need to implement the following:
- a phase oracle that checks the solution (based on the way we decided to encode them)
- a diffuser (also called the reflection operator) for the grover search
- the circuit itself

## Solution 

### Analyzing the connected edges

Firstly, we tried to find in the graph certain aspects or properties that would make the problem encoding and implementation easier. We tried modeling the graph by its edges and we found:
    `0 -> 2, 3, 4`; 
    `1 -> 2, 3, 4`; 
    `2 -> 0, 1, 4`; 
    `3 -> 0, 1, 4`; 
    `4 -> 0, 1, 2, 3`.
    
We saw 4 like a "common factor" (since there's an edge between each other node and it), so we tought about taking it out of the ecuation since we can always start and end on this node. This also simplifies the checks we have to do on a solution because we don't need to also check if our solution is a cycle. 

### Modeling idea

With node 4 having the above property, we can minimize our oracle size and qubit count by excluding node 4 from the calculation, and concentrating on the solutions of the form 4-A-B-C-D-4, where A,B,C and D are the other nodes. Therefore, the problem becomes: how do we choose the order of nodes in the cycle so that the cycle is a valid one? Expanding further, a valid cycle would be one in which the nodes are connected by existent egdes and also it must be hamiltonian (passes once and only once through each node). Also, there are possible solutions of this form, like for example the cycle 4-3-1-2-0-4.

Using this observation, only 4 nodes remain to be taken in considaration, nodes 0, 1, 2 and 3. The earlier model in which we mappeed every node to all its neighbour nodes, now shrinks to:
0 -> 2, 3; 
1 -> 2, 3; 
2 -> 0, 1; 
3 -> 0, 1; 

(Hmm, interesting how 0 and 1 have the same neighbours, and that the same is true for 2 and 3 as well)

The shrinked graph:

<img width="249" alt="Screenshot 2022-04-10 at 11 00 28" src="https://user-images.githubusercontent.com/93521998/162608910-8cd0a73f-5bbb-4bbd-9104-a13f12c413ea.png">


Looking at the numbers from a binary perpsective, if we were to encode the nodes in the circuit, since they are 4 at number, we could use 2 qbits for each to map them like:
0 would be mapped as 00,
1 would be mapped as 01,
2 would be mapped as 10,
3 would be mapped as 11.

This mapping proves useful when we look closer at the binary version of our shrinked node mapping model:
`00 -> 10, 11    // 0 -> 2, 3`
`01 -> 10, 11    // 1 -> 2, 3`
`10 -> 00, 01    // 2 -> 0, 1`
`11 -> 00, 01    // 3 -> 0, 1`

In this mapping we chose, we happen to be able to easily check if 2 nodes are connected by an edge, by simply looking at the left digit from the binary representation of the 2 nodes. If the digits are opposite, the nodes are connected.  

To make a better use of this observation, we can look at our 4 nodes representation in the following way:
    
     0 |  1 |  2 |  3
    ax | by | cz | dt
    
Useful query we can easily answer:
    - are nodes of positions 0 and 1 connected <=> is a != b ?
        
We implemented the operation above by using a custom gate with a XOR like behaviour: it takes 3 qbits, and writes on the 3rd the result of the operation qubit_1 XOR qubit_2.
    
### Defining a Hamiltonian Cycle
    
When we tought about how to check for a hamiltonian cycle, we observed that obeying 2 simple rules is enough:
    1. every consecutive node should be linked by an edge (4 total nodes => 3 edges to consider => 3 checks)
    2. no repeating nodes (can be done with 2 checks)
    

## Implementation

Since we have 4 nodes, and we're using 2 qubits for the encoding of a node, we need 8 qubits. From 0 to 7, we have the following qubit pairs to represent the nodes: (0,1), (2,3), (4,5), (6,7).
    
Those qubits are put in superposition. For a solution, the qubits should have the following properties:
- 0 != 2 (qubit_0 different from qubit_2 = edge in between)
- 2 != 4 (edge in between)
- 4 != 6 (edge in between)
- 1 != 5 (*)
- 3 != 7 (*)
    => 5 checks => we use 5 aux qubits and 1 for the final multi-toffoli gate

The last 2 checks ensure, alongside the first 3, that no node is repeating. We should keep in mind the notation we presented before:
    -0-| -1-|-2-|-3-
    ax | by | cz | dt
    
The first 3 checks are to be sure all nodes are connected. This means adjacent nodes cannot have the same value in their left qubit, meaning:
    a != b
    b != c
    c != d.
And since those are binary numbers, a must be equal to c. Knowing this, our nodes binary represantation would be:
    -0-|-1-|-2-|-3-
     ax | by| az|bd

Since a != b, we can only have duplicate nodes on either nodes 0 and 2 or nodes 1 and 3. To check those are different, we simply look to have:
    x != z
    y != d.
    
### Last step to finish the oracle

With this, the problem specific part of the oracle is alost done. We found that our solutions have the properties that:
    a != b
    b != c
    c != d
    x != z
    y != d
    
A Hamiltonian cycle starting from node 4, always has the properties above. All that is left is to take the results of the fifth operations above and put them into a Multi-Controlled Toffoli Gate (works like an and) with all 5 aux qubits as the controls. This result is written on another qubit, but this is done only to trigger the phase kickback to mark the solutions with a negative phase.



## Optimization

    Out first implementation uses 14 qbits. 
    While writing this documentation, we figured out that based on the modeling of the problem that we used and on the input graph:
        - every node in the shrinked graph has exacly 2 neighbours
        - we only consider 4 nodes
        - the nodes can be divides in 2 groups: those that in binary representation starts with 0 or with 1. Those that start with the same digit cannot be found together
        
    Having the above means that is enough to compute using only 2 nodes, since they are also defining the remaining 2.
    Explanation:
        We had initially: node_1 - node_2 - node_3 - node_4.
        To have common edge, adiacent nodes need to start with the same digit in binary representaition.
        Since no duplicates are allowed, is node_1 start with 1 in binary, then node_3 also needs to start with 1. There are only 2 such numers, so after finding one, you immediatly can know the other one. 
        
   This solution only uses 8 qbits, as oposed to 14, so it's master and uses less memory but it loses solutions. The pool of possible solutions is bigger in the first implementation.
   
   
   The first idea was to check just once the 2 nodes to see if they are connected, but this condition has the entire half of the possible results as solutions, and this doesn't work with Grover's Algorithm. In this situation, we had to add 2 more checks on our 4 qubits to restrict the pool of possible solutions and to have a way to detect them by looking at the probabilities.
   
   # Simulations
   
   *<b>A small summary with the different solutions found by different simulators and arhitectures can be found in the Visualisation & Disscusion topics (the last two topics) from the QuantumCrew_Grovers_Challange.ipynb. </b>*
   
   # Discussions

We think that the comparison between the two solutions we provided is nicely summarised by the following table:

| Criteria | First solution: 14 qubits | Second solution: 8 qubits |
| --- | --- | --- |
| Pros: |  returns all the possible hamiltonian cycles from the graph |  smaller circuit that can be simulated (faster) |
| Cons: |  requires more gates and qubits to run |  returns only part of the solutions for the problem |
| Generalize? |  might be easier to generalize on a smaller graph if the oracle is implemented using the edges |  harder to apply to other problems |


While the first implementation seems to be easier to expand to a more general solution as well as giving us all the possible cycles from the graph it has one downfall: the depth of the circuit is quite large. This can be a problem, for example we weren't able to get it to run on the SV1 simulator since the toffoli gate implemented in qiskit required more qubits to implement in braket than available at the time. The second solution on the other hand, while utilising a smaller circuit, it seems to us that is constructed to work especially well on this task, and might not be scalable for a more general solution (also, is reliable to give only part of the solutions, so not quite good for a more general task of finding all the hamiltonian cycles)


