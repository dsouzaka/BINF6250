# BINF6250
# Project 08: The Viterbi Algorithm

## Description
This project implements the Viterbi algorithm for hidden Markov models (HMMs). Given an HMM (initial, transition, and emission probabilities) and a sequence of observed symbols, the Viterbi algorithm finds the single most probable sequence of hidden states that could have produced the observations.

The algorithm uses dynamic programming: it fills a matrix with the best log score for each state at each position, stores a traceback pointer for each cell, and then follows the pointers back from the best final state to recover the path. All calculations are done in log space to prevent numerical underflow on long sequences.

The code is organized as a `HiddenMarkovModel` class so that later projects can add the forward, backward, and Baum-Welch algorithms as new methods on the same model.


## Documentation/Usage
### Setup
Requirements:
- Python 3

Clone the repository and switch to the project branch:

    git clone https://github.com/dsouzaka/BINF6250.git
    cd BINF6250
    git checkout project08_PR


### Running the project
Open the project notebook `project08/project08.ipynb` and run the cells in order. 

Create a model from the three probability dictionaries, in the same format the project spec gives them. The states and the alphabet are read from the dictionary keys, so nothing needs to be listed separately.

```python
init_probs = {"I": 0.2, "G": 0.8}
 
trans_probs = {"I": {"I": 0.7, "G": 0.3},      # trans_probs[from_state][to_state]
               "G": {"I": 0.1, "G": 0.9}}
 
emit_probs = {"I": {"A": 0.1, "C": 0.4, "G": 0.4, "T": 0.1},   # emit_probs[state][symbol]
              "G": {"A": 0.3, "C": 0.2, "G": 0.2, "T": 0.3}}
 
hmm = HiddenMarkovModel(init_probs, trans_probs, emit_probs)
 
result = hmm.viterbi("GGCACTGAA")
print(result["path"])
print(result["log_probability"])
```
 
Output:
 
```
['G', 'G', 'G', 'G', 'G', 'G', 'G', 'G', 'G']
-13.929108456051065
```

`viterbi()` takes a string or a list of symbols and returns a dict:
 
| Key | What it holds |
|---|---|
| `"path"` | the most probable hidden state path, first position to last |
| `"log_probability"` | natural log of the joint probability of that path and the observations |
| `"score"` | the score table, one `{state: log score}` dict per position |
| `"arrows"` | the traceback table, one `{state: previous state}` dict per position |
 
A model that is not a valid HMM (mismatched keys, a row that does not sum to 1, a probability outside 0 to 1, an emission row missing a symbol) raises a `ValueError` when it is created. An empty sequence, or a symbol no state can emit, raises a `ValueError` when `viterbi()` is called:
 
```python
hmm.viterbi("GGNAC")
# ValueError: No state can emit symbol N at position 2.
```

### Functions
`HMMSetup` (stores the model, checks it, builds the log tables)
 
- **`__init__(self, initial_probabilities, transition_probabilities, emission_probabilities)`**: Stores the three dictionaries, reads `states` from the initial probabilities and `alphabet` from the emission rows, then calls `validate()` and `to_log_space()`.
- **validate(self)**: Checks that every dictionary uses the same states, every emission row uses the same symbols, every probability is between 0 and 1, and every distribution sums to 1 (within 1e-6). Raises a `ValueError` naming the dictionary and state that failed.
- **safe_log(self, p)**: Natural log of `p`, returning `-inf` when `p` is 0.
- **to_log_space(self)**: Builds `log_init`, `log_trans`, and `log_emit` from the raw probabilities. Can be called again whenever the raw probabilities change.
- **ViterbiMixin** (adds Viterbi decoding, used through `HiddenMarkovModel`)
- **initialization_viterbi(self, sequence)**: Creates the score and arrow tables and fills position 0 with log initial + log emission of the first symbol.
- **calculate_viterbi_score(self, sequence)**: Fills positions 1 onward. For each state, keeps the best route from the previous position (max) and stores which state it came from (argmax), then adds the emission once.
- **termination_viterbi(self)**: Finds the state with the highest score at the last position. Raises a `ValueError` if every path has probability 0.
- **traceback_viterbi(self, final_state)**: Follows the arrows back from the final state and returns the path from the first position to the last.
- **viterbi(self, sequence)**: Checks the sequence, runs the four steps above, and returns the result dict.
- **HiddenMarkovModel(HMMSetup, ViterbiMixin)**: Combines the two. This is the class you create.



### Results and testing
**Hand-calculated values.** Worked out on paper before writing code, using the spec's example parameters:
 
