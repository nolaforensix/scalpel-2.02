// Scalpel Copyright (C) 2005-8 by Golden G. Richard III and 
// 2007-8 by Vico Marziale.
// Written by Golden G. Richard III and Vico Marziale.
//
// This program is free software; you can redistribute it and/or
// modify it under the terms of the GNU General Public License as
// published by the Free Software Foundation; either version 2 of the
// License, or (at your option) any later version.
// 
// This program is distributed in the hope that it will be useful, but
// WITHOUT ANY WARRANTY; without even the implied warranty of
// MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
// General Public License for more details.

// You should have received a copy of the GNU General Public License
// along with this program; if not, write to the Free Software
// Foundation, Inc., 51 Franklin Street, Fifth Floor, Boston, MA
// 02110-1301, USA.
//
// Thanks to Kris Kendall, Jesse Kornblum, et al for their work 
// on Foremost.  Foremost 0.69 was used as the starting point for 
// Scalpel, in 2005.
//


#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <math.h>
#include <unistd.h>
#include <cutil.h> 
#include "common.h"


// returns TRUE if a is alpha
#define ISALPHA(a) ((a >= 'A' && a <= 'Z') || (a >= 'a' && a <= 'z'))


// Globals.
__constant__ char pattern[MAX_PATTERNS][MAX_PATTERN_LENGTH];
__constant__ char lookup_headers[LOOKUP_ROWS][LOOKUP_COLUMNS]; 
__constant__ char lookup_footers[LOOKUP_ROWS][LOOKUP_COLUMNS]; 

char host_patterns[MAX_PATTERNS][MAX_PATTERN_LENGTH];
char hostlookup_headers[LOOKUP_ROWS][LOOKUP_COLUMNS]; 
char hostlookup_footers[LOOKUP_ROWS][LOOKUP_COLUMNS]; 

int GPU_ERROR = 0;	// Unused for now, but shouldn't be ;) 

char *dev_in;		// device buffer for input
char *dev_out;	// device buffer for results output
int *dev_count;	// device buffer for intermediate results


// Forward declarations for private functions.
static void checkCUDAError(const char *msg);
/*
static int set_last_device();		// currently unused
static int enumerate_devices();	// currently unused
*/

