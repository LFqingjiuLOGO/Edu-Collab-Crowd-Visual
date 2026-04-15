# 推荐教程

[Pytorch+cpp/cuda extension 教學 tutorial 6 反向傳播 - English CC -](https://www.youtube.com/watch?v=l_Rpk6CRJYI&list=PLDV2CyUo4q-LKuiNltBqCKdO9GH4SS_ec)

视频的相应仓库[https://github.com/kwea123/pytorch-cppcuda-tutorial](https://github.com/kwea123/pytorch-cppcuda-tutorial)

包括类的编写，torch.autograd和反向传播等


# 简单函数的pybind编译

cuda kernel可以被绑定（bind）为一个python函数被使用，一般分为两种方式，`JIT即时编译方式`和`setup extension方式`

## cpp bind过程 

无论使用任何一种方式，都需要额外写一段cpp代码来完成绑定过程

```cpp
#include <torch/extension.h>
#include <torch/types.h>
#include "kernels.h" //放你写好的kernel

// 给torch看的接口函数
torch::Tensor CUDA_MatMul_naive(torch::Tensor &C, 
    torch::Tensor &A, torch::Tensor &B, int M, int K, int N){
    
    MatMul_naive_interface(A, B, C, M, K, N);
    return C;
}

PYBIND11_MODULE(TORCH_EXTENSION_NAME, m){
    m.def("the_name_to_call_the_func_in_python", &CUDA_MatMul_naive, "descriptions of the function")
}

```


## JIT 即时编译模式
env requirements
```
torch,nanja
```
代码示例

```python
import torch
from torch.utils.cpp_extensions import load

lib = load(
    name='torch_matrix_mul', 
    sources=['kernels.cu', 'bindings.cpp'],  # CUDA源文件和绑定文件列表
    extra_cuda_cflags=['-O3', '-lineinfo'],  # 额外的CUDA编译标志，可选
    verbose=True  # 显示编译过程，可选  
)

# 函数调用示例 
# lib.the_name_to_call_the_func_in_python(A, B, C, M, K, N)
```

## setupExtension 将模块提前编译为lib
env requirements

    torch,setuptools,


需要提前写setup.py执行编译过程，每次cuda代码更改都要再次编译

```python

from setuptools import setup, Extension
from torch.utils.cpp_extension import BuildExtension, CUDAExtension

# 示例，新增的模块名称torch_matmul_ext，
setup(  
    name='torch_matmul_ext',
    ext_modules=[
        CUDAExtension(
            name='torch_matmul_ext',
            sources=[
                'kernels.cu',
                'bindings.cpp',
            ],
            extra_compile_args={
                'cxx': ['-O3'],
                'nvcc': [
                    '-O3', 
                    '-lineinfo',    #ncu调试用
                    '-arch=sm_75',  # 根据你的GPU架构选择合适的arch
                ]
            }
        )
    ],
    cmdclass={
        'build_ext': BuildExtension
    }
)        

```


可以提前设置架构的环境变量

```bash
# 提前查询设备的计算能力 RTX2080是Turning架构 Compute Capability=7.5
export TORCH_CUDA_ARCH_LIST="7.5"   # linux
# set TORCH_CUDA_ARCH_LIST=7.5    #  windows powershell
```

如果想编译并安装到当前的python环境的site-packages下
```bash
pip install .
```

如果仅仅想在当前文件夹下生成编译结果
```bash
python setup.py build_ext --inplace
```
按照上面的例子，会在当前项目下生成 `torch_matmul_ext.cp310-win_amd64.pyd`
