# CS389: HW1
## Benchmarking the memory hierarchy
### Part 1: Benchmarking

**Important: compile with -O3 flag.**
```c++
#include <iostream>
#include <fstream>
#include <random>

int main() {
    const double memory_reads = 1000000000;  // Times to read memory at a single take
    const double times_repeated = 20;  // Times to repeat a take at a specific memory size
    const int n_from = 0;  // Size of memory (in power of 2) at which we start measuring.
    const int n_to = 16;  // Size of memory (in power of 2) at which we stop measuring.
    const int size = 1024 * 1048576;  // Size of array allocated from which we are going to read.
    const int n_of_ints = size / sizeof(int);  // Size of array in int size.
    const int jump_from = 16;
    const int jump_to = 128;
    const int mult_from = 1;
    const int mult_to = 1048576;

    std::random_device rand_dev;
    std::mt19937 generator(rand_dev());
    std::uniform_int_distribution<int> distr_filler(0, 1000000);
    std::chrono::steady_clock::time_point start, end;  // start and end are the time points at which we start and end one take.

    int i, j, k, size_number, mult, jump, indexed_size_int;  // i, j, k are indices, size_number is the size of a specific memory step in KiB.
    double time_per_read, elapsed_seconds, iterated_time;  // time_per_read is self explanatory,
                                                           // elapsed_seconds is the total time of the repeated takes, iterated_time is the time of a single take.
    
    int *allocated_memory = new int[n_of_ints];  // initializes array of size 'size'
    std::fill_n(allocated_memory, n_of_ints, distr_filler(generator));  // fills array with random integers

    // Creates arrays with memory sizes from 'n_from' to 'n_to'.
    int sizes[(n_to + 1)];
    for (i = n_from; i <= n_to; i++) {
        sizes[i] = int((pow(2.0, i) * 1024) - 1);
    }

    // Prepares results.csv file to save results.
    std::ofstream results;
    results.open("results.csv");

    /*
     * There are three for loops, the first one iterates through the different memory sizes we want to try.
     * The second for loop repeats the takes 'times_repeated' times to then get the mean.
     * The third loop repeats the memory reads 'memory_reads' times. Results are stored in iterated_time for each take, then to get average.
     * The memory read is done by indexing every 'jump' integers in a limited part of the array of specific 'indexed_size_int' size
     * and performing an operation on them, a multiplication by a random integer.
     */
    for (i = n_from; i <= n_to; i++) {
        elapsed_seconds = 0;
        indexed_size_int = sizes[i] / sizeof(int);
        for (j = 0; j < times_repeated; j++) {
            std::uniform_int_distribution<int> distr_jump(jump_from, jump_to);
            std::uniform_int_distribution<int> distr_mult(mult_from, mult_to);
            jump = distr_jump(generator);
            mult = distr_mult(generator);

            start = std::chrono::steady_clock::now();  // Start counting time
            for (k = 0; k < memory_reads; k++) {  // Repeat memory read 'memory_reads' times.
                allocated_memory[(k * jump) & indexed_size_int] *= mult;  // Iterates and multiplies the jump-th element by a random 'mult' int through
                                                                          // the first part of size 'indexed_size_int' of the array. It iterates through
                                                                          // a the bitwise and operator that acts like a mod but doesn't take that much time.
                                                                          // In this way, it's going over and over through the first elements for each size,
                                                                          // only accessing a 'indexed_size_int' part of that array.
            }
            end = std::chrono::steady_clock::now();  // Stop counting time

            iterated_time =  // gets time in a single take ('memory_reads' memory reads) and divides it by the number of 'memory reads' to ge the mean
                    (std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count()) / memory_reads;
            elapsed_seconds = elapsed_seconds +
                              iterated_time;  // adds the time of a single take to the total time of takes of a specific memory size.
        }
        size_number = (sizes[i] + 1) / 1024;
        time_per_read = elapsed_seconds / times_repeated;  // Calculates mean of takes
        printf("%d, %f \n", size_number, time_per_read);  // Prints results to terminal
        results << size_number << ", " << time_per_read << std::endl;  // Saves result to 'results.csv'
    }
    results.close();  // Closes results.csv file
    system("gnuplot -p -e \"; set autoscale; set title 'cs389.hw1: Benchmarking the memory hierarchy'; set terminal qt; set xlabel 'Buffer size (KiB)'; set ylabel 'Latency per memory read (ns)'; set logscale x 2; plot 'results.csv' w linespoints lc rgb 'red'\"");
    delete[] allocated_memory;  // Deletes array
    return 0;
}
```

### Part 2: Graphing
![Graph](https://github.com/daherrero/cs389-hw1/blob/master/results4.svg)

Size (KiB) | Latency (ns)
-----------|-------------
1    |0.405008
2    |0.406193
4    |0.402897
8    |0.407586
16   |0.402482
32   |0.403377
64   |0.673096
128  |0.873224
256  |0.90853
512  |1.39748
1024 |1.56757
2048 |1.59204
4096 |1.59708
8192 |3.91962
16384|8.02413
32768|9.33662
65536|9.09222
### Part 3: Analysis

1. From size **1** to size **64** it looks like it is only using the **L1 cache** as the read time are less than 1 nanosecond. 
From size **64** to size **256** it may be fetching from both **L1 and L2 caches**, as the speeds are not constant and increases with the size of the memory the program accesses. 
Therefore the **L1 cache** is smaller than **256 KiB**. The **L2 cache** looks like it goes from **256 KiB** to around **4096 KiB**
And next big jump appears right after the **4096 KiB** mark and ends around the **16384 KiB** mark, so the **L3 cache** should end in that range, around the **8192 KiB** mark.
2. I get a reasonable speed for the L1 cache, but after that, the speeds are faster than they should be, this may be because the processor knows the pattern of accessing the memory more or less and it loads some stuff into L1 and therefore the speeds are significantly higher (2 ns vs 7 ns for **L2**).
3. My processor is an **Intel i7-7820HQ** and it is supposed to have a **256 KiB L1 cache**, a **1 MiB L2 cache** and a **8 MiB L3 cache**, so more or less it falls into my predictions. 