// Performs search for headers and footers in bufize dev_in, and puts encoded
// results into dev_out. 
__global__ void gpudigbuffer_kernel(char *dev_in, char *dev_out, int bufsize,
									int longestneedle, int *dev_count, char wildcard) {

 // Per-block shared memory, size determined by kernel invocation.
  extern __shared__  char sdata[];
  
  // My thread, block id, the number of threads per block, and my global tid, 
  // aka my unique thread tid across all thread blocks.
  const unsigned int tid = threadIdx.x;
  const unsigned int bid = blockIdx.x;
  const unsigned int num_thread_blocks = gridDim.x;
  const unsigned int g_tid = (THREADS_PER_BLOCK * bid) + tid; 
   
  // Per thread block input and output buffers in shared memory.
	char *inbuf = sdata;
	char *outbuf = sdata + THREADS_PER_BLOCK;

	// Clear the output buffer.  
	outbuf[tid] = 0;
	__syncthreads();
	
  // Note the last thread (globally) we want to have search in this buffer. 
  const unsigned int bytes_for_last_block = bufsize % (THREADS_PER_BLOCK - longestneedle);
  const unsigned int last_thread = ((num_thread_blocks - 1) *\
  		THREADS_PER_BLOCK) + (bytes_for_last_block - 1);
  
  // Get the block of global data that this thread block is responsible for.
  // Must account for needles which might overlap block boundaries.
  inbuf[tid] = dev_in[g_tid - (longestneedle * bid)];
  __syncthreads();

		
	int case_insen = FALSE;	// Is the current needle case insensitive?
	int result_ix = 0;			// Index in output buffer where we can write.
	int wc_shift = 0;				// For needles with leading wildcards.
	int i = 0, j = 0;
	
	// Lets find headers.
	// Loop over potential matches.
	// Get first potential match.
	char pindex = lookup_headers[(unsigned char)inbuf[tid]][i];
	while(pindex != LOOKUP_ENDOFROW) {
		j=0;
		case_insen = !(pattern[pindex][PATTERN_CASESEN]);
		wc_shift = pattern[pindex][PATTERN_WCSHIFT];

		if((((int)tid - wc_shift) >= 0) &&  
				(tid < (THREADS_PER_BLOCK - longestneedle + wc_shift)) && 
				(g_tid < last_thread + wc_shift)){
			// This may look complicated, but breaking it up is Retardedly Slower. 
			// while(inbuf matches pattern) {
			while(((inbuf[tid +j] == pattern[pindex][j+1+wc_shift]) ||	// direct match
					(pattern[pindex][j+1+wc_shift] == wildcard) ||					// wildcard match
					(case_insen && ISALPHA(inbuf[tid+j]) && ISALPHA(pattern[pindex][j+1+wc_shift]) && // case insensitive
					(((inbuf[tid+j] - pattern[pindex][j+1+wc_shift]) == 32) || 
					((inbuf[tid+j] - pattern[pindex][j+1+wc_shift]) == -32))))  &&
					j < (pattern[pindex][0] - wc_shift)) {
				j++;
    	}
     	// Have match if number of matching bytes equals the size of the pattern.
     	if (j == (pattern[pindex][0] - wc_shift)) {
				// Result_ix keeps track of where in the output buffer we can write.
				// atomicAdd returns the pre-incremented value. We encode a match as
				// 2 bytes, 1st: the matching pattern index, 2nd: the position in the
				// input buffer where the match begins.
				result_ix = atomicAdd(&dev_count[bid], 1);
				outbuf[result_ix*2] = pindex/2 + 1;
				outbuf[(result_ix*2)+1] = tid - wc_shift;	
     	}
    }
		i+=1;
		// Get next potential match.
		pindex = lookup_headers[(unsigned char)inbuf[tid]][i];
	}
	__syncthreads();
		
	// Same as above but for footers. No shift as we assume footers will not
	// have leading wildcard characters.
	if ((tid < (THREADS_PER_BLOCK - longestneedle)) && (g_tid < last_thread)){
	i=0; j=0;
	pindex = lookup_footers[(unsigned char)inbuf[tid]][i];
	while(pindex != LOOKUP_ENDOFROW) {
		j=0;
		case_insen = !(pattern[pindex][PATTERN_CASESEN]);
		while(((inbuf[tid + j] == pattern[pindex][j+1]) ||	// direct match
				(pattern[pindex][j+1] == wildcard) ||					// wildcard match
				(case_insen && ISALPHA(inbuf[tid + j]) && ISALPHA(pattern[pindex][j+1]) && // case insensitive
				(((inbuf[tid+j] - pattern[pindex][j+1]) == 32) || 
				((inbuf[tid+j] - pattern[pindex][j+1]) == -32))))  &&
				j < pattern[pindex][0]) {
			j++;
    }
     	if (j == pattern[pindex][0]) {
				result_ix = atomicAdd(&dev_count[bid], 1);
				outbuf[result_ix*2] =  -1 * (pindex/2+1);
				outbuf[(result_ix*2)+1] = tid;
      } 
			i+=1;
			pindex = lookup_footers[(unsigned char)inbuf[tid]][i];
		}
	}
	__syncthreads();


	// The scalpel engine requires that it see the needles in the order in which
	// they appear in the image, the above code gives no ordering guarantees, 
	// so we'll just go ahead and sort, but only if we found any results.
	if(dev_count[bid] > 1) {
		// Sort outbuf by foundat index first. Bubble! Really? Not parallel? Luser.
		if(tid == 0) {
			int changed = TRUE;
			char tmp = 0;
			while(changed) {
				changed = FALSE;
				for(i=0; i<(dev_count[bid]*2) - 2; i+=2) {
					if((unsigned char)outbuf[i+1] > (unsigned char)outbuf[i+3]) {
						tmp = outbuf[i+1];
						outbuf[i+1] = outbuf[i+3];
						outbuf[i+3] = tmp;
						tmp = outbuf[i];
						outbuf[i] = outbuf[i+2];
						outbuf[i+2] = tmp;
						changed = TRUE;
					}
				}
			}
		}
	}
	__syncthreads();				

	// Copy the results buffer out to global RAM, but only if dev_count[bid] says
	// that we found some results to copy back, otherwise skip the write.
	if(dev_count[bid] > 0) {
		if(tid < RESULTS_SIZE_PER_BLOCK) {
			dev_out[(bid*RESULTS_SIZE_PER_BLOCK)+tid] = outbuf[tid];
		}
	}
	__syncthreads();
	// Done with this buffer.
}
	

