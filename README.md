# ECON 5010 Problem Set 1 — Question 9

## Performance Optimization Exercise

For Question 9, I forked my partner's repository, cloned the fork using GitHub Desktop, and opened `perf_exercise.jl` in VS Code.

The goal of this exercise was to improve the performance of the Julia code without changing what the functions compute. I used `@time`, `@allocated`, and `@code_warntype` to identify performance problems and compare the code before and after optimization.

---

## 1. Baseline

Before making any changes, I ran the original program twice using:

`@time main()`

I used the second run as the main baseline because the first run may contain additional Julia compilation overhead.

### Baseline result

- Time: 0.160448 seconds
- Allocations: 2.01 million
- Memory allocated: 137.604 MiB
- GC time: 61.14%

The original program ran correctly, so I used its outputs as the reference when checking the optimized version.

---

## 2. `compute_stats`

### Before optimization

I profiled the original function using:

`@time compute_stats()`

`@allocated compute_stats()`

`@code_warntype compute_stats()`

Results:

- Time: 0.004782 seconds
- Allocations: 7
- Memory allocated: 208 bytes
- Return type: `Vector{Any}`

### Observations

`@code_warntype` showed that:

- `results` had type `Vector{Any}`
- The function body also returned `Vector{Any}`
- The statistical operations using the global variable `data` were inferred as `Any`

The original function used a non-constant global variable and created its result using:

