# Writing a Bootloader

In this assignment, you are going to write a program that loads an S-record file into memory and jumps to the first address in the S-record.
There are many different file formats that can be used to store a program on disk.
Whatever format we use, we need to be able to store the instruction encodings for the program.
Mostly file formats used in real computers have some complicated data structures that also store symbol tables (addresses of functions and variables) and a bunch of other junk.
In this assignment, we are going to be loading a very simple data structure called an S-record.


## S-Records

S-record is a file format for storing compiled programs.
They contain the binary instruction encodings of a program represented in hexadecimal format.
Hex files are convenient because they are human readable.
We can use a text editor to look directly at the contents of the file (no special tools required).
The file format is super simple--no weird headers or anything.

### Example S-Record

I have written a simple 'Hello World' program below and annotated each instruction with its hexadecimal encoding in the comments.
When we compile this program, we need to save (1) the instruction encodings and (2) the ASCII encodings for the characters in the string.
We will load all the binary values into memory when we execute the program.

        ORG    $1000
    START:    
        LEA HELLOMSG,A1          ; 43F9 0000 100C
        MOVE.L #14,D0            ; 700E
        TRAP #15                 ; 4E4F
        BRA START                ; 60F4

    HELLOMSG:
        DC.B 'HELLO WORLD',0
        END    START

Below I have pasted the S-record generated for this program:

    S021000036384B50524F47202020323043524541544544204259204541535936384B6D
    S11B100043F90000100C700E4E4F60F448454C4C4F20574F524C4400F1
    S804001000EB

Each line in the S-record serves a different purpose.
In this program, we have three lines: one that starts with `S0`, one that starts with `S1` and one that starts with `S8`.

1. `S02100003638...`
The S0 line is a comment line that is used to store the name of the program (in ASCII).
This line contains the text `68KPROG   20CREATED BY EASY68K`.
Your bootloader can ignore `S0` lines.
2. The S1 line contains loadable data.
It is split into several fields:

`S1`   `1B`   `1000`   `43F90000100C700E4E4F60F448454C4C4F20574F524C4400` `F1`
* `S1` - Line type, indicating data line
* `1B` - Number of data bytes encoded in this line.
* `1000` - The load address of the first byte in this line. In the program, we used the `ORG $1000` directive to tell Easy68k that we want our program to load at address 0x1000.
* `43F90000...` - The raw data of our program (instructions and data). Note that 0x43F9 is the beginning of the instruction encoding of the `LEA` instruction.
* `F1` - A one-byte checksum of the program data.
3. The S8 line indicates the end of the S-record file.
Your bootloader can ignore `S8` lines.

The first address in this S-record is `0x1000`.
For this homework assignment, after we load this S-record into memory, we would jump to address `0x1000` to start running the program.


## Reading a File in Easy68k

In Easy68k, reading from a file is a two-step process.
First, we need to open the file, second we read bytes from the file into memory.
In Easy68k, file reads are all block-wise reads, meaning that data is read from a file on disk into a buffer in large blocks.
We normally don't read just one byte at a time (although you can if you set the block size to 1).

### Opening a File

You can use the `TRAP #15` instruction to open a file.
You should probably encapsulate your file opening code inside a function.


    A1 - Path to the file
    D0 - 51 decimal (file open task number)

    Return - D1 has file ID

### Reading from a File

You can use the `TRAP #15` instruction to read from a file.
You should probably encapsulate your file reading code inside a function.


    A1 - Pointer to the buffer
    D0 - 53 decimal (file read task number)
    D1 - File ID (returned by open)
    D2 - Number of bytes to read

## Converting from Hex to Binary

You can't directly load and execute the contents of the S-record.
The hexadecimal S-record needs to converted to binary before loading it into memory and running it.
We do this by converting each one-byte ASCII (hex) character to binary at a time.

For the digits (0-9), this is easy: just subtract the encoding for `0`.
Example: to convert ASCII 4 to binary 4: `0x34` - `0x30` = `0x04`.
However, if we have letters (`A` - `F`), we need to convert ASCII `A` to binary `A`.
ASCII `A` is encoded by `0x41`.
If we just subtract the encoding for ASCII 0, we get:  `0x41` - `0x30` = `0x11` --- not what we wanted.
We need to subtract 7 more in order to get binary `0x0A`: `0x41` - `0x30` - `0x07` = `0x0A`.

| Hex Encoding  | ASCII Character |
|---------------|-----------------|
|      `30`     |        0        |
|      `31`     |        1        |
|      `32`     |        2        |
|      `33`     |        3        |
|      `34`     |        4        |
|      `35`     |        5        |
|      `36`     |        6        |
|      `37`     |        7        |
|      `38`     |        8        |
|      `39`     |        9        |


You should write a function that converts a pair of hex nibbles to a binary byte.

## Suggested Approach

1. Write a function that converts a hex string to a binary integer. This was Homework 6. You may want to modify your submission for Homework 6 so that one of its parameters is the number of hex characters to convert to binary.
2. Write a function that opens a file.
3. Write a function that reads from a file into a buffer in memory.
4. Build a state machine that parses S-records. Tips:

* Start this by drawing your state transition diagram out on paper.
* Give every state a meaningful name. For example, the states in my FSM are called LOOK_FOR_S, LOOK_FOR_RECORD_TYPE, LOOK_FOR_NBYTES, etc.
* You need to convert the address and data fields of the S-record from hex to binary. The address field of an `S1` line is 2 bytes (4 nibbles) long. The data field of the S-record should be converted one byte at a time. Each byte of data shoud be converted to binary and saved into memory at the address specified.


I have written code for an example state machine in 68k assembly below.


    ; STATE DEFINITIONS. EACH STATE GETS ASSIGNED A UNIQUE INTEGER IDENTIFIER.
    STATE_INIT    EQU 0
    STATE_1       EQU 1
    STATE_2       EQU 2
    ; ETC...
    
    ORG    $1000
    START:                  ; first instruction of program
    
    ; Open S-rec file, read file contents into buffer...
    
    ; Below is the code for implementing the FSM:
    PROCESS_FSM:
        CMPI.L #STATE_INIT, state
        BEQ PROCESS_STATE_INIT
        CMPI.L #STATE_1, state
        BEQ PROCESS_STATE_1
        CMPI.L #STATE_2, state
        BEQ PROCESS_STATE_2
        ; ETC...

    PROCESS_STATE_INIT:
        ; PUT YOUR CODE IN HERE TO DO WHAT NEEDS TO BE DONE IN STATE_INIT.
        ; UPDATE THE state VARIABLE TO GO TO THE NEXT STATE.
        MOVE.L #STATE_1,state
        BRA PROCESS_FSM

    PROCESS_STATE_1:
        ; PUT YOUR CODE IN HERE TO DO WHAT NEEDS TO BE DONE IN STATE_1.
        ; UPDATE THE state VARIABLE TO GO TO THE NEXT STATE.
        MOVE.L #STATE_2,state
        BRA PROCESS_FSM
    
    PROCESS_STATE_2:
        ; PUT YOUR CODE IN HERE TO DO WHAT NEEDS TO BE DONE IN STATE_2.
        ; UPDATE THE state VARIABLE TO GO TO THE NEXT STATE.
        MOVE.L #STATE_1,state
        BRA PROCESS_FSM

        SIMHALT             ; halt simulator

    state:
        dc.l STATE_INIT
        END    START        ; last line of source



## Grading

|              Task                                     |   Points   |
|-------------------------------------------------------|------------|
| `hex2int` Working                                     |     5      |
| File open Working                                     |     5      |
| File read Working                                     |     5      |
| FSM Ignores `S0` and `S8` lines                       |     5      |
| FSM Parses the correct address from `S1` lines        |     10     |
| FSM reads data from `S1` lines and places it into mem |     20     |




