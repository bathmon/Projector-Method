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
            Rlatt[:, k] = parse.(Int64, split(readline(file)))
            
            for j in 1:num_wann
                for i in 1:num_wann
                    row = split(readline(file))
                    hopps[i, j, k] = ComplexF64(parse(Float64, row[3]), parse(Float64, row[4]))
                end
            end
        end
        
        # 读取 position 算符矩阵（pos_r）
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
