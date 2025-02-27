# Examples

The full code of the examples described here is available at the [examples](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/) directory. 

## Mean difference of coordinates 

Computing the mean difference in `x` position between random particles. The closure is used to remove the indexes and the distance of the particles from the parameters of the input function, as they are not needed in this case.

```julia
using CellListMap

# System properties
N = 100_000
sides = [250,250,250]
cutoff = 10

# Particle positions
x = [ sides .* rand(3) for i in 1:N ]

# Initialize linked lists and box structures
box = Box(sides,cutoff)
cl = CellList(x,box)

# Function to be evaluated from positions 
f(x,y,sum_dx) = sum_dx + abs(x[1] - y[1])
normalization = N / (N*(N-1)/2) # (number of particles) / (number of pairs)

# Run calculation (0.0 is the initial value)
avg_dx = normalization * map_pairwise(
    (x,y,i,j,d2,sum_dx) -> f(x,y,sum_dx), 0.0, box, cl 
)
```

The example above can be run with `CellListMap.Examples.average_displacement()` and is available in the
[average_displacement.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/average_displacement.jl) file.

## Histogram of distances

Computing the histogram of the distances between particles (considering the same particles as in the above example). Again,
we use a closure to remove the positions and indexes of the particles from the function arguments, because they are not needed. The distance, on the other side, is needed in this example:

```julia
# Function that accumulates the histogram of distances
function build_histogram!(d2,hist)
    d = sqrt(d2)
    ibin = floor(Int,d) + 1
    hist[ibin] += 1
    return hist
end;

# Initialize (and preallocate) the histogram
hist = zeros(Int,10);

# Run calculation
map_pairwise!(
    (x,y,i,j,d2,hist) -> build_histogram!(d2,hist),
    hist,box,cl
)
```
Note that, since `hist` is mutable, there is no need to assign the output of `map_pairwise!` to it. 

The example above can be run with `CellListMap.Examples.distance_histogram()` and is available in the
[distance_histogram.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/distance_histogram.jl) file.

## Gravitational potential

In this test we compute the "gravitational potential", assigning to each particle a different mass. In this case, the closure is used to pass the masses to the function that computes the potential.

```julia
# masses
const mass = rand(N)

# Function to be evaluated for each pair 
function potential(i,j,d2,mass,u)
    d = sqrt(d2)
    u = u - 9.8*mass[i]*mass[j]/d
    return u
end

# Run pairwise computation
u = map_pairwise((x,y,i,j,d2,u) -> potential(i,j,d2,mass,u),0.0,box,cl)
```

The example above can be run with `CellListMap.Examples.gravitational_potential()` and is available in the
[gravitational_potential.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/gravitational_potential.jl) file.

## Gravitational force

In the following example, we update a force vector of for all particles.

```julia
# masses
const mass = rand(N)

# Function to be evaluated for each pair: update force vector
function calc_forces!(x,y,i,j,d2,mass,forces)
    G = 9.8*mass[i]*mass[j]/d2
    d = sqrt(d2)
    df = (G/d)*(x - y)
    forces[i] = forces[i] - df
    forces[j] = forces[j] + df
    return forces
end

# Initialize and preallocate forces
forces = [ zeros(SVector{3,Float64}) for i in 1:N ]

# Run pairwise computation
map_pairwise!(
    (x,y,i,j,d2,forces) -> calc_forces!(x,y,i,j,d2,mass,forces),
    forces,box,cl
)

```

The example above can be run with `CellListMap.Examples.gravitational_force()` and is available in the
[gravitational_force.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/gravitational_force.jl) file.

!!! note
    The parallelization works by splitting the `forces` vector in as many tasks as necessary, and each task will update an independent `forces` array, which will be reduced at the end. Therefore, there is no need to deal with atomic operations or blocks in the `calc_forces!` function above for the update of `forces`, which is implemented as if the code was running serially. The same applies to other examples in this section.

## Nearest neighbor

