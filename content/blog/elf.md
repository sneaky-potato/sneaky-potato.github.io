---
title: "Staring inside programs: ELF"
date: 2025-09-03
description: "deep dive into linux extensible and linkable format"
tags: ["computer"]
---

{{< lead >}}
To the level where hexadecimals appear
{{< /lead >}}

## ELF
**E**xtensible and **L**inkable **F**ormat is a file format used for executable binaries in the linux operating system.

However ELFs are used for more than just executables, they are used for storing object files as well as shared libraries.
You can look at a files raw hexadecimal bytes using `xxd` and we will use it on ELFs, let us have a look at the first line inside the raw bytes of `ls` executable.
```shell
$ xxd /bin/ls | head -1
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```
Performing this activity on a different ELF will generate similar result
```shell
$ xxd /bin/cat | head -1
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
```

But we know `ls` and `cat` are different programs, right?
Well these bytes are exact same because they happen to be the part of a 16 byte ELF header. And since both `ls` and `cat` are ELF executables, we see the same [magic bytes](https://en.wikipedia.org/wiki/List_of_file_signatures)

This can be verified by using a tool like `readelf` to summarize the header values for any ELF executable
```
$ readelf -h /bin/ls
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x61d0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          149360 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30
```

## An experiment
Let's try to directly modify the bytes of an elf file with our editor.
Consider this exercise

1. Write a minimal C (just a return 0 will do) file
2. Compile it into ELF executable using `gcc`
3. Look at the raw hexadecimal bytes of the ELFs using `xxd`
4. Change some bytes and reverse the process using `xxd -r` 

---

```c
// minimal.c
#include <stdio.h>

int main() {
    printf("hello world\n");
    return 0;
}
```

```bash
$ gcc -o minimal minimal.c
$ xxd -p minimal > minimal.dmp
```

### Look at raw dump

look at `minimal.dmp` and you will find the raw bytes of minimal executable

```bash
$ head -1 minimal.dmp
7f454c4602010100000000000000000003003e0001000000501000000000
```

We can see the magic bytes of the ELF

Now we find what is the  value for `"hello world\n"`  and try to search in the ELF

```bash
$ echo "hello world" | xxd -p
68656c6c6f20776f726c640a
```

Now use vim or any editor to search for this string (`68656c6c6f20776f726c64` ) in `minimal.dmp` and replace it with something else

![search_elf_file](/search_elf_file.png)

### Edit ELF

We have found the string in the dmp file, I shall change some letters in this string now. I shall change the hexadecimal for space (`20` ) with that of a hyphen (`2d` )

![edit_elf_file](/edit_elf_file.png)

Now reverse the dmp file back into executable using `xxd -r`

```bash
$ xxd -p -r minimal.dmp > minimal
$ ./minimal
hello-world
```
However this executable is quite heavy for our purposes

```bash
$ readelf -h minimal
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Position-Independent Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1050
  Start of program headers:          64 (bytes into file)
  Start of section headers:          13976 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         13
  Size of section headers:           64 (bytes)
  Number of section headers:         31
  Section header string table index: 30
```

It has 13 program headers and 31 section headers

we ideally want ELF with a 1 program header and 0 section header

### Script to automate

Write a `make.sh` script to take all `.dmp` files in the current working directory and reverse them into elf binaries

```bash
#!/bin/bash
# make.sh
for f in *.dmp ; do
    a=`basename $f .dmp`
    cut -d'#' -f1 <$f | xxd -p -r >$a # we will clean all lines starting with #
    # this allows us to write some comments in the dump file for explanation
    chmod +x $a
done
```

## Exit programs

Now, I shall attempt to do something sinister

Write the simplest exit program (which exit with status code 42) in 32 bit format. I've put all the comments to explain what each line means.
Comments will be cleaned by the `cut` command in the make script.

```bash
# exit32.dmp
#  -------------- ELF HEADER --------------
                # all numbers are in base 16
                # 00 number of bytes to be used in the ELF
7F 45 4C 46     # 04 e_ident[EI_MAG]: ELF magic bytes
01              # 05 e_ident[EI_CLASS]: 1=32-bit, 2=64-bit, for 32 bit addresses would be 4 byte long
   01           # 06 e_ident[EI_DATA]: 1=little-endian, 2=big=endian
      01        # 07 e_ident[EI_VERSION]: ELF header version
         00     # 08 e_ident[EI_OSABI]: Target OS ABI; should be 0 for System V
00              # 09 e_ident[EI_ABIVERSION]: ABI version; 0 is ok for Linux
   00 00 00     # 0C e_ident[EI_PAD]: unused, should be 0
00 00 00 00     # 10

02 00           # 12 e_type: object file type; 2=executable
      03 00     # 14 e_machine: instruction set architecture; 03=x86, 3E=amd64
01 00 00 00     # 18 e_version: ELF identification version; must be 1
54 80 04 08     # 1C e_entry: memory address of entry point (where process starts)
34 00 00 00     # 20 e_phoff: file offset where program headers begin

00 00 00 00     # 24 e_shoff: file offset where section headers begin
00 00 00 00     # 28 e_flags: 0 for x86

34 00           # 2A e_ehsize: size of this header (34: 32-bit, 40: 64-bit)
      20 00     # 2C e_phentsize: size of each program header (20: 32-bit, 38: 64-bit)
01 00           # 2E e_phnum: #program headers
      28 00     # 30 e_shentsize: size of each section header (28: 32-bit, 40: 64-bit)

00 00           # 32 e_shnum: #section headers
      00 00     # 34 e_shstrndx: index of section header containing section names

#  ------------ PROGRAM HEADER ------------

01 00 00 00     # 38 p_type: segment type; 1: loadable

54 00 00 00     # 3C p_offset: file offset where segment begins
54 80 04 08     # 40 p_vaddr: virtual address of segment in memory (x86: 08048054)
    
00 00 00 00     # 44 p_paddr: physical address of segment, unspecified by 386 supplement
0C 00 00 00     # 48 p_filesz: size in bytes of the segment in the file image ############

0C 00 00 00     # 4C p_memsz: size in bytes of the segment in memory; p_filesz <= p_memsz
05 00 00 00     # 50 p_flags: segment-dependent flags (1: X, 2: W, 4: R)

00 10 00 00     # 54 p_align: 1000 for x86

#  ------------ PROGRAM SEGMENT ------------

B8 01 00 00 00  # 59 eax <- 1 (exit)
BB 2A 00 00 00  # 5E ebx <- 2A (param)
CD 80           # 60 syscall >> int 80

```

Recompile back to binary using `make.sh`  and run it

```bash
$ chmod +x make.sh
$ ./make.sh
$ ./exit32
$ echo $?
42
```

Success

Now write a similar ELF but this time 64 bit

```bash
# exit64.dmp
#  -------------- ELF HEADER --------------
                           # all numbers are in base 16
                           # 00 number of bytes to be used in the ELF
7F 45 4C 46                # 04 e_ident[EI_MAG]: ELF magic bytes
02                         # 05 e_ident[EI_CLASS]: 1=32-bit, 2=64-bit, for 32 bit addresses would be 4 byte long
   01                      # 06 e_ident[EI_DATA]: 1=little-endian, 2=big=endian
      01                   # 07 e_ident[EI_VERSION]: ELF header version
         00                # 08 e_ident[EI_OSABI]: Target OS ABI; should be 0 for System V
00                         # 09 e_ident[EI_ABIVERSION]: ABI version; 0 is ok for Linux
   00 00 00                # 0C e_identpEI_PAD]: unused, should be 0 
00 00 00 00                # 10

02 00                      # 12 e_type: object file type; 1=relocable, 2=executable
      3E 00                # 14 e_machine: instruction set architecture; 03=x86, 3E=amd64(x86-64)
01 00 00 00                # 18 e_version: ELF identification version; must be 1
78 00 40 00 00 00 00 00    # 20 e_entry: memory address of entry point (where process starts)
40 00 00 00 00 00 00 00    # 28 e_phoff: file offset where program headers begin: start immediately after ELF header

00 00 00 00 00 00 00 00    # 30 e_shoff: file offset where section headers begin

00 00 00 00                # 34 e_flags: 0 for x86

40 00                      # 36 e_ehsize: size of this header (34: 32-bit, 40: 64-bit)
      38 00                # 38 e_phentsize: size of each program header (20: 32-bit, 38: 64-bit)
01 00                      # 3A e_phnum: #program headers
      40 00                # 3C e_shentsize: size of each section header (28: 32-bit, 40: 64-bit)

00 00                      # 3E e_shnum: #section headers
      00 00                # 40 e_shstrndx: index of section header containing section names

#  ------------ PROGRAM HEADER ------------

01 00 00 00                # 44 p_type: segment type; 1: loadable
05 00 00 00                # 48 p_flags: segment-dependent flags (1: X, 2: W, 4: R)

78 00 00 00 00 00 00 00    # 50 p_offset: ELF header size + this program header size
78 00 40 00 00 00 00 00    # 58 p_vaddr: 0x40000 + offset)
00 00 00 00 00 00 00 00    # 60 p_paddr: irrelevant for x86_64

10 00 00 00 00 00 00 00    # 68 p_filesz:  (just count the bytes for your machine instructions)

10 00 00 00 00 00 00 00    # 70 p_memsz: (if this is greater than file size, then it zeros out the extra memory)

00 10 00 00 00 00 00 00    # 78 p_align: 1000 for x86

#  ------------ PROGRAM SEGMENT -----------

48 C7 C0 3C 00 00 00       # 7F mov rax, 60 (0x3C)
48 C7 C7 2A 00 00 00       # 86 mov rdi, 42 (0x2A)
0F 05                      # 88 syscall >> int 80

```

Repeat the previous steps and it should work as well

## Moment of truth

Check the final output from readelf and see how well you did

```bash
$ readelf -h exit32
ELF Header:
  Magic:   7f 45 4c 46 01 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF32
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Intel 80386
  Version:                           0x1
  Entry point address:               0x8048054
  Start of program headers:          52 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               52 (bytes)
  Size of program headers:           32 (bytes)
  Number of program headers:         1
  Size of section headers:           40 (bytes)
  Number of section headers:         0
  Section header string table index: 0
```

```bash
$ readelf -h exit64
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x400078
  Start of program headers:          64 (bytes into file)
  Start of section headers:          0 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         1
  Size of section headers:           64 (bytes)
  Number of section headers:         0
  Section header string table index: 0
```

So we just wrote raw ELF hexadecimal values and were able to run it as well, how cool is that?
