# DiffEqOperators.jl

[![Build Status](https://github.com/SciML/DiffEqOperators.jl/workflows/CI/badge.svg)](https://github.com/SciML/DiffEqOperators.jl/actions?query=workflow%3ACI)
[![GitlabCI](https://gitlab.com/JuliaGPU/DiffEqOperators.jl/badges/master/pipeline.svg)](https://gitlab.com/JuliaGPU/DiffEqOperators.jl/pipelines)
[![Stable](https://img.shields.io/badge/docs-stable-blue.svg)](http://diffeqoperators.sciml.ai/stable/)
[![Dev](https://img.shields.io/badge/docs-dev-blue.svg)](http://diffeqoperators.sciml.ai/dev/)

DiffEqOperators.jl is a package for finite difference discretization of partial
differential equations. It serves two purposes:

1. Building fast lazy operators for high order non-uniform finite differences.
2. Automated finite difference discretization of symbolically-defined PDEs.

#### Note: (2) is still a work in progress!

For the operators, both centered and
[upwind](https://en.wikipedia.org/wiki/Upwind_scheme) operators are provided,
for domains of any dimension, arbitrarily spaced grids, and for any order of accuracy.
The cases of 1, 2, and 3 dimensions with an evenly spaced grid are optimized with a
convolution routine from `NNlib.jl`. Care is taken to give efficiency by avoiding
unnecessary allocations, using purpose-built stencil compilers, allowing GPUs
and parallelism, etc. Any operator can be concretized as an `Array`, a
`BandedMatrix` or a sparse matrix.

## Documentation

For information on using the package,
[see the stable documentation](https://diffeqoperators.sciml.ai/stable/). Use the
[in-development documentation](https://diffeqoperators.sciml.ai/dev/) for the version of
the documentation which contains the unreleased features.

## Example 1: Automated Finite Difference Solution to the Heat Equation

```julia
using OrdinaryDiffEq, ModelingToolkit, DiffEqOperators

# Parameters, variables, and derivatives
@parameters t x
@variables u(..)
@derivatives Dt'~t
@derivatives Dxx''~x

# 1D PDE and boundary conditions
eq  = Dt(u(t,x)) ~ Dxx(u(t,x))
bcs = [u(0,x) ~ cos(x),
       u(t,0) ~ exp(-t),
       u(t,Float64(pi)) ~ -exp(-t)]

# Space and time domains
domains = [t ∈ IntervalDomain(0.0,1.0),
           x ∈ IntervalDomain(0.0,Float64(pi))]

# PDE system
pdesys = PDESystem(eq,bcs,domains,[t,x],[u])

# Method of lines discretization
dx = 0.1
order = 2
discretization = MOLFiniteDifference(dx,order)

# Convert the PDE problem into an ODE problem
prob = discretize(pdesys,discretization)

# Solve ODE problem
sol = solve(prob,Tsit5(),saveat=0.1)
```

## Example 2: Finite Difference Operator Solution for the Heat Equation

```julia
using DiffEqOperators, OrdinaryDiffEq

nknots = 100
h = 1.0/(nknots+1)
knots = range(h, step=h, length=nknots)
ord_deriv = 2
ord_approx = 2

const Δ = CenteredDifference(ord_deriv, ord_approx, h, nknots)
const bc = Dirichlet0BC(Float64)

t0 = 0.0
t1 = 0.03
u0 = u_analytic.(knots, t0)

step(u,p,t) = Δ*bc*u
prob = ODEProblem(step, u0, (t0, t1))
alg = KenCarp4()
sol = solve(prob, alg)
```
