was using too much cpu so i said
" have it report using less that 1 percent "  chatgpt
my compile command     --- modify to suit your fath and pile

nvcc -ccbin "C:\Program Files (x86)\Microsoft Visual Studio\2022\BuildTools\VC\Tools\MSVC\14.42.34433\bin\Hostx64\x64" -arch=sm_60  gpu_monitor2.cu -o gpu_monitor2.exe  -allow-unsupported-compiler -D_ALLOW_COMPILER_AND_STL_VERSION_MISMATCH

" =====================================
#include <iostream>
#include <thread>
#include <chrono>
#include <cuda_runtime.h>

// Dummy kernel to keep the GPU active
__global__ void keepGpuAwake() {
    while (true) {
        // Simple computation to keep GPU active
        int idx = threadIdx.x + blockIdx.x * blockDim.x;
        if (idx == 0) {
            // Prevent compiler optimizations with dummy operations
            clock_t start = clock();
            while (clock() - start < 1000) {
                // Busy wait (1 second per loop iteration)
            }
        }
    }
}

int main() {
    size_t freeMemory = 0, totalMemory = 0;

    // Launch a persistent dummy kernel on the GPU
    keepGpuAwake<<<1, 1>>>();
    cudaError_t err = cudaGetLastError();
    if (err != cudaSuccess) {
        std::cerr << "Kernel launch failed: " << cudaGetErrorString(err) << std::endl;
        return 1;
    }

    std::cout << "Monitoring GPU memory. Press Ctrl+C to stop." << std::endl;

    while (true) {
        // Get current CUDA memory info
        err = cudaMemGetInfo(&freeMemory, &totalMemory);
        if (err != cudaSuccess) {
            std::cerr << "CUDA Error: " << cudaGetErrorString(err) << std::endl;
            break;
        }

        // Convert bytes to MB for easier readability
        std::cout << "Free: " << freeMemory / (1024.0 * 1024.0) 
                  << " MB, Total: " << totalMemory / (1024.0 * 1024.0) << " MB" << std::endl;

        // Sleep for 5 seconds to reduce CPU usage
        std::this_thread::sleep_for(std::chrono::seconds(5));
    }

    return 0;
}