```julia
results = []

This created an array without a concrete element type and caused type instability.

Optimization

I changed the function so that data is passed as an argument and the result vector has a concrete Float64 element type.

function compute_stats(data)
    results = Float64[
        sum(data),
        mean(data),
        maximum(data),
        minimum(data),
        std(data)
    ]

    return results
end

I also changed the call inside main() from:

stats = compute_stats()

to:

stats = compute_stats(data)
After optimization

I profiled the new version using:

@time compute_stats(data)

@allocated compute_stats(data)

@code_warntype compute_stats(data)

Results:

Time: 0.006781 seconds
Allocations: 2
Memory allocated: 96 bytes
Return type: Vector{Float64}

The number of allocations decreased from 7 to 2, and allocated memory decreased from 208 bytes to 96 bytes.

The single-run execution time was slightly higher in this particular measurement. Since this is a very small function, the timing is sensitive to measurement noise. The allocation results and type information are more informative here.

After optimization, @code_warntype showed:

data::Vector{Float64}
results::Vector{Float64}
Body::Vector{Float64}

The previous Any types disappeared, confirming that the function is now type stable.

3. monte_carlo_pi
Before optimization

I profiled the original function using:

@time monte_carlo_pi(1_000_000)

@allocated monte_carlo_pi(1_000_000)

@code_warntype monte_carlo_pi(1_000_000)

Results:

Time: 0.025436 seconds
Allocations: about 2.00 million
Memory allocated: 76.294 MiB
GC time: 35.53%
@allocated: 80,000,000 bytes
Observations

The function itself was type stable, so type instability was not the main problem.

The main problem was this line inside the loop:

point = [rand(), rand()]

A new two-element Vector{Float64} was created during every iteration.

Since the loop runs 1,000,000 times, this created a very large number of temporary arrays and caused substantial memory allocation and garbage-collection overhead.

Optimization

I replaced the temporary vector with two scalar variables.

function monte_carlo_pi(n)
    count = 0

    for i in 1:n
        x = rand()
        y = rand()

        if x^2 + y^2 <= 1.0
            count += 1
        end
    end

    return 4 * count / n
end
After optimization

Results:

Time: 0.002249 seconds
Allocations: 0
Memory allocated: 0 bytes

@code_warntype showed:

x::Float64
y::Float64
Body::Float64

The temporary Vector{Float64} disappeared.

This optimization eliminated the unnecessary allocations inside the loop and greatly reduced execution time.

4. row_sums
Before optimization

I tested the function using a 2000 × 2000 matrix:

A_test = rand(2000, 2000)

I then used:

@time row_sums(A_test)

@allocated row_sums(A_test)

@code_warntype row_sums(A_test)

Results:

Time: 0.018796 seconds
Allocations: about 8.01 thousand
Memory allocated: 30.782 MiB
@allocated: 32,277,173 bytes
Return type: Vector{Any}
Observations

The original implementation had three performance problems:

sums = []

created an output vector without a concrete element type.

row = A[i, :]

created a copy of each row.

The function also used push! to grow the result vector one element at a time even though the final number of rows was already known.

Optimization

I preallocated the result vector and used a view instead of copying each row.

function row_sums(A)
    n = size(A, 1)
    sums = zeros(eltype(A), n)

    for i in 1:n
        row = @view A[i, :]
        sums[i] = sum(row)
    end

    return sums
end
After optimization

Results:

Time: 0.013657 seconds
Allocations: 3
Memory allocated: 15.726 KiB
@allocated: 16,087 bytes
Return type: Vector{Float64}

@code_warntype showed:

sums::Vector{Float64}
row::SubArray{Float64, ...}
Body::Vector{Float64}

The SubArray confirms that @view avoids copying the row.

This reduced memory allocation from about 30.8 MiB to about 15.7 KiB.

5. build_report
Before optimization

I used:

labels_test = ["sum", "mean", "max", "min", "std"]
values_test = [1.0, 2.0, 3.0, 4.0, 5.0]

and profiled the function using:

@time build_report(labels_test, values_test)

@allocated build_report(labels_test, values_test)

@code_warntype build_report(labels_test, values_test)

Results:

Time: 0.000014 seconds
Allocations: 20
Memory allocated: 2.344 KiB
@allocated: 2,400 bytes

The function was already type stable.

Attempted optimization

The original function repeatedly concatenated strings:

report = report * labels[i] * ": " * string(values[i]) * "\n"

Since strings are immutable, I tested an alternative using IOBuffer:

function build_report(labels, values)
    io = IOBuffer()

    for i in eachindex(labels, values)
        println(io, labels[i], ": ", values[i])
    end

    return String(take!(io))
end
Result of the attempted optimization

The IOBuffer version produced:

Time: 0.000023 seconds
Allocations: 26
Memory allocated: 2.516 KiB
@allocated: 2,576 bytes

For the five-line report used in this exercise, the IOBuffer version was slightly slower and allocated slightly more memory.

Therefore, I reverted to the original implementation.

This experiment showed that an optimization that may be useful for large strings does not necessarily improve performance for a very small workload. Benchmarking was necessary to determine which implementation was actually better for this program.

6. unstable_sum
Before optimization

I profiled the original function using:

@time unstable_sum(data)

@allocated unstable_sum(data)

@code_warntype unstable_sum(data)

Results:

Time: 0.009017 seconds
Allocations: 1
Memory allocated: 16 bytes
@allocated: 16 bytes
Return type: Union{Float64, Int64}
Observations

The original function initialized the accumulator using:

total = 0

This made total initially an Int64.

However, the elements in data are Float64, so after adding a value from the vector, total could become a Float64.

@code_warntype therefore showed:

total::Union{Float64, Int64}
Body::Union{Float64, Int64}

This indicated type instability.

Optimization

I initialized the accumulator as a floating-point value instead:

function unstable_sum(xs)
    total = 0.0

    for x in xs
        if x > 0.5
            total += x
        end
    end

    return total
end
After optimization

Results:

Time: 0.002472 seconds
Allocations: 1
Memory allocated: 16 bytes
Return type: Float64

The output remained unchanged for the same input data.

After optimization, @code_warntype showed:

total::Float64
Body::Float64

The previous Union{Float64, Int64} disappeared, confirming that the function is now type stable.

7. Final Performance

After completing the optimizations, I ran the full program again using:

@time main()

I ran it more than once and used the later run for comparison.

Final result
Time: 0.038188 seconds
Allocations: 126
Memory allocated: 30.540 MiB
GC time: 15.04%
Baseline vs. optimized version
Metric	Baseline	Optimized
Time	0.160448 s	0.038188 s
Allocations	2.01 million	126
Memory allocated	137.604 MiB	30.540 MiB
GC time	61.14%	15.04%

The optimized program ran approximately 4.2 times faster than the baseline.

The execution time decreased by about 76%, memory allocation decreased by about 78%, and the number of allocations decreased from about 2.01 million to only 126.

The remaining memory allocation is partly due to the 2000 × 2000 random matrix created inside main().

Overall, the largest performance improvements came from:

eliminating temporary allocations inside monte_carlo_pi
avoiding row copies and preallocating output in row_sums
improving type stability in compute_stats
improving type stability in unstable_sum

The build_report experiment also showed the importance of benchmarking proposed optimizations rather than assuming that a different implementation will always be faster.