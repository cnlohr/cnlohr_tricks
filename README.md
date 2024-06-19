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
