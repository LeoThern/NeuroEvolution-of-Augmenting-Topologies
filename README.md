# NEAT: NeuroEvolution of Augmenting-Topologies
Implementing and experimenting with NEAT algorithm on RL problems, gymnasium as benchmark.

---

## Original Paper

“Evolving Neural Networks through Augmenting Topologies”, Stanley & Miikkulainen in 2002 https://nn.cs.utexas.edu/downloads/papers/stanley.cec02.pdf

### Summary

Novel techniques to allow for *gene* tracking in neural networks (aligning network features during crossover) and speculative changes in the topology of the network. (As part of neural evolution reinforcement learning)

### Key Methods

- Historical Markings (align genomes by *Innovation Number*)
- Speciation (protects novel topology changes)
- Complexification (start simple, evolve to complex)

## Network Encoding (Genome)

Each network is encoded by a set of nodes and a set of edges:

- Node gene
    - type {input, hidden, output}
- Connection genes
    - In-node
    - Out-node
    - Weight
    - Enable-bit
    - Innovation Number (global marker)

The original NEAT implementation used a modified (steepened) sigmoid activation function for all nodes: $y = \frac{1}{1+e^{-4.9x}}$.

## Speciation and Fitness Sharing

The population of networks is clustered by their *Compatiblity Distance* $\delta$ into species, an individuals raw fitness is adjusted to a shared fitness by dividing by size (count) of their species.

$$
\delta = \frac{c_1 E}{N}+\frac{c_2 D}{N}+c_3 \cdot \Delta \bar{W} \qquad f'_i = \frac{f_i}{\vert S\vert}
$$

E = number of excess genes (non-matching outside of common range)

D = number of disjoint genes (non-matching inside of common range)

(most implementations skip differentiating between excess and disjoint genes)

$\Delta\bar{W}$ = Average weight difference of matching genes

N = number of genes in larger genome

c_{1,2,3} = hyperparameters

### Clustering

For each cluster center we pick a random (or highest fitness) representative from each species prev. generation, then assign every individual greedily to the first species/cluster where *Compatiblity Distance* $\delta$ is smaller than a threshold.
Instead of a fixed threshold you can also adjust it dynamically from target number of species.

The number of offspring allocated to a species is proportional to the sum of their (shared) fitness.

## Evolution Loop

The allocated offspring count is typically divided into 25% mutation without crossover (cloning and mutation from rank-based probabilistic selection) and 75% crossover/mating.

A small probability (e.g. 0.1%) is also allocated to *interspecies mating* (selecting second parent from population excl. own species).

To combat stagnation a drop-off age is implemented: if a species maximum fitness does not improve within X generations it is considered stagnant and their fitness is penalized (e.g. set to 0). The species containing the single best individual is typically immune from this.

### Initialization

The initial population consists only of *minimal* networks, only with fully connected input and output nodes, with random weight.

*This optimizes the lowest-dimensional space possible first before expanding*

### Selection

Basic elite selection based on fitness score, the single best individual survives without mutation.

### Crossover

Parent 1 and Parent 2 are randomly matched, genes are aligned by their *Innovation Number.*

Genes with matching *Innovation Number* (homologous) are simply inherited randomly from either parent, with the enable-bit staying off with 75% probability if it is disabled in either parent.

Non-matching genes are inherited from the more fit parent (or both if equal fitness).

### Mutations

Probabilities are hyperparameters.

- weight mutation
    - for every connection gene either:
        - perturbation (~90%) - add sample from uniform distribution (clipped)
        - replacement (~10%) - choose sample from uniform distribution

- add link
    - adds connection with incremented *Innovation Number* and random weight
- add node
    - an existing connection is split by flipping the old enable-bit and creating two new connections (N_1 → N_new [weight=1.0] and N_new → N_2 [weight=original])
    - for each new connection incremented *Innovation Number*

If two identical mutations occur in the same generation by chance, NEAT assigns them the same innovation number.
