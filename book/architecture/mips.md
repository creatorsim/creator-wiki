# MIPS architecture


## Overview

[Reference Guide](https://creatorsim.github.io/creator/guides/mips32.pdf).


## System calls

| Service      | Trap Code  | Input                                    | Output                           | Notes |
|---	       |---         |---                                       |---                               |---    |
| print_int    | $v0 = 1    | $a0  = int    to be printed              | Print $a0  to display            |       |
| print_float  | $v0 = 2    | $f12 = float  to be printed              | Print $f12 to display            |       |
| print_double | $v0 = 3    | $f12 = double to be printed              | Print $f12 to display            |       |
| print_string | $v0 = 4    | $a0  = 1st char's address                | Print string in stardard output  |       |
| read_int     | $v0 = 5    |                                          | Read integer to $v0              |       |
| read_float   | $v0 = 6    |                                          | Read float   to $v0              |       |
| read_double  | $v0 = 7    |                                          | Read double  to $v0              |       |
| read_string  | $v0 = 8    | $a0 = buffer address, $a1= buffer length | Read string                      |       |
| sbrk         | $v0 = 9    | $a0 = number of bytes                    | $v0 points to the allocated memory | Allocation from heap |
| exit         | $v0 = 10   |                                          |                                    | End of execution |


