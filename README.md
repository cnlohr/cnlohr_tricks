# Some tricks I've found over time


## Linux

### My program segfaulted, why?

Use the following command:

```sh
coredumpctl gdb
bt
```

To get a backtrace of the most recently crashed program.

## Embedded / C

### Fast Approximate Integer Square Root

First, get the approximate square root from the next power-of-2, then refine it.

```c
int apsqrt( int i )
{
    if( i == 0 ) return 0;
    int x = 1<<( ( 32 - __builtin_clz(i) )/2);
    x = (x + i/x)/2;
//    x = (x + i/x)/2; //Not really needed.
    return x;
}
```
![apsqrt](https://github.com/cnlohr/cnlohr_tricks/blob/master/media/apsqrt.png?raw=true)

### Fast Binary-Shift (no-multiply) IIR filter

If you need an IIR to compute a high or low-pass filter on your data, here is a solution that gives it to you with just a "running average" state.

This is an IIR filter, which means that as time goes on old values matter less and less.

You will need to tune IIR_AMOUNT to your liking, higher values will be more damped, lower values will be faster to respond.

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