| Observation | Expected | Notebook output |
|---|---|---|
| `"G"`, position 0 | I = log(0.2 × 0.4) = log(0.08), G = log(0.8 × 0.2) = log(0.16) | `{'I': -2.5257, 'G': -1.8326}` |
| `"GG"`, position 1 | I = log(0.0224) from I, G = log(0.0288) from G | `{'I': -3.7987, 'G': -3.5474}`, arrows `{'I': 'I', 'G': 'G'}` |
 
**Spec example (`"GGCACTGAA"`).** The best path stays in the background state `G` for all nine positions, with log probability -13.929. The GC-rich start looks like it might be an island, but the 0.8 starting probability and the 0.9 chance of staying in `G` make switching into `I` and back out too costly for such a short GC run.
 
**Coin example (`"HHTT"`, fair vs loaded coin).** Path `['L', 'L', 'F', 'F']`, log probability -4.464.
 
**Biased die example (Durbin et al., 1998).** Rolls `123666666121` give `Fair Fair Fair` followed by nine `Loaded`, log probability -20.488. The last three rolls (1, 2, 1) stay `Loaded` because switching back to `Fair` costs a 0.10 transition while staying costs 0.90, and three ordinary rolls are not enough evidence to pay for the switch.
 
**Input checks.** Each of these raises a `ValueError` with a message naming what failed:
 
```
Caught: Transition row for I has states {'I', 'X'}, expected {'I', 'G'}
Caught: Transition row for I sums to 0.8999999999999999, not 1
Caught: Emission probability -0.1 of A in state I must be between 0 and 1
Caught: Emission row for I has symbols {'C', 'A'}, expected {'G', 'C', 'A', 'T'}
```
 
**What these tests do not cover yet:** the hand-calculated checks are printed and compared by eye rather than asserted, and only short sequences were checked against hand or brute-force answers.



## Pseudocode
### HMMSetup
 
**`__init__`**
 
```
# store 3 probability dictionaries as attributes
# states = list of keys of initial_probabilities
# alphabet = every symbol that appears as a key in any emission row
# call validate() to check inputs and raise an error if something is wrong
# call to_log_space() to build log versions of the 3 tables
```
 
**`validate`**
 
```
# check that and raise a ValueError with a message showing what dict and state failed
#   transition dict has the same state as initial (rows and columns)
#   emission dict has the same states as initial (rows)
#   every emission row has the same symbols as the alphabet        (added)
#   initial probabilities sum to ~1
#   each transition row sums to ~1
#   each emission row sums to ~1
#   every probability is between 0 and 1
```
 
**`safe_log`**
 
```
# (added) if p is 0:
# (added)     return -inf (an impossible move or emission)
# (added) otherwise:
# (added)     return the natural log of p
```
 
**`to_log_space`**
 
```
#   build log versions of all 3 tables (self.log_init, self.log_trans, self.log_emit)
#   handle log 0 = -inf
#       if p = 0, store float('-inf') since a zero probability move is impossible
#   (added) can be called any time the raw probabilities change,
#   (added) so the log tables never go stale (needed for Baum-Welch later)
```
 
### ViterbiMixin
 
**`initialization_viterbi`**
 
```
#   create score grid: self.score[i][state] -> log score at position i, state
#   create arrow grid: self.arrows[i][state] -> which previous state won the max
#   fill only the first position (index 0, symbol = sequence[0]):
#   for each state:
#       score = log initial[state] + log emission[state][sequence[0]]
#       arrow = None (no previous state yet)
#   positions after are filled by calculate_viterbi_score
```
 
**`calculate_viterbi_score`**
 
```
#for each position i from 1 to len(sequence) - 1:
#   symbol = sequence[i]
#   score_column = empty dict
#   arrow_column = empty dict
#   for each current state:
#       best_value = -inf
#       best_previous = None
#       for each previous state:
#           candidate = self.score[i-1][previous] + log transition[previous][current]
#           if candidate > best_value:
#               best_value = candidate
#               best_previous = previous
#       score_column[current] = best_value + log emit[current][symbol]
#       arrow_column[current] = best_previous
#   append score column to self.score
#   append arrow column to self.arrows
```
 
**`termination_viterbi`**
 
```
#   best_score = -inf
#   best_state = None
#   for state in last column of self.score (self.score[-1]):
#       if self.score[-1][state] > best_score:
#           best_score = self.score[-1][state]
#           best_state = state
#   if best_state is None, raise error (every path impossible)
#   return best_state, best_score
```
 
**`traceback_viterbi`**
 
```
# path = [final_state]
# current = final_state
# for i from len(self.arrows) - 1 backwards to 1:
#   current = self.arrows[i][current] #state at position i-1
#   append current to path
# reverse path
# return path
```
 
