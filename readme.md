# Q2.jl

This package serves as an interface between Julia and [q/kdb](https://code.kx.com/q/) (similar to [rkdb](https://github.com/KxSystems/rkdb)) so that commands and data can be sent to a remote kdb session and vice versa. The package serves as a replacement to the unmaintained and outdated [Q.jl](https://github.com/enlnt/Q.jl) with the main difference being fewer dependencies and a more minimal set of features, though with many elements ported over. It includes two parts. First is the Julia interface, which lets a Julia process connect to a q/kdb server and exchange data. The second is a kdb interface  `J.q` which allows julia to be run from within a kdb/q process.

Currently only Mac and Linux have been tested and are known to be supported.

## Julia Interface

The package exports the types `KDBConnection` and `KDBHandle`, as well as the functions `open`, `close!` and `execute`. Please see the function docstrings and examples below for more information.

### Installation
Just run `using Pkg; Pkg.add(Q2)` as you would normally.

### Examples

```julia
using Q2
conn = KDBConnection(host="localhost", port=1234) # alo supports password, etc.

# one method of opening connections
h = open(conn)
mytable = execute(h,"n:`int\$1e3;([]a:n?100;b:n?1000f;c:n?`4)")
mylist1 = execute(h, "1+til 10")
mylist2 = execute(h, "25?(0nf,10?1f)")
myatom1 = execute(h, ".z.P")
close!(h)

# another method of opening connections (opens and closes automatically)
tbl = DataFrame(x=randn(500),y=rand([:A,:B,:X],500))
open(conn) do h
    execute(h,"show",tbl) # send table to kdb + display it
    execute(h,"{til[10],x}",[1,2,3,:abcd,missing,nothing,[:paul,:john,:george,:ringo]])
end
```

### General Notes
 - Any null time or numeric value is converted to a `missing` when it is in an Atomic list. For example the list `0 2 3 0Nj` in q will converted to a `Union{Float64,Missing}[0,2,3,missing]`, however this conversion will not occur if this is a mixed list or an atom, since it is impossible to know the type of the `missing` value in these cases.
 - Unsupported kdb types for conversions: GUIDs, Functions


## kdb/q interface

Once installed (see below), run `\l J.q` to load the package. Julia session is started on package load.
This package contains the following functions:
 - `.J.e[x]` - executes a string `x` and shows result (never returns). 
 - `.J.er[x]` - executes a string `x` and return result as native `q` object (if possible. See table on conversions).
 - `.J.wrapfn[x]` - wraps a Julia function with name contained in string `x` as kdb function.
 - `.J.set[nm;x]` - takes a kdb object `x` and assigns it to variable named `nm` (string/sym) in the Julia global namespace.
 - `.J.repl[]` - launches the Julia REPL for current session. `ctrl+D` to close the REPL and go back to `q)`. Useful for debugging.

### Notes
The first available julia binary available in the `PATH` will be used. Requires `Q2.jl` to be available for full functionality, warning will be given if not available. The conversion of objects between q and julia are done using `Q2.jl`, so the same conversion rules apply (see type conversions section).
The following optional environment variables are read when loading the package
 - `EMBEDJL_CMD_ARGS` - string containing additional command arguments (example: `--threads auto --home=... -O 3`)
 - `EMBEDJL_RUN_CMD` - string containing commands to be run in julia directly after launch

### Installation

`cd` into the `<root>/embed/` folder and run `make all && make install` to install the package. This will install the package into your `$QHOME`.

### Examples

```
\l J.q

.J.e "v=[1,2,3,4,5]; @show v";                  / run but don't return
J) prod(1:10)                                   / does the same but without quotes or escapes
x:.J.er "5.*v";                                 / eval and return
fn:.J.wrapfn["sum"];                            / wrap some function
fn[1;2;3;4]; fn[100?1f];                        / variable number of arguments
.J.set["mylist";20?`4]; .J.e "@show mylist";    / move a variable into julia 
```



## Type Conversions kdb/q $\longleftrightarrow$ Julia

| **kdb/q type**    | **Received from kdb/q**                         | **Sent From Julia**                                               |
|:---------------	|:-------------------------------------------	|:---------------------------------------------------------------	|
| `bool`     	    | `Bool`                                    	| `Bool`                                                        	|
| `byte`        	| `UInt8`                                   	| `UInt8`                                                       	|
| `short`       	| `Int16`                                   	| `Int16`                                                       	|
| `int`         	| `Int32`                                   	| `Int32`                                                       	|
| `long`        	| `Int64`                                   	| `Int64`                                                       	|
| `real`        	| `Float32`                                 	| `Float32`                                                     	|
| `float`       	| `Float64`                                 	| `Float64`                                                     	|
| `char`        	| `Char`                                    	| `Char`                                                        	|
| `symbol`      	| `Symbol`                                  	| `Symbol`                                                      	|
| `timestamp`   	| `NanoDates.NanoDate`                      	| `TimesDates.TimeDate`, `<:Dates.AbstractDateTime`             	|
| `month`       	| `Date`                                    	| NA                                                            	|
| `date`        	| `Date`                                    	| `Dates.Date`                                                  	|
| `datetime`    	| `NanoDates.NanoDate`                      	| NA                                                            	|
| `timespan`    	| `Dates.Time`                              	| `Dates.Time`                                                  	|
| `minute`      	| `Minute`                                  	| NA                                                            	|
| `second`      	| `Second`                                  	| NA                                                            	|
| `time`        	| `Dates.Time`                              	| NA                                                            	|
| `table`       	| `DataFrame`                              	    | Any `Tables.jl` interface                                     	|
| `keyed table` 	| `DataFrame`                              	    | NA                                                            	|
| `dictionary`  	| `Dictionary{Symbol,Any}`                  	| `<:AbstractDictionary`                                        	|
| `atomic list` 	| `Vector{T}` or `Vector{Union{T,Missing}}` 	| Any iterator with `eltype` equal to `T` or `Union{T,Missing}` 	|
| `mixed list`  	| `Vector{Any}`                             	| Any iterator with `eltype = Any`                              	|
| `functions`     	| Unsupported                                	| Unsupported                                                     	|


