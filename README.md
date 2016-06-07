# Parallel Multi-Objective Optimization via Genetic Algorithm

## Abstract

I implemented a massively parallel genetic algorithm framework that resembles a modified version of the island model. The framework is built atop CUDA 7.5 and utilizes the experimental device lambda feature. I used this framework to compute Nash equilibrium by coevolving game-playing strategies between interacting agents.

## Background and Design

### Game Theroy

Game theory studies the interactions of rational decision-makers. We can simulate various games using evolutionary strategies to determine the equilibrium behavior, or Nash equilibrium, of the players.

Consider the game following game in which two countries choose a tarrif rate to impose on imports. Each function `u1` and `u2` represents the payoff of a given country as a function of the rates `t1` and `t2` imposed.

```
u1(t1, t2) = -t1(t1 - t2 - 2)
u2(t1, t2) = -t2(t2 - t1 - 8)
```

The Nash equilbiria of this strategic game is the equilibria where neither country would increase their payoff by increasing their tariff. In this particular game, analytical computations calculate that the unique Nash equilbrium is `(t1*, t2*) = (4, 6)`. We will determine whether similiar results can be achieved using our genetic algorithm framework.

### Genetic Algorithms

Genetic algorithms are used to solve optimization problems using a process similar to natural selection. This class of algorithms generates and measures the fitness of massive populations across many iterations, so it’s a promising candidate for GPU acceleration since each gene's fitness can be computed on an individual thread.