**`viterbi`**
 
```
# if sequence is empty, raise ValueError
# for each position and symbol in sequence:
#   if no state has nonzero emission probability for a symbol
#       raise ValueError that names the symbol and position
# initialization_viterbi(sequence) #fills position 0
# calculate_viterbi_score(sequence) #fills position 1 onward
# best_state, best_score = termination_viterbi()
# path = traceback_viterbi(best_state)
# return {"path", "log_probability", "score", "arrows"}
```
 
### HiddenMarkovModel
 
```
# (added) combine HMMSetup and ViterbiMixin into one class
# (added) HMMSetup gives it the stored model and log tables
# (added) ViterbiMixin gives it the viterbi methods
# (added) future projects add their mixin to this list, nothing else changes
```
 
---


## Formulas
### The Viterbi Algorithm
Dynamic programming

$\pi_i$ = initial probabilities

* the chance that the first position is in state $i$

$a_{ij}$ = transition probabilities

* the chance that the next position is in state $j$ when this position is in state $i$

$b_j(k)$ = emission probabilities

* the chance that a position in state $j$ shows symbol $k$

Each row of the transition table and each row of the emission table adds up to 1, because each row lists every possible outcome from one starting point.

$\lambda = (A, B, \pi)$

* the three sets together

Assumptions (Markov property):

* The next state depends only on the current state, not on any earlier position
* Each observation depends only on the state at its own position

### Formal Description by Durbin et al.

Durbin writes emissions as $e_\ell(x_i)$ instead of $b_j(k)$; they mean the same thing. Durbin also adds a silent begin/end state, numbered 0.

#### Initialization ($i = 0$):

$$v_0(0) = 1, \quad v_k(0) = 0 \text{ for } k > 0$$

#### Recursion ($i = 1 \ldots L$):

$$v_\ell(i) = e_\ell(x_i) \cdot \max_k \big( v_k(i-1) \, a_{k\ell} \big)$$

$$\text{ptr}_i(\ell) = \arg\max_k \big( v_k(i-1) \, a_{k\ell} \big)$$

#### Termination:

$$P(x, \pi^*) = \max_k \big( v_k(L) \, a_{k0} \big)$$

$$\pi_L^* = \arg\max_k \big( v_k(L) \, a_{k0} \big)$$

#### Traceback ($i = L \ldots 1$):

$$\pi_{i-1}^* = \text{ptr}_i(\pi_i^*)$$

#### Glossary

* $\max$: the highest value
* $\arg\max$: which state gave the highest value (stored as the traceback pointer)
* $e_\ell(x_i)$: emission probability of symbol $x_i$ from state $\ell$
* $a_{k\ell}$: transition probability from state $k$ to state $\ell$
* $k$: the previous state (the one maximized over)
* $\ell$: the current state (the cell being filled)
* $L$: the length of the observation sequence
* $i$: the position in the sequence
* $x$: the observation sequence; $x_i$ is the symbol at position $i$
* $\pi^\*$: the most probable path; $\pi_i^\*$ is its state at position $i$ (not the same as the initial probabilities $\pi_i$)

### In our implementation

* Our model has no end state, so termination drops $a_{k0}$:

$$P(x, \pi^\*) = \max_k v_k(L), \qquad \pi_L^\* = \arg\max_k v_k(L)$$

* We are given initial probabilities directly, so column 1 is $v_k(1) = \pi_k \, e_k(x_1)$ and the recursion starts at $i = 2$.
* To avoid underflow, we work in log space (max stays max; products become sums):

$$\log v_\ell(i) = \log e_\ell(x_i) + \max_k \big( \log v_k(i-1) + \log a_{k\ell} \big)$$

## Design Decisions

### Why the code is built this way
 
**Dictionaries, keyed by state and symbol names.** The spec hands us the model as dictionaries, so we kept it that way all the way through. Every lookup reads like the math, for example `log_trans[previous][current]`, and because nothing depends on row numbers, a state label can never end up attached to the wrong row of numbers. One of our earlier array-based drafts had exactly that bug, where the output names were swapped, and nothing crashed, so this was a deliberate choice. The tradeoff is speed: pure Python loops are slower than NumPy, which only matters for very long sequences.

**States and alphabet come from the dictionaries.** `states` is read from the keys of the initial probabilities and `alphabet` from the emission rows. Nothing is hard-coded, so the same class runs the CpG example, the coin example, and the biased die example without changes.


