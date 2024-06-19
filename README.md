# Some tricks I've found over time

The contents of this readme is public domain where possible. And if licensing is required, it may be licensed under the MIT, NewBSD, ISC, Zlib, MIT0 or CC0 licenses.

## Useful single-file header libraries

Included are the URLs directly to the headers themselves.  You can "Copy Link Location" then `wget` or `curl` it into where you need it.

Single-file-header only libraries are intended to make it easy to add funcitonality to a C project without extra build steps and most of the time come with zero dependencies.

The pattern is typically:

In one .c file in your project:
```
#define PACKAGE_IMPLEMENTATION
#include "package.h"
```

If you need the header file to use it in other .c files in your project, just include the package, directly.

```
#include "package.h"
```

Note that some header-only libraries are actually .c files.  But that's ok, you can just `#include` the c file in your code.

| Name/Package/Page | Description | File/URL |
| --- | --- | --- |
| [stb](https://github.com/nothings/stb/) | Easily write .bmp, .png, .tga, .jpeg files from C | [stb_image_write.h](https://raw.githubusercontent.com/nothings/stb/master/stb_image_write.h) |
| [stb](https://github.com/nothings/stb/) | Load .ttf files and get raster pixel data | [stb_truetype.h](https://raw.githubusercontent.com/nothings/stb/master/stb_truetype.h) |
| [stb](https://github.com/nothings/stb/) | Load .png, .pnm, .jpg, .bmp, .psd, .tga, .gif, .hdr, and .pic files | [stb_truetype.h](https://raw.githubusercontent.com/nothings/stb/master/stb_image.h) |
| [olive.c](https://github.com/tsoding/olive.c) | Tiny graphics library for working on pixel buffers | [olive.c](https://raw.githubusercontent.com/tsoding/olive.c/master/olive.c) |

Single-file headers I wrote:

| Name/Package/Page | Description | File/URL |
| --- | --- | --- |
| [rawdraw](https://github.com/cnlohr/rawdraw) | Window / opengl context creation, input management on many platforms (like SDL but 1/1000th the size) good for Android, Linux, Windows, Web | [rawdraw_sf.h](https://raw.githubusercontent.com/cntools/rawdraw/master/rawdraw_sf.h) |
| [hufftreegen_sf](https://github.com/cnlohr/cntools) | Create huffman trees of bitstreams, and the means to decode them | [hufftreegen.h](https://raw.githubusercontent.com/cnlohr/cntools/master/hufftreegen_sf/hufftreegen.h) |
| [cnrbtree](https://github.com/cnlohr/cntools/tree/master/cnrbtree) | Arbitrary header-only (with macro magic) red-black tree | [cnrbtree.h](https://raw.githubusercontent.com/cnlohr/cntools/master/cnrbtree/cnrbtree.h) |
| [cnhash](https://github.com/cnlohr/cntools/tree/master/cnhash) | C Hash table/map implementation | [cnrbtree.h](https://raw.githubusercontent.com/cnlohr/cntools/master/cnhash/cnhash.h) |
| [mini-rv32ima](https://github.com/cnlohr/mini-rv32ima) | Header-only RISC-V Emulator capable of booting nommu linux | [mini-rv32ima.h](https://github.com/cnlohr/mini-rv32ima/blob/master/mini-rv32ima/mini-rv32ima.h) |
| [heatshrink-sfh](https://github.com/cnlohr/heatshrink-sfh) | Very stripped down implementation of the heatshrink LVSS decompression algorithm | [heatshrink_sf.h](https://raw.githubusercontent.com/cnlohr/heatshrink-sfh/master/heatshrink_sf.h) |
| [rtgz-tinf-util](https://github.com/cnlohr/rtgz-tinf-util) | Custom deflate decompressor, and custom compressor for targeting tiny decode windows | [tinf_sf.h](https://raw.githubusercontent.com/cnlohr/rtgz-tinf-util/master/tinf_sf.h) |
| [csgp4](https://github.com/cnlohr/csgp4) | SGP4 Orbital Mechanics Header-Only file (portable to HLSL) for computing satellite positions at times | [csgp4.h](https://raw.githubusercontent.com/cnlohr/csgp4/master/csgp4.h) |


## Embedded / C

### Fast Approximate Integer Square Root

First, get the approximate square root from the next power-of-2, then refine it.

```c
int apsqrt( int i )
{
	if( i == 0 ) return 0;
	int x = 1<<( ( 32 - __builtin_clz(i) )/2);
	x = (x + i/x)/2;
//	x = (x + i/x)/2; //Not really needed.
	return x;
}
```
![apsqrt](https://github.com/cnlohr/cnlohr_tricks/blob/master/media/apsqrt.png?raw=true)

### Fast Binary-Shift (no-multiply) IIR filter

If you need an IIR to compute a high or low-pass filter on your data, here is a solution that gives it to you with just a "running average" state.

This is an IIR filter, which means that as time goes on old values matter less and less.

You will need to tune IIR_AMOUNT to your liking, higher values will be more damped, lower values will be faster to respond.

For removing DC offset, or computing high-pass, take your value, and subtract the filtered value.

```c
	#define IIR_AMOUNT 8
	static int ifilt; // Accumulates the average.

	// Note, the ifilt average will be (1<<IIR_AMOUNT)*average, so
	// if IIR_AMOUNT is 8 and you are using values > 255, and ifilt
	// is a uint16_t, then, it will OVERFLOW.

	int v = rand() % 1000;
	ifilt = ifilt + v - (ifilt>>IIR_AMOUNT);


	int filtered_value = ifilt>>IIR_AMOUNT;
```
![ifilt](https://github.com/cnlohr/cnlohr_tricks/blob/master/media/ifilt.png?raw=true)

### Goertzel's sinewave / DFT.

A lot of times, you will want to use a sinewave, or need to do DFT for specific tone.

And sin/cos may not be fast.  Well, good news, goertzels is to the rescue.  You can
generate a sin/cos without any sin or cos!

Note that in the below sample, the power will be 0, because the sample is 1.0, so it
is treated as a "DC offset" and thus gets nulled out.

Also, goertzels does not need to compute an I and a Q to detect power.

Note that you can store the "coeff" in a table for lookup, it is fixed per the frequency you are observing.

```c
#include <stdio.h>
#include <math.h>

int main()
{
	const float omegaPerSample = 0.015708; // pi / 200
	const int numSamples = 400; // enough to go from 0 to 2pi

	float coeff = 2 * cos( omegaPerSample );
	int i;

	// TRICKY: When you want a sinewave, initialize with omegaPerSample.  This
	// is crucial.  The initial state will have massive consequences.
	float sprev = omegaPerSample;
	float sprev2 = 0;
	for( i = 0; i < numSamples; i++ )
	{
		// If you wanted to do a DFT, set SAMPLE to your incoming sample.
		float SAMPLE = 1.0;

		// Here is where the magic happens.
		float s = SAMPLE + coeff * sprev - sprev2;
		sprev2 = sprev;
		sprev = s;
		printf( "%f\n", s ); 
	}

	// For DFT, your power will be:
	float power = sprev*sprev + sprev2*sprev2 - (coeff * sprev * sprev2);
	printf( "%f\n", power );
}
```

![goertzels0](https://github.com/cnlohr/cnlohr_tricks/blob/master/media/goertzels0.png?raw=true)

Below is an example using Goertzel's Algorithm to extract the phase and magnitude of a sinewave of an incoming signal, against a specific target frequency. If you think this is magical, that makes two of us.

```c
#include <stdio.h>
#include <math.h>

int main()
{
	const float omegaPerSample = 0.015708; // pi / 200
	const int numSamples = 400; // enough to go from 0 to 2pi


	for( float phaseOffsetOfSignal = 0; phaseOffsetOfSignal < 3.1415926*2.0; phaseOffsetOfSignal += 0.01 )
	{
		float coeff = 2 * cos( omegaPerSample );
		int i;

		// TRICKY: When you want a sinewave, initialize with omegaPerSample.  This
		// is crucial.  The initial state will have massive consequences.
		float sprev = omegaPerSample;
		float sprev2 = 0;
		for( i = 0; i < numSamples; i++ )
		{
			// If you wanted to do a DFT, set SAMPLE to your incoming sample.
			float SAMPLE = sin( phaseOffsetOfSignal + i * omegaPerSample );

			// Here is where the magic happens.
			float s = SAMPLE + coeff * sprev - sprev2;
			sprev2 = sprev;
			sprev = s;
		}

		// For DFT, your power will be:
		float power = sprev*sprev + sprev2*sprev2 - (coeff * sprev * sprev2);
		//printf( "Power: %f\n", power );

		float coeff_s = 2 * sin( omegaPerSample );
		double rR = 0.5 * coeff   * sprev - sprev2;
		double rI = 0.5 * coeff_s * sprev;
		printf( "%f,%f\n", rR, rI);
	}
}
```

![goertzels1](https://github.com/cnlohr/cnlohr_tricks/blob/master/media/goertzels1.png?raw=true)

## Linux

### My program segfaulted, why?

Use the following command:

```sh
coredumpctl gdb
bt
```

To get a backtrace of the most recently crashed program.

### How to make system calls in C, when programming without a standard library.

You have to make a syscall, but you can use inline assembly to wrap it so that it doesn't have to spend time/effort getting the registers setup correctly.

Be sure to compile with: `gcc -Os -nostdlib -Wl,-e"start" -ffunction-sections -fdata-sections -flto -Wl,--gc-sections` for optimal behavior, also set `start` to be your start function.

```c
#include <asm/unistd.h>      // compile without -m32 for 64 bit call numbers

// In x86_64 only.


// call write( int fd, uint8_t ptr, int length )
asm volatile
(
    "syscall"
    : "=a" (ret)
    //                 EDI      RSI       RDX
    : "0"(__NR_write), "D"(/*fd*/fd), "S"(ptr), "d"(length)
    : "rcx", "r11", "memory"
);
```

Lastly don't forget to exit!
```c
asm volatile
(
    "syscall"
    : "=a" (r)
    //                 EDI  
    : "0"(__NR_exit), "D"(/*fd*/0), "S"(0), "d"(0)
    : "memory"
);
```

