was using too much cpu so i said
" have it report using less that 1 percent "  chatgpt

" =====================================
#include <atomic>
#include <csignal>
#include <iostream>
#include <vector>
#include <thread>
#include <chrono>
#include <cuda_runtime.h>

#define CUDA_CHECK(call)                                                    \
    {                                                                       \
        cudaError_t err = call;                                             \
        if (err != cudaSuccess) {                                           \
            std::cerr << "CUDA Error at " << __FILE__ << ":" << __LINE__    \
                      << " - " << cudaGetErrorString(err) << std::endl;     \
            exit(EXIT_FAILURE);                                             \
        }                                                                   \
    }

__global__ void dummy_kernel(int* data, int N) {
    int idx = threadIdx.x + blockIdx.x * blockDim.x;
    if (idx < N) {
        data[idx] = data[idx] * 2; // Simple operation
    }
}

// Global flag for exiting the loop
std::atomic<bool> keepRunning(true);

// Signal handler to set the exit flag
void signalHandler(int signum) {
    keepRunning = false;
}

void runCudaKernel() {
    const int N = 1024 * 1024;
    std::vector<int> data(N, 1);

    int* dev_data = nullptr;
    CUDA_CHECK(cudaMalloc((void**)&dev_data, N * sizeof(int)));
    CUDA_CHECK(cudaMemcpy(dev_data, data.data(), N * sizeof(int), cudaMemcpyHostToDevice));

    const int threadsPerBlock = 256;
    const int blocksPerGrid = (N + threadsPerBlock - 1) / threadsPerBlock;

    try {
        while (keepRunning) {
            dummy_kernel<<<blocksPerGrid, threadsPerBlock>>>(dev_data, N);
            CUDA_CHECK(cudaDeviceSynchronize());

            // Sleep for 1 second
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    } catch (const std::exception& ex) {
        std::cerr << "Error: " << ex.what() << std::endl;
    }

    if (dev_data) {
        CUDA_CHECK(cudaFree(dev_data));
    }
}

int main() {
    std::signal(SIGINT, signalHandler);

    int deviceCount = 0;
    CUDA_CHECK(cudaGetDeviceCount(&deviceCount));
    if (deviceCount == 0) {
        std::cerr << "No CUDA devices detected!" << std::endl;
        return -1;
    }

    CUDA_CHECK(cudaSetDevice(0));
    std::cout << "Running CUDA kernel. Press Ctrl+C to exit..." << std::endl;

    runCudaKernel();

    std::cout << "Program terminated safely." << std::endl;
    return 0;
}