// Host-side code to set up for the GPU powered search above.
int gpuSearchBuffer(char *input, int bufsize, char *output, int longestneedle, char wildcard) {
	
	int maxblocks = (SIZE_OF_BUFFER / (THREADS_PER_BLOCK - longestneedle))+1;
	
	// copy host buffer to device memory
  CUDA_SAFE_CALL(cudaMemcpy(dev_in, input, bufsize,
			    cudaMemcpyHostToDevice));
		
	// Set up kernel execution parameters.	
	int num_thread_blocks = (bufsize / (THREADS_PER_BLOCK - longestneedle)) +1;
  dim3 grid(num_thread_blocks, 1, 1);
  dim3 threads(THREADS_PER_BLOCK, 1, 1);
  
  // Clear the results buffer between invocations. Device RAM is persistent
  // across kernel invocations.
  CUDA_SAFE_CALL(cudaMemset((void *)dev_count, 0, num_thread_blocks*sizeof(int)));	
  CUDA_SAFE_CALL(cudaMemset((void *)dev_out, 0, maxblocks*RESULTS_SIZE_PER_BLOCK));	
  
	// Execute the kernel.
  gpudigbuffer_kernel<<<grid, threads, (THREADS_PER_BLOCK*2)>>>
								 (dev_in, dev_out, bufsize, longestneedle, dev_count, wildcard); 

	// Kernel invocation is asynchronous, wait till done before copying results.
	checkCUDAError("kernel invocation");
  cudaThreadSynchronize();

  // Copy encoded results back to the host.
  CUDA_SAFE_CALL(cudaMemcpy(output, dev_out, maxblocks*RESULTS_SIZE_PER_BLOCK, cudaMemcpyDeviceToHost));
	
	return GPU_ERROR;
}


// Helper functions.

// Copy the header / footer patterns table to constant memory on the GPU.
void copytodevicepattern(char hostpatterntable[MAX_PATTERNS][MAX_PATTERN_LENGTH]) {
	
	memcpy(host_patterns, hostpatterntable, MAX_PATTERNS*MAX_PATTERN_LENGTH);
	CUDA_SAFE_CALL(cudaMemcpyToSymbol(pattern, hostpatterntable, sizeof(pattern), 0));
}


// Copy the header lookup table to constant memory on the GPU.
void copytodevicelookup_headers(char hostlookuptable[LOOKUP_ROWS][LOOKUP_COLUMNS]){

	memcpy(hostlookup_headers, hostlookuptable, LOOKUP_ROWS*LOOKUP_COLUMNS);
	CUDA_SAFE_CALL(cudaMemcpyToSymbol(lookup_headers, hostlookuptable, sizeof(lookup_headers), 0));
}


// Copy the footer lookup table to constant memory on the GPU.
void copytodevicelookup_footers(char hostlookuptable[LOOKUP_ROWS][LOOKUP_COLUMNS]){

	memcpy(hostlookup_footers, hostlookuptable, LOOKUP_ROWS*LOOKUP_COLUMNS);
	CUDA_SAFE_CALL(cudaMemcpyToSymbol(lookup_footers, hostlookuptable, sizeof(lookup_footers), 0));
}


// Wrapper for allocating RAM on the GPU.
void ourCudaMallocHost(void **ptr, int len)
{
		if(cudaHostAlloc(ptr, len, cudaHostAllocDefault)) {
				fprintf(stderr, "\nERROR: cudaMallocHost \n\n");
		}
}


// Allocate persistent device memory. 
int gpu_init(int longestneedle) {

	// We could set a device here, but to do it intelligently we need to know
	// not only which device is not attached to a display, we also need to know
	// which pci-x bus supports pinned memory. Have no idea how to do that
	// except to test live. Maybe later.

	// Allocate (persistent) device memory for the entire run.
	
	// Input buffer.
	CUDA_SAFE_CALL(cudaMalloc((void**)&dev_in, SIZE_OF_BUFFER));
	
	// The maximum number of blocks the GPU will run.
	int maxblocks = (SIZE_OF_BUFFER / (THREADS_PER_BLOCK - longestneedle))+1;
	
	// Output buffer.
	CUDA_SAFE_CALL(cudaMalloc((void**)&dev_out, maxblocks*RESULTS_SIZE_PER_BLOCK));
	
	// Buffer for counts of needles found per block. 
	CUDA_SAFE_CALL(cudaMalloc((void**)&dev_count,maxblocks*sizeof(int)));
		
	return GPU_ERROR;
}


// Free persistent device memory.
int gpu_cleanup() {

	CUDA_SAFE_CALL(cudaFree(dev_in));	
	CUDA_SAFE_CALL(cudaFree(dev_out));
	CUDA_SAFE_CALL(cudaFree(dev_count));
	
	return GPU_ERROR;

}


// Checks if the last CUDA operation generated an error.
static void checkCUDAError(const char *msg)
{
    cudaError_t err = cudaGetLastError();
    if( cudaSuccess != err) 
    {
        fprintf(stderr, "Cuda error: %s: %s.\n", msg, 
                                  cudaGetErrorString( err) );
        exit(EXIT_FAILURE);
    }                         
}

/*
// Set us up to use the last device (least likely to be use by display?)
// Pay attention to which device gived speedups for pinned memory, it's
// not necessarily all of them.
static int set_last_device() {

	int device_count;
    CUDA_SAFE_CALL(cudaGetDeviceCount(&device_count));
    if(device_count == 0) {
        fprintf(stderr, "There is no device supporting CUDA\n");
	}
	device_count--;
	fprintf(stderr, "Setting device %d\n", device_count); 
	cudaSetDevice(device_count);
	
	return GPU_ERROR;
	
}


// Print out info on devices installed on the system
static int enumerate_devices() {
	
	int deviceCount;
    CUDA_SAFE_CALL(cudaGetDeviceCount(&deviceCount));
    if (deviceCount == 0)
        printf("There is no device supporting CUDA\n");
    int dev;
    for (dev = 0; dev < deviceCount; ++dev) {
        cudaDeviceProp deviceProp;
        CUDA_SAFE_CALL(cudaGetDeviceProperties(&deviceProp, dev));
        if (dev == 0) {
            if (deviceProp.major == 9999 && deviceProp.minor == 9999)
                printf("There is no device supporting CUDA.\n");
            else if (deviceCount == 1)
                printf("There is 1 device supporting CUDA\n");
            else
                printf("There are %d devices supporting CUDA\n", deviceCount);
        }
        printf("\nDevice %d: \"%s\"\n", dev, deviceProp.name);
        printf("  Major revision number:                         %d\n",
               deviceProp.major);
        printf("  Minor revision number:                         %d\n",
               deviceProp.minor);
        printf("  Total amount of global memory:                 %u bytes\n",
               deviceProp.totalGlobalMem);
    #if CUDART_VERSION >= 2000
        printf("  Number of multiprocessors:                     %d\n",
               deviceProp.multiProcessorCount);
        printf("  Number of cores:                               %d\n",
               8 * deviceProp.multiProcessorCount);
    #endif
        printf("  Total amount of constant memory:               %u bytes\n",
               deviceProp.totalConstMem); 
        printf("  Total amount of shared memory per block:       %u bytes\n",
               deviceProp.sharedMemPerBlock);
        printf("  Total number of registers available per block: %d\n",
               deviceProp.regsPerBlock);
        printf("  Warp size:                                     %d\n",
               deviceProp.warpSize);
        printf("  Maximum number of threads per block:           %d\n",
               deviceProp.maxThreadsPerBlock);
        printf("  Maximum sizes of each dimension of a block:    %d x %d x %d\n",
               deviceProp.maxThreadsDim[0],
               deviceProp.maxThreadsDim[1],
               deviceProp.maxThreadsDim[2]);
        printf("  Maximum sizes of each dimension of a grid:     %d x %d x %d\n",
               deviceProp.maxGridSize[0],
               deviceProp.maxGridSize[1],
               deviceProp.maxGridSize[2]);
        printf("  Maximum memory pitch:                          %u bytes\n",
               deviceProp.memPitch);
        printf("  Texture alignment:                             %u bytes\n",
               deviceProp.textureAlignment);
        printf("  Clock rate:                                    %.2f GHz\n",
               deviceProp.clockRate * 1e-6f);
    #if CUDART_VERSION >= 2000
        printf("  Concurrent copy and execution:                 %s\n",
               deviceProp.deviceOverlap ? "Yes" : "No");
    #endif
	
	}
		
	return GPU_ERROR;	
		
}
*/
