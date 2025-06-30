# Orginal Code Structure
This profile is an explanation for the original Julia code adopt from Dr. Zhuocheng Lu. The original code is written for Photon Drag Effect calculation.  
This working directory consists of:  
1. **Constants.jl**:  
2. **Sub.jl**:  
3. **TBconductivity.jl**:  
4. **TBmatrix.jl**:  
5. **TBread.jl**:  
6. **WriteToFile.jl**:  

## TBread.jl
```
# info: read and write TB infomation

# module TBread

# Export for external access
# export Wannier90TB, readWAN90TB

# Usage
using LinearAlgebra
using Printf
using Dates

# struct
struct Wannier90TB
  Latt::Array{Float64,2}
  num_wann::Int64
  nrpts::Int64
  ndegen::Array{Int64,1}
  Rlatt::Array{Int64,2}
  hopps::Array{ComplexF64,3}
  pos_r::Array{ComplexF64,4}
end

# new方法构建结构体 类似于python的class，内涵多种数据结构
function Wannier90TB(Latt, num_wann, nrpts, ndegen, Rlatt, hopps, pos_r)
    new(Latt[:,:], num_wann, nrpts, ndegen[:], Rlatt[:,:], hopps[:,:,:], pos_r[:,:,:,:])
end
 
function readWAN90TB(filename::String)
    # 打开文件
    open(filename, "r") do file
        # 跳过第一行（时间信息）
        readline(file)

        # 读取晶格矢量
        # Latt = [parse.(Float64, split(readline(file))) for _ in 1:3]
        Latt = zeros(Float64, 3, 3) 
        for k in 1:3
            Latt[k, :] = parse.(Float64, split(readline(file)))
        end
        
        # 读取 num_wann 和 nrpts
        # num_wann: number of wannierfunction, the Hamiltonian size = num_wann * num_wann
        # nrpts: number of real-space hopping terms \neq num_wann
        num_wann = parse(Int64, readline(file))
        nrpts = parse(Int64, readline(file))
        
        # 读取简并度（ndegen）
        ndegen = Int64[]
        for _ in 1:ceil(Int64, nrpts / 15)
          append!(ndegen, parse.(Int64, split(readline(file))))
        end

        # 初始化矩阵
        Rlatt = zeros(Int64, 3, nrpts)  
        hopps = zeros(ComplexF64,num_wann,num_wann,nrpts)
        pos_r = zeros(ComplexF64,num_wann,num_wann,3,nrpts) 
        
        # 读取 hopping 信息
        for k in 1:nrpts
            # 跳过空行
            readline(file)
            # 读取每个 block 的 Rlatt
            # Rlatt：晶格偏移矢量
            Rlatt[:, k] = parse.(Int64, split(readline(file)))
            
            for j in 1:num_wann
                for i in 1:num_wann
                    # 读取Rlatt偏移的hopping强度矩阵：num_wann * num_wann
                    # 第一列是实部，第二列是虚部
                    row = split(readline(file))
                    hopps[i, j, k] = ComplexF64(parse(Float64, row[3]), parse(Float64, row[4]))
                end
            end
        end
        
        # 读取 position 算符矩阵（pos_r）
        # position matrix = Re<n|r_x|m> Im<n|r_x|m> Re<n|r_y|m> Im<n|r_y|m> Re<n|r_z|m> Im<n|r_z|m>
        for k in 1:nrpts
            # 跳过空行
            readline(file)
            # 跳过每个 block 的 Rlatt
            readline(file)
            
            for j in 1:num_wann
                for i in 1:num_wann
                    row = split(readline(file))
                    pos_r[i, j, 1, k] = ComplexF64(parse(Float64, row[3]), parse(Float64, row[4]))
                    pos_r[i, j, 2, k] = ComplexF64(parse(Float64, row[5]), parse(Float64, row[6]))
                    pos_r[i, j, 3, k] = ComplexF64(parse(Float64, row[7]), parse(Float64, row[8]))
                end
            end
        end
        
        # 返回数据
        w90tb = Wannier90TB(Latt, num_wann, nrpts, ndegen, Rlatt, hopps, pos_r)
        return w90tb
    end
end

# end # module TBread
```
Explanation for the functions within