Here we compute the indexes of the particles that satisfy the minimum distance between two sets of points, using the linked lists. The distance and the indexes are stored in a tuple, and a reducing method has to be defined for that tuple to run the calculation.  The function does not need the coordinates of the points, only their distance and indexes.

```julia
# Number of particles, sides and cutoff
N1=1_500
N2=1_500_000
sides = [250,250,250]
cutoff = 10.
box = Box(sides,cutoff)

# Particle positions
x = [ SVector{3,Float64}(sides .* rand(3)) for i in 1:N1 ]
y = [ SVector{3,Float64}(sides .* rand(3)) for i in 1:N2 ]

# Initialize auxiliary linked lists
cl = CellList(x,y,box)

# Function that keeps the minimum distance
f(i,j,d2,mind) = d2 < mind[3] ? (i,j,d2) : mind

# We have to define our own reduce function here
function reduce_mind(output,output_threaded)
    mind = output_threaded[1]
    for i in 2:length(output_threaded)
        if output_threaded[i][3] < mind[3]
            mind = output_threaded[i]
        end
    end
    return mind
end

# Initial value
mind = ( 0, 0, +Inf )

# Run pairwise computation
mind = map_pairwise( 
    (x,y,i,j,d2,mind) -> f(i,j,d2,mind),
    mind,box,cl;reduce=reduce_mind
)
```

The example above can be run with `CellListMap.Examples.nearest_neighbor()` and is available in the
[nearest_neighbor.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/nearest_neighbor.jl) file.

The example `CellListMap.Examples.nearest_neighbor_nopbc()` of [nearest\_neighbor\_nopbc.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/nearest_neighbor_nopbc.jl) describes a similar problem but *without* periodic boundary conditions. Depending on the distribution of points and size it is a faster method than usual ball-tree methods. 

## Neighbor lists

### The `CellListMap.neighborlist` function
The package provides a `neighborlist` function that implements this calculation. Without periodic boundary conditions, just do:

```julia-repl
julia> x = [ rand(3) for _ in 1:10_000 ];

julia> CellListMap.neighborlist(x,0.05)
24778-element Vector{Tuple{Int64, Int64, Float64}}:
 (1, 62, 0.028481068525796384)
 ⋮
 (9954, 1749, 0.04887502372299809)
 (9974, 124, 0.040110356034451795)
```
or `CellListMap.neighborlist(x,y,r)` for computing the lists of pairs of two sets closer than `r`.

The returning array contains tuples with the index of the particle in the first vector, the index of the particle in the second vector, and their distance.

If periodic boundary conditions are used, the `Box` and `CellList` must be constructed in advance:
```julia-repl
julia> x = [ rand(3) for _ in 1:10_000 ]; 

julia> box = Box([1,1,1],0.1);

julia> cl = CellList(x,box);

julia> CellListMap.neighborlist(box,cl)

julia> CellListMap.neighborlist(box,cl)
209506-element Vector{Tuple{Int64, Int64, Float64}}:
 (1, 121, 0.05553035041478053)
 (1, 1589, 0.051415489701932444)
 ⋮
 (7469, 7946, 0.09760096646331885)
```

### Implementation

The implementation of the above function follows the principles below. 
 The empty `pairs` output array will be split in one vector for each thread, and reduced with a custom reduction function. 

```julia
# Function to be evaluated for each pair: push pair
function push_pair!(i,j,d2,pairs)
    d = sqrt(d2)
    push!(pairs,(i,j,d))
    return pairs
end

# Reduction function
function reduce_pairs(pairs,pairs_threaded)
    for i in eachindex(pairs_threaded)
        append!(pairs,pairs_threaded[i])
    end
    return pairs
end

# Initialize output
pairs = Tuple{Int,Int,Float64}[]

# Run pairwise computation
map_pairwise!(
    (x,y,i,j,d2,pairs) -> push_pair!(i,j,d2,pairs),
    pairs,box,cl,
    reduce=reduce_pairs
)
```

The full example can be run with `CellListMap.Examples.neighborlist()`, available in the file 
[neighborlist.jl](https://github.com/m3g/CellListMap.jl/blob/main/src/examples/neighborlist.jl).