**Validate before anything else.** `validate()` runs inside `__init__`, before the log tables are built. A bad model (a missing state, a row that does not sum to 1, a negative probability, an emission row missing a symbol) fails right away with a message naming the dictionary and state, instead of crashing somewhere in the middle of the recursion with a confusing `KeyError`.

**A mixin structure.** The class is split into pieces so each project can add an algorithm without rewriting the earlier ones:
 
| Class | Job |
|---|---|
| `HMMSetup` | stores the three dictionaries, validates them, builds the log tables |
| `ViterbiMixin` | adds Viterbi decoding (Project 08) |
| `HiddenMarkovModel` | combines them, and is the class you actually use |

A mixin is a class that adds abilities to another class rather than standing on its own. `ViterbiMixin` cannot run by itself, since it relies on `self.states` and the log tables that `HMMSetup` creates. Python searches the classes in the order they are listed (`HiddenMarkovModel`, then `HMMSetup`, then `ViterbiMixin`), which is called the method resolution order. We picked mixins over making Viterbi a subclass of the HMM because Viterbi is something you do with an HMM, not a kind of HMM, and because it keeps each algorithm in its own notebook cell.


## Successes
- **Two working implementations.** The group wrote the full algorithm two ways: one using NumPy arrays and one using dictionaries keyed by state and symbol names. Comparing the two helped catch bugs that a single version might have hidden.
- **One method per step of the algorithm.** Initialization, recursion, termination, and traceback are each their own method, called in order by `viterbi()`. This made each step easier to test on its own and will make it easier to reuse for the forward and backward algorithms.
- **Verified against hand-calculated results.** We tested the code against values worked out by hand (the first two columns of the grid for the spec's example), the occasionally dishonest casino from the lecture slides, and the biased die example from Durbin et al. (1998), and the outputs matched.
- **Helper functions to prevent bugs.** `to_log_space()` and `safe_log()` convert the probabilities to log space and handle zeros without crashing, and `validate()` checks the input dictionaries (matching keys, values between 0 and 1, rows summing to 1) before the algorithm runs, so a bad model fails right away with a clear error message.

## Struggles
- **Building the structure without skeleton code.** Unlike earlier projects, there was no template, so we had to decide on the class design, method breakdown, and data structures ourselves, while also planning for future projects to build on the same code.
- **Choosing between NumPy arrays and dictionaries.** We had working versions using each and had to pick one for the group code. We chose dictionaries because most of the group had approached the project that way. If later projects need the speed or matrix operations of NumPy (for example, Baum-Welch), we plan to switch to the array version.
- **Handling different input types.** The final code only accepts dictionaries. We did not implement it this time, but a data type conversion function that turns other formats (such as arrays) into the dictionary format would make the class more flexible in future projects.

## Reflections
### Group Leader: Katelyn
Starting this project, I had somewhat of an understanding of the Viterbi algorithm. Mara created an outline including the formulas and a basic structure for us to follow, and having that to reference and to follow was extremely helpful since I definitely struggled with not having any skeleton code for this project. We all worked on creating our own implementations from this structure and then had a meeting to discuss any thoughts we had, like how to address different data types and what structures would work best. Our group worked well together to help fill gaps in each other's understanding.
I used Claude to help walk me through examples of the Viterbi algorithm, which I felt really helped me understand what we were doing with this project. I also had it help me troubleshoot areas where I was stuck with the pseudocode, and I definitely found this project more difficult than past ones. Once we had the pseudocode, the Python was not too bad. 
Since this project is the start of many others, we had to make sure that we prepared for that. This was definitely different than other projects because we didn't previously need to change things so they could be expanded upon, but this process seems to better represent what this process would look like in the real world. I am interested to see how this project will be expanded upon in the future modules and how we can add more algorithms to our HMM class.

### Trivedi: 
I’d actually worked with HMMs before, in BINF 6400 (Genomics) last semester, where we covered Viterbi, posterior decoding, and CpG island models, and built HMM classifiers for a problem set. So I came in thinking this would mostly be review, but it felt pretty different. There, the focus was on using HMMs and getting the math right for a specific problem. Here, there was no starter code at all; the model could have any number of states or symbols, and the code had to be designed so other algorithms could be added later. That meant I had to understand the algorithm well enough to build it from nothing, not just apply it. So before touching any code, I worked through the first two columns of the Viterbi grid by hand with the spec’s example, and that ended up being the most useful thing I did all project. I kept getting stuck on small things, like using only the starting probabilities in column 1, or mixing up the state G with the nucleotide G, or thinking the emission decided the traceback. I also expected log space to change which route wins the max, but comparing the actual numbers showed it doesn’t, which is how we noticed the spec’s line about the max turning into a sum was wrong for Viterbi.


### Other Members: Mara
I was generally unfamiliar with HMMs going into this project, and did not have any experience with the Viterbi algorithm. Because we were given no background or template code like we usually would, I went ahead and compiled everything from the assignment and primer pages to "recreate" the way we normally do projects, which I think helped a lot. I also did the calculations by hand, although without converting to log, to double-check the output. This project built off of the previous projects very nicely, particularly with the recursion and traceback. In my code version, I structured everything around NumPy arrays for more efficient indexing, but struggled with input data conversion, such as the provided examples being dictionaries. The indexing, while not necessarily hard, was difficult to keep track of, causing a lot of debugging towards the end. 

This was also my first time future-proofing. It is much more difficult than I had originally anticipated because, without knowing what will need to be expanded in the next projects, it's hard to know the best way to structure everything. Because we knew the topic of the next projects, we had an idea of what to do, but without having learned it in class or seen the assignment yet, it was a challenge. My group mates and I were able to come up with insights into how to structure our code to accommodate this. 



## Appendix (AI usage, citations, etc.)
### References
Durbin, R., Eddy, S. R., Krogh, A., & Mitchison, G. (1998). *Biological sequence analysis: Probabilistic models of proteins and nucleic acids*. Cambridge University Press.

Feng, D.-F., & Doolittle, R. F. (1987). Progressive sequence alignment as a prerequisite to correct phylogenetic trees. Journal of Molecular Evolution, 25(4), 351–360. https://doi.org/10.1007/BF02603120

Forney, G. D. (1973). The Viterbi algorithm. *Proceedings of the IEEE*, 61(3), 268–278.

Gascuel, O., & Steel, M. (2006). Neighbor-joining revealed. Molecular Biology and Evolution, 23(11), 1997–2000. https://doi.org/10.1093/molbev/msl072

Kosakovsky Pond, S. L., Weaver, S., Leigh Brown, A. J., & Wertheim, J. O. (2018). HIV-TRACE (TRAnsmission Cluster Engine): A tool for large scale molecular epidemiology of HIV-1 and other rapidly evolving pathogens. Molecular Biology and Evolution, 35(7), 1812–1819. https://doi.org/10.1093/molbev/msy016

Langmead, B. (n.d.). *Hidden Markov models* [Lecture slides]. Johns Hopkins University. https://www.langmead-lab.org/teaching-materials

Rhee, S.-Y., Gonzales, M. J., Kantor, R., Betts, B. J., Ravela, J., & Shafer, R. W. (2003). Human immunodeficiency virus reverse transcriptase and protease sequence database. Nucleic Acids Research, 31(1), 298–303. https://doi.org/10.1093/nar/gkg100

Saitou, N., & Nei, M. (1987). The neighbor-joining method: A new method for reconstructing phylogenetic trees. Molecular Biology and Evolution, 4(4), 406–425. https://doi.org/10.1093/oxfordjournals.molbev.a040454

Smith, T. F., & Waterman, M. S. (1981). Identification of common molecular subsequences. Journal of Molecular Biology, 147(1), 195–197. https://doi.org/10.1016/0022-2836(81)90087-5

Studier, J. A., & Keppler, K. J. (1988). A note on the neighbor-joining algorithm of Saitou and Nei. Molecular Biology and Evolution, 5(6), 729–731. https://doi.org/10.1093/oxfordjournals.molbev.a040527 


### AI Usage
Katelyn: I used Claude (Anthropic) throughout this project, mainly to understand the algorithm and check work. I used it to walk me through the Viterbi algorithm step by step, explain related concepts such as conditional independence and object-oriented programming, and review and correct pseudocode and the formatting of the formulas in the notebook. Claude also guided me through writing a dictionary-based implementation, which became the base of the group’s final code: it pointed out errors and gave hints while I wrote most of the methods myself, and it wrote the `__init__` method body and the docstrings and comments. I also used it to test implementations against hand-checked examples, which helped identify bugs and edge cases, and to draft the Setup section of the README and revise my Description section.

Trivedi: I used Claude (Anthropic) to help me understand calculations and assist me in understanding how the Viterbi traceback works. I asked Claude how decorators and OOP concepts will help me keep my code more dynamic and open to future changes. Claude also helped me write tests for the functions to confirm the expected output and calculations. I also revisited notes and materials from BINF6400 on HMM. 

Mara: I used ChatGPT (OpenAI) to double-check my indexing throughout the program and to establish my Class and initialization in such a way that I can add to it in the next projects. Prompt used: "Implementing the Viterbi algorithm for HMM and checking my indexing through the NumPy arrays. Do not write any code for me"