**open(f::Function, args...; kwargs...)**
Apply the function f to the result of open(args...; kwargs...) and close the resulting file descriptor upon completion.

Examples
```
julia> write("myfile.txt", "Hello world!");

julia> open(io->read(io, String), "myfile.txt")
"Hello world!"

julia> rm("myfile.txt")
open(filename::AbstractString; lock = true, keywords...) -> IOStream
```

**parse(type, str; base)**
Parse a string as a number. For Integer types, a base can be specified (the default is 10). For floating-point types, the string is parsed as a decimal floating-point number. Complex types are parsed from decimal strings of the form "R±Iim" as a Complex(R,I) of the requested type; "i" or "j" can also be used instead of "im", and "R" or "Iim" are also permitted. If the string does not contain a valid number, an error is raised.

!!! compat "Julia 1.1" parse(Bool, str) requires at least Julia 1.1.

Examples
```
julia> parse(Int, "1234")
1234

julia> parse(Int, "1234", base = 5)
```

**difference between parse. and parse**
parse(T, x) 是对单个字符串 x 进行类型解析；  
parse.(T, xs) 是对一个字符串数组 xs 中的每个元素分别解析，返回一个对应类型的数组。

**readline(io::IO=stdin; keep::Bool=false)**
readline(filename::AbstractString; keep::Bool=false)  
Read a single line of text from the given I/O stream or file (defaults to stdin). When reading from a file, the text is assumed to be encoded in UTF-8. Lines in the input end with '\n' or "\r\n" or the end of an input stream. **When keep is false (as it is by default), these trailing newline characters are removed from the line before it is returned.** When keep is true, they are returned as part of the line.  

Return a String. See also copyline to instead write in-place to another stream (which can be a preallocated IOBuffer).

See also readuntil for reading until more general delimiters.

Examples  
```
julia> write("my_file.txt", "JuliaLang is a GitHub organization.\nIt has many members.\n");

julia> readline("my_file.txt")
"JuliaLang is a GitHub organization."

julia> readline("my_file.txt", keep=true)
"JuliaLang is a GitHub organization.\n"

julia> rm("my_file.txt")
julia> print("Enter your name: ")
Enter your name:

julia> your_name = readline()
Logan
"Logan"
Base.readline is a function with 4 methods

readline(s::Base.IOStream) in Base at iostream.jl:452

readline() in Base at io.jl:619

readline(s::Core.IO) in Base at io.jl:619

readline(filename::Core.AbstractString) in Base at io.jl:617
```

## TBmatrix.jl

**Base.include([mapexpr::Function,] m::Module, path::AbstractString)** 
Evaluate the contents of the input source file in the global scope of module m. Every module (except those defined with baremodule) has its own definition of include omitting the m argument, which evaluates the file in that module. Returns the result of the last evaluated expression of the input file. During including, a task-local include path is set to the directory containing the file. Nested calls to include will search relative to that path. This function is typically used to load source interactively, or to combine files in packages that are broken into multiple source files.  

The optional first argument mapexpr can be used to transform the included code before it is evaluated: for each parsed expression expr in path, the include function actually evaluates mapexpr(expr). If it is omitted, mapexpr defaults to identity.  

!!! compat "Julia 1.5" Julia 1.5 is required for passing the mapexpr argument.  

Base.include is a function with 3 methods  

include(mapexpr::Core.Function, mod::Core.Module, _path::Core.AbstractString) in Base at Base.jl:558  

include(mod::Core.Module, _path::Core.AbstractString) in Base at Base.jl:557  

include(x::Core.AbstractString) in Base at Base.jl:558  

