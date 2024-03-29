```@meta
CurrentModule = ScenTrees
```

# Kernel Density Estimation

This is a non-parametric technique for generating trajectories used to generate and improve scenario trees and scenario lattices in the stochastic approximation algorithm. This is a case where the distributions are not available explicitly and so the samples can be drawn from a data by proper approximations. It involves two steps:

    1. Density estimation and,
    2. Stochastic approximation process.

The stochastic approximation process has already been described above so we will concentrate on the density estimation step. The density estimation step involves two steps:

    (a) Estimation of the distribution of conditional densities and,
    (b) Composition method

We estimate the distribution of transition densities given the history of the process. This distribution is estimated by the non-parametric kernel density estimation. The density at stage ``t+1`` conditional on the history ``\mathbf{x_{t}} = (x_0,x_{i_{1}},x_{i_{2}},\ldots,x_{i_{t}})`` is estimated by
```math
\hat{f}_{t+1}(x_{t+1}|\mathbf{x_{t}}) = \sum_{j=1}^{N} w_j(\mathbf{x_{t}}) \cdot k_{h_N}(x_{t+1} - \xi_{j,t+1})
```
where ``k(.)`` is a kernel function, ``h_N`` is the bandwidth and ``k_{h_N}`` is the weighted kernel function.

The weights are given by
```math
w_j(\mathbf{x_{t}}) = \frac{k\big(\frac{x_{i_1} - \xi_{j,1}}{h_N}\big)\cdot \ldots \cdot k\big(\frac{x_{i_t} - \xi_{j,t}}{h_N}\big)}{\sum_{\ell=1}^{N}k\big(\frac{x_{i_1} - \xi_{l,1}}{h_N}\big)\cdot \ldots \cdot k\big(\frac{x_{i_t} - \xi_{l,t}}{h_N}\big)}.
```
Notice from above that the weights depend on the history ``\mathbf{x_{t}}`` up to stage ``t`` only. Also, the weights sum up to 1 for every ``\mathbf{x_{t}}$, i.e., $\sum_{j=1}^{N} w_j(\mathbf{x_{t}}) = 1``.

We use the composition method to find the samples from the conditional distribution ``\hat{f}_{t+1}(.|\mathbf{x_{t}})`` as follows. Pick a random number ``U \in (0,1)`` where ``U`` is uniformly distributed then find a summation index $j^{\star} \in {1,2,\ldots,N}$ such that
``\sum_{j=1}^{j^{\star}-1} w_j(\mathbf{x_{t}}) < U \leq \sum_{j=1}^{j^{\star}} w_j(\mathbf{x_{t}}).``

The cumulative sum of weights leads to a high probability of picking a data path near the observation.

The value of the data ``\xi_t`` is obtained by setting the value at stage ``t`` to
```math
\xi_t = X_{j^{\star,t}} + \text{rand}_{k_{h_N}}
```
where ``\text{rand}_{k_{h_N}}`` is a random value sampled from the kernel estimator using the composition method.

The generated data point is according to the distribution of the density at the current stage and dependent on the history of all the data points. It has been shown that the choice of the kernel does not have an important effect on density estimation. Hence, we employ the following logistic kernel as default: ``k(x) = \frac{1}{(e^x + e^{-x})^2}.``

One of the most important factor to consider in density estimation is the bandwidth. We use the Silverman's rule of thumb to obtain the bandwidths for each of the data columns as follows ``h_N = \sigma(X_t)\cdot N^{\frac{-1}{N+4}}``
where ``N`` is the number of available trajectories and ``\sigma(X_t)`` is the standard deviation of data in stage ``t``.

Using the above procedure, every new sample path starts with ``\mathbf{\tilde{\xi_1}} := (x_1)``. Using the composition method, we find a new sample ``\tilde{\xi_2}`` from ``\hat{f}_1(.|\mathbf{\tilde{\xi_1}})`` at the first stage. Next, we set ``\mathbf{\tilde{\xi_2}} = (\tilde{\xi_1},\tilde{\xi_2})`` and generate a new data point ``\tilde{\xi_2}`` from ``\hat{f}_2(.|\mathbf{\tilde{\xi_1}})``  at the second stage. The process continues in that manner until the final stage ``T`` where we get the final new sample path ``\mathbf{\tilde{\xi_T}} = (\tilde{\xi_0},\tilde{\xi_1},\ldots,\tilde{\xi_T})`` which is generated form the initial data ``\mathbf{x_{t}}``.

The new sample path ``\mathbf{\tilde{\xi_T}}`` is what we will feed in the stochastic approximation algorithm to generate either a scenario tree or a scenario lattice.

## Implementation of the above process

The above process is implemented in our library in `KernelDensityEstimation.jl`.In this script, we use the concept of function closures which we assume that the user of this library is aware of. The user is required to provide a data in 2 dimension, i.e., a matrix of Float64 or Int64, to this function and also the distribution of the kernel that he/she wants to use. The default kernel distribution is the Logistic kernel. The kernel distribution should conform to the distributions stated in [Distributions.jl](https://github.com/JuliaStats/Distributions.jl) library.

Most of the times when you load the data into Julia, it recognizes it as a `DataFrame` type. But we have wrote the function in a manner that the user should provide a Matrix of either floats or integers. In the following procedure, we use the function `Matrix` to convert the loaded dataframe into a matrix which is the right type of input into the function.

The function `KernelScenarios(data::Union{Array{Int64,2},Array{Float64,2}}, kernelDistribution = Logistic; Markovian = true)` takes a ``(N \times T)`` dimensional data and the distribution of the kernel you want to employ. The $N$ rows of the data are the number of trajectories in the initial data and the $T$ columns is the number of stages in in each trajectory of the data. We also let the user to specify whether the process he/she is creating is a Markovian process or not. We are aware that scenario lattices are natural discretization of Markovian process. Hence the user should create samples for a scenario tree for the condition `Markovian=false` and scenario lattices for the condition `Markovian = true`.

Since `TreeApproximation!` and `LatticeApproximation` function needs a process function for generating samples that doesn't take any inputs, we employ the concept of function closures inside the above function. The function `KernelScenarios(data,kernelDistribution;Markovian=true)` is a getfield type of a function closure and so is sufficient to be a function required for stochastic approximation process.

To confirm the above statement, consider a ``1000x5`` dimsneional data from random walk. What is important to be said is that we use the package [`CSV`.jl](https://github.com/JuliaData/CSV.jl) to read the data into Julia and since we need the data in matrix form, we use the function `Matrix` from the package [`DataFrames.jl`](https://github.com/JuliaData/DataFrames.jl) to convert the dataframe into an array in two dimension which is then the input of our function.

```julia
julia> using ScenTrees, CSV
julia> data = CSV.read(".../RandomDataWalk.csv")
julia> Rdw = Matrix(data)
julia> Kdt = KernelScenarios(Rwd,Logistic;Markovian=true)
(::getfield(ScenTrees, Symbol("#closure#52")){Array{Float64,2},Int64,Int64,Array{Float64,1},Array{Float64,1},Array{Float64,1}}) (generic function with 1 method)
julia> ExampleTraj = KernelScenarios(Rwd,Logistic;Markovian = true)()
5-element Array{Float64,1}:
  2.9313
 -2.0964
  3.7671
  2.1476
  0.9424
```
As in `ExampleTraj` above, this function returns is a new sample according to the distribution of the density at the current stage and dependent on the history of all the data points.

We use the above data to approximate a scenario lattice in `5` stages with a branching structure of ``(1\times 3\times 4\times 5\times6)``  and ``100,000`` number of iterations as follows:

```julia
julia> KernExample = LatticeApproximation([1,3,4,5,6],KernelScenarios(Rwd,Logistic;Markovian=true),100000);
julia> PlotLattice(KernExample)
```
The above plot function gives the following plot result:

![Scenario Lattice From Kernel Trajectories](../assets/KernLattice.png)

To generate a scenario tree using the same process, we use the following procedure:

```julia
julia> KernTree = TreeApproximation!(Tree([1,2,2,2],1),KernelScenarios(gsdata,Logistic;Markovian=false),100000,2,2);
julia> treeplot(KernTree)
```
![Scenario Tree From Kernel Trajectories](../assets/kerneltree.png)