We will use the methods described in "Nash Genetic Algorithms" [[1]](#references) to compute Nash equilibrium via genetic algorithm, though we will modify it for better parallelization. Note that Nash equilibrium is computed by having each player optimizing their fitness function while the other parameters are fixed by the competing players. 

In this project, the genetic representation of a given agent corresponds with a particular strategy. Our implementation will isolate subpopulations within each multiprocessor “island” since this eliminates the slow process of inter-multiprocessor communication. Further, this allows us to minimize copy operations into slower memory. These “islands” will compete and evolve independently except for the occasional asynchronous “migration” of agent between island.

The migrations will transfer the most fit individuals between islands in order to provide better competitors to compete against, therefore getting closer to the Nash equilbrium. Random mutations and crossing of genes will make sure we reach the global optimum rather than simply a local optimum.

## Previous Work

### GPU–Accelerated Genetic Algorithms

Researchers have shown that CUDA implementations of genetic algorithms can introduce significant speedups over their CPU counterparts. Specifically, researchers have proposed an “island”-based model for fast massively-parallel computation [[3]](#references). Competing models involve generating the populations on the CPU and running the fitness tests of the GPU, but this doesn’t have quite as large a speedup due to expensive copying operations. It has been shown that the parallel island model produces solutions with similar quality to the CPU based algorithm despite the modifications.

### Co-Evolvability of Games

Researchers have studied the ability of coevolutionary genetic algorithms to evolve game-playing strategies [[4]](#references). Some studies have found that Nash equilibrium can be found by genetic algorithms. If this is feasible for all games, accelerated evolutionary algorithms might provide a fast way to estimate mixed Nash equilibrium, which is sometimes too computationally complex to feasibly compute analytically. It is not clear that this sort of computation is always possible though. Some studies note that collusion develops among agents that coevolve, and thus the Nash equilibrium is not always found. It seems that the feasibility of its computation might depend on the game type, the fitness function, and the evolutionary technique. 

## Implementation

### Genetic Algorithm

The genetic algorithm framework is implemented using CUDA device lambdas so any arbitrary genetic algorithm can be encoded in the framework without having to write a separate kernel. The framework supports specification of `spawn`, `evaluate`, `mutate`, and `cross` functions to specify the algorithm. Each function takes in a `curandState_t` that has been seeded specifically for that thread so that randomized behavior is easily implemented. Note that we seed each thread with its thread id. It is sufficient to do this since we only care that each thread has independent random numbers, but we do not care if the program acts randomly between invocations.

The `simulate` function also takes in a specification parameter that indicates how the algorithm should run, including the number of iterations. Further, the developer can specify the `kingdom_count`, the `island_count_per_kingdom`, and the `island_population`. These parameters require some explaning. The island model typically involves using a separate block for independent evolution of each population. We similarly regard each block as an island, but extend to model to include the concept of a "kingdom". A kingdom is an alliance of islands that is acting to achieve the same goal. In our implementation, each kingdom works together to optimize a separate criteria of the nash equilbrium. Each subpopulation within the kingdom simplify ensures genetic diversity is maintained. Migration occurs interkingdom in a ring between islands in that kingdom and intrakingdom in a ring between islands of the same cross-section of different kingdoms. Note that since our genome is represent as a floating point number, we can atomically mutate it in global memory and thus can asynchronously migrate genomes between islands without corruption.

Our genetic algorithm framework does not include a `selection` operator as is typical. This is because we take advantage of the parrallel nature to implement a binary reduction that computes the individual with the greatest fitness. During the reduction, we compare two genes and elimite the less fit one and replace it with a copy of the more fit one. In the process, it wipes out individuals with the worst fitness---approximately 50% of the population depending on the distribution of individuals within the islands. Though it seems odd to elimiate individuals of the population based partially on placement, it allows us to optimize the reduction to perform very quickly.

After evaluation, selection, and migration, we cross and mutate the individuals before repeating the process. This evolution loop will continue until we reach the desired number of iterations and then will stop. The kernel will finish once all populations have been evolved to the desired number of iterations. Note that there is no syncronization between each island by design as to ensure speedy performance.

### Multi-objective Optimization

The multi-objective optimization layer is written on top of the genetic algorithm framework. For reasons we will discuss later, it's API is exposed as a macro with a special syntax for specifying the optimization problem. Specifically, it is implemented to compute the Nash equilbirum of a set of `N` equations each with `N` arguments. Each player's controls the `i`th argument and uses the `i`th equation as its fitness function. Here is the encoding of the tarrif problem described above:

```c
    float optimized_arguments[2];
    float optimized_fitness[2];
    optimize(10000000, optimized_arguments, optimized_fitness, {
        func(0, args[0] * (args[0] - args[1] - 2));
        func(1, args[1] * (args[1] - args[0] - 8));
    });
```

The optimize macro takes a few parameters. First, the number of generations to evolve the population before returning the result. Next, statically allocated arrays that will, after running, contain the optimized arguments and the resulting optimized fitnesses. Note that it determines the number of parameters from the length of these arrays, and it is undefined behavior to have a different number of functions than should be expected by the length of the arrays. Finally, the last argument is a domain-specific language that describes each function. The syntax must use the `func` macro, which takes the function index as the first argument and its defintion as the second argument. Note that the second argument may use the implicitly defined `args` array to obtain the arguments of the function. We will discuss why this particular syntax was used later in the report.

Once run, this will call the kernel and wait for the specified number of generations to complete before copying the result into the statically allocated arrays.

## Results

As previously discussed, the analytical solution to this problem is `(4, 6)`. Let's compare that to the CPU and GPU computed result. We will run both the CPU and GPU code with `island_count_per_kingdom = 10`, `island_population = 500`, and one thousand iterations.

| Hardware | Optimized Arguments  | Optimized Fitness        | Runtime  |
|----------|----------------------|--------------------------|----------|
| CPU      | (3.998751, 6.000210) | {-16.000841, -35.992508} | 2,540 ms |
| GPU      | (5.818570, 8.769202) | {-28.805599, -44.278927} |    83 ms |

Clearly the GPU run much more quickly but the produced less accurate results. We will increase the number of iterations to one-hundred thousand iterations and compare again.

| Hardware | Optimized Arguments  | Optimized Fitness        | Runtime    |
|----------|----------------------|--------------------------|------------|
| CPU      | (3.999207, 5.998609) | {-15.994436, -35.995243} | 249,000 ms |
| GPU      | (5.045824, 6.620116) | {-18.035248, -42.538933} |     737 ms |

With an increased number of iterations, we see that the GPU result becomes much closer to the analytical solution and still runs very, very quickly---much faster than the CPU runtime with 100x less iterations. Also, it is clear that the CPU algorithm becomes infeasible to run as the number of generations increase, a compelling argument for GPU parallelization of this sort of problem. Since genetic algorithms require simulating many individuals at once, all doing the same thing, we see immense speedups between the CPU and GPU version.

### Accuracy

It's important to recognize the much lower accuracy of the GPU output compared to the CPU output. This is something that is not fully understood, and would benefit from greater research. One potential contributing factor is quality of random number generation. The CUDA documentation warns that generating random numbers with separate `curandState_t`s on each thread could potentially result in correlated random numbers. It is unclear if that might be the cause, but that is definitely something that ought to be looked into. If this were to be solved, the GPU impelementation would be clearly preferable for any sort of parallelizable genetic algorithm.

## Challenges

### Algorithmic Design

TODO

### Experimental CUDA Device Lambda

TODO (macros, lack of objects, lack of structs, lack of types)

Blah [[1]](#references)

### Template Metaprogramming

TODO (header files)

## Future Directions

TODO: Talk about seeding
TODO: Talk about selection
TODO: Talk about generalizing more
TODO: Talk about stopping conditions and making sure an island doesn't finish way before the others
TODO: Talk about focused on CUDA lambdas and parallelization
TODO: Talk about mutation, cross, etc.
TODO: Talk about constraints, maximization, etc.

## Installation and Usage

## Performance Analysis

### Nash Equilibrium Computation

TODO (results too)

## Conclusion

TODO

## References

1. [Nash Genetic Algorithms : examples and applications](http://ieeexplore.ieee.org/xpls/abs_all.jsp?arnumber=870339)
2. [Examples and exercises on finding Nash equilibria of two-player games using best response functions](https://www.economics.utoronto.ca/osborne/2x3/tutorial/NEIEX.HTM)
3. [GPU-based Acceleration of the Genetic Algorithm](http://www.gpgpgpu.com/gecco2009/7.pdf)
4. [Co-Evolvability of Games in Coevolutionary Genetic Algorithms](http://delivery.acm.org/10.1145/1580000/1570208/p1869-lin.pdf?ip=131.215.158.223&id=1570208&acc=ACTIVE%20SERVICE&key=1FCBABC0A756505B%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35&CFID=626681181&CFTOKEN=86827508&__acm__=1465302757_bd6a01c0e44c7266dd7ac4a5f67008b7)
