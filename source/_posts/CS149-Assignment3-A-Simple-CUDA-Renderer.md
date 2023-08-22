---
title: CS149-Assignment3 A Simple CUDA Renderer
tags:
  - 技术
  - 学习
categories:
  - CS149
date: 2023-06-07 22:18:34
---


## Part 1

The result is illustrated in the following figure.

### Question 1

As you can see, due to the parallelism of the GPU, the bandwidth is huge.

### Question 2

From the results, we can get that the bottleneck is the memory and CPU of the host itself. For the execution time of the kernel, the speed is very fast. However, the speed is low for copying the host memory into the device memory and vice versa.

## Part 2

First, we need to implement the parallelism form of `scan`. The algorithm is super wonderful. You should first understand the algorithm.

```c
void exclusive_scan_iterative(int* start, int* end, int* output) {

    int N = end - start;
    memmove(output, start, N*sizeof(int));

    // upsweep phase
    for (int two_d = 1; two_d <= N/2; two_d*=2) {
        int two_dplus1 = 2*two_d;
        parallel_for (int i = 0; i < N; i += two_dplus1) {
            output[i+two_dplus1-1] += output[i+two_d-1];
        }
    }

    output[N-1] = 0;

    // downsweep phase
    for (int two_d = N/2; two_d >= 1; two_d /= 2) {
        int two_dplus1 = 2*two_d;
        parallel_for (int i = 0; i < N; i += two_dplus1) {
            int t = output[i+two_d-1];
            output[i+two_d-1] = output[i+two_dplus1-1];
            output[i+two_dplus1-1] += t;
        }
    }
}
```

It may seem that we need `N` threads for every inner loop, however, this is a stupid idea, we should calculate how many threads we need for each inner loop.

## Part 3

Before diving into how to write a render using CUDA. We first think about how to implement a render with cpp. And after that, you need to read the code written with CUDA.

### First Implementation

For the first implementation, I focus on how to write the correct code. The hint part `exclusive_scan` has made me think we could first store the circle render parameters for each pixel. Thus we can scan the circle render parameters for each pixel.

Thus, we can solve the two important questions:

+ *Atomicity*: we have made the memory independent for each pixel. So we can write whatever we want without any synchronization and mutation.
+ *Order*: We have make an explicit array of render parameters. So the order doesn't matter.

So, We first need to construct an array to hold the render parameters for each pixel.

```c++
cudaMalloc(&cudaDevicePixelData, image->width * image->height * numCircles * 4 * sizeof(float));
```

Now, we first write the render information for every pixel. The core code is below:

```c++
    int pixelPtrStart = 4 * cuConstRendererParams.numCircles * (pixelY * imageWidth + pixelX) + 4 * circleIndex;
    float4* pixelPtr = (float4*)(&cuConstRendererParams.pixelData[pixelPtrStart]);
    float4 color;
    color.x = rgb.x;
    color.y = rgb.y;
    color.z = rgb.z;
    color.w = alpha;

    *pixelPtr = color;
```

Then we can launch a new kernel to calculate for each pixel:

```c++
for (int i = 0; i < numCircles; ++i) {
    float4* pixelPtr = (float4*)(&cuConstRendererParams.pixelData[startIndex + i * 4]);
    color.x = (*pixelPtr).w * (*pixelPtr).x + (1 - (*pixelPtr).w) * color.x;
    color.y = (*pixelPtr).w * (*pixelPtr).y + (1 - (*pixelPtr).w) * color.y;
    color.z = (*pixelPtr).w * (*pixelPtr).z + (1 - (*pixelPtr).w) * color.z;
    color.w += (*pixelPtr).w;
  }
  *imagePtr = color;
```

However, `cudaDevicePixelData` is too big, which will consume so much memory, which would exceed the maximum memory 16GB. So we need to reuse the render information. We should not store the information for every pixel. Because the render information of each circle is deterministic. But the things we need to make sure is that whether the circle has contributed to the pixel. So instead of storing the render information for each pixel, we could just use bit mask to indicate whether the circle has contributed to the pixel.

Now, the result is illustrated by below.

![first render score](https://s2.loli.net/2023/06/07/JyrZ64K8W2uI3Tp.png)

As you can see, we have passed some tests and the performance is not good at all. The reason why there are some failed tests is that the bytes exceed $2^{63}$, which would overflow.

### Best practice

We should use a clever way to solve this problem. The problem is a two-dimensional problem. So we'd better design the grid and block for two dimensions, as the following figures illustrates:

![grid and block design for rendering](https://s2.loli.net/2023/08/22/MWiVgyBYRU2CmlD.png)

From the above figures, we could write the following code to initialize the grid and block size:

```c++
void CudaRenderer::render() {
  int block_x = 16, block_y = 32;
  dim3 blockDim(block_x, block_y);
  int grid_x = (image->width + blockDim.x - 1) / blockDim.x;
  int grid_y = (image->height + blockDim.y - 1) / blockDim.y;
  dim3 gridDim(grid_x, grid_y);
}
```

And we should render a block each time, we have already split the pixels into the block. For each block, we should handle the each pixel, we could calculate the current pixel's x coordinate and y coordinate and the current thread id:

```c++
int thread_id = threadIdx.y * blockDim.x + threadId.x;
int pixel_index_x = blockId.x * blockDim.x + threadId.x;
int pixel_index_y = blockId.y * blockDim.y + threadId.y;
```

So what a thread should do? We have already mapped a pixel $(x, y)$ with a specified thread. So the idea should be simple enough.

+ **We could traverse sequentially all the circles to tell whether the circle has contributed to this pixel, and we calculate to achieve rendering**.

However, this way is too slow. For every pixel, we could calculate a lot of useless calculation. For example, if there are total 100,000,000 circles, there would be many useless circles for this pixel. So we should find a way to find the circles in the current block. So we could use a shared memory (a bit map) to know whether the *BLOCK* size circle has contributed to the block. So for each pixel (each thread) we could calculate ONLY one circle to know whether it has contributed to this block. And for each pixel, we could use the contributed circles to sequentially achieve rendering.

Look at what parallelism we have done:

+ Each pixel renders itself using two-dimension task split.
+ Each pixel handles 1 circle to tell whether it contributes to the current block for $num_{circles} / size_{block}$ loops.

It's a really hard question, but it is very useful.
