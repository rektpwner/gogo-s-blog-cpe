---
title: "Episode 2: guessing source-code (reverse engineering) of an cryptographic algorithm to break it"
date: 2024-10-12T23:00:00+02:00
weight: 2
draft: false
---

### reverse engineering/decompilation, mathematical analysis, and exploitation of Vigenere Cisco algorithm

## Identify:

In this course we are going to learn what to do when we do not have the source code of a program but when we need to still read the source code in order to find vulnerabilities that the developpers have let.

You will have to use this code reading technic (reverse engineering) to audit a proprietary hashing algorithm known as "Vigenere Cisco".

Trought the main goal of another CPE article is to provide a mathematical proof of the lack of security of Vigenere Cisco, this article focus on how to extract the exact source code (decompilation) of the viginere Cisco hash algorithm present in the routers. 

In this tutorial, we will see how to decompile Packet Tracer to read the source code of a vulnerable hash algorithm to then break it.


## I - Guessing vigenere cisco hashing algorithm (decompilation)

## 1: Downloading the audited program:

As the same output in the real life cisco routers as what we could see in `packet tracer`. Then I suggest to decompile Packet Tracer from internet instead of openning router hardware.

Let's read the Packet Tracer source code!

Download Packet Tracer from: https://skillsforall.com/resources/lab-downloads

```bash
sudo dpkg -i Packet_Tracer821_amd64_signed_ab5472da4e.deb

$ which packettracer
/usr/local/bin/packettracer
```

Find the file `PacketTracer` with: `ls /opt/pt/bin`.

The file `packettracer` provides running commands:

```
$ cat /usr/local/bin/packettracer
#!/bin/bash

echo Starting Packet Tracer 8.2.1

PTDIR=/opt/pt
export LD_LIBRARY_PATH=/opt/pt/bin
pushd /opt/pt/bin > /dev/null
./PacketTracer "$@" > /dev/null 2>&1
popd > /dev/null
```

Run the command `packettracer`. Create a virtual router and then run the following commands:

```
Would you like to enter the initial configuration dialog? [yes/no]: n


Press RETURN to get started!



Router>
Router>enable
Router#configure terminal
Enter configuration commands, one per line.  End with CNTL/Z.
Router(config)#enable password mon_mot_de_passe
Router(config)#line vty 0 4
Router(config-line)#password mon_mot_de_passe
Router(config-line)#login
Router(config-line)#
Router(config-line)#service password-encryption
Router(config)#exit
Router#show run | include password
%SYS-5-CONFIG_I: Configured from console by console

service password-encryption
enable password 7 082C434036140A032D0F093B3A2A373B36
 password 7 082C434036140A032D0F093B3A2A373B36
Router#
```

In any cisco router you could run as well. It fits with the output of the program.

In order to redo same commands as me you could also download it from my repository

```
wget https://github.com/gogo2464/packet-tracer-linux-8.2.1.0118-archive/releases/download/packet-tracer-linux-8.2.1.0118/pt.tar
tar -xvf pt.tar
cd pt.tar
cd pt
```


## II - Reversing the checksum, method by attacking the logic behind the algorithm

Our goal is to read the logic of the cryptography algorithm to then code a program with a logic that reverses it.

## 1: static analysis: looking for symbols

The next command allow you to read the Packet Tracer program disassembly.

```
PTDIR="$(pwd)/pt/"
export LD_LIBRARY_PATH="$(pwd)/pt/bin/"
radare2 -a x86 -b 64 -A ./pt/bin/PacketTracer
```

### 2 static analysis: looking for symbols

In a terminal, run:

```
PTDIR="$(pwd)/pt/"
export LD_LIBRARY_PATH="$(pwd)/pt/bin/"
radare2 -a x86 -b 64 -A ./pt/bin/PacketTracer
```

Developpers always need to debug their code to fix bug. Sometimes, the developpers ignore or forget to remove the "debug mode" of their compilation tool. This situation allow a reverse engineer to make reverse engineering more easily.

Packet Tracer contains symbols and then can be reversed more easily.

```
> f~LineCon.password_type7
0x019396c0 1 method.CommandSet::Common::LineCon.password_type7_std::vector_std::__cxx11::basic_string_char__std::char_traits_char___std::allocator_char_____std::allocator_std::__cxx11::basic_string_char__std::char_traits_char___std::allocator_char_________CommandSet:
```

At this moment, let's jump to `0x019396c0` to analyze the code of the command related to `type7` hashing:

```
s method.CommandSet::Common::LineCon.password_type7_std::vector_std::__cxx11::basic_string_char__std::char_traits_char___std::allocator_char_____std::allocator_std::__cxx11::basic_string_char__std::char_traits_char___std::allocator_char_________CommandSet:
```

```
pd 895 @ sym.Util::encryptType7_char_const__char__unsigned_int_
```

Let's move directly to 

```
radare2 -a x86 -b 64 -s sym.Util::encryptType7_char_const__char__unsigned_int_ /opt/pt/bin/PacketTracer
```

### 2.3: Looking to with the help of dynamic analysis

### 2.3.1: Obtain the hardcoded password

```bash
PTDIR="$(pwd)/pt/"
export LD_LIBRARY_PATH="$(pwd)/pt/bin/"
gdbserver localhost:2345 pt/bin/PacketTracer
```

From another terminal:
 
```
radare2 -a x86 -b 64 -c "dc" -s sym.Util::encryptType7_char_const__char__unsigned_int_ -e dbg.exe.path=pt/bin/PacketTracer -d gdb://localhost:2345
```

Now from Packet Tracer GUI, open a terminal in a router and type:

```
enable
configure terminal
enable password mon_mot_de_passe
line vty 0 4
password mon_mot_de_passe
login

service password-encryption
exit
show run | include password
```

Then I type `:` to open the console and I run the command:

```bash
VV
```

We could realize there is a graph that represent the if/else conditions and loops (control flow graph) of the program.

According to the graph, [there is the corresponding source code reverse engineered](https://codeberg.org/gogo/viegenere-cisco-proprietary-algorythm-decompiled-from-packet-tracer)

From this point we could also put breakpoint in the graph with the command `db 0x12345678` if you replace `0x12345678` with the place in the program (offset) where is located the line that you would like to debug to see how the function works.

Vigenere Cisco algorithm has two main weaknesses:
- The algorithm does not compare two changed values (hashed) together but quickly reverse it own algorithm to compare 2 unchanged texts.
- The algorithm contains a password in it own readable algorithm to change and revert to the changes in its own algorithm. This is a weakness. Ensure by checking `Kerckhoffs` principle online.

- we could also guess the plain password from the hashed password. At this point the third issue is a details because the two other points are enough to reverse the algorithm.

The logic behind how to reverse the algorithm is not intuitive. You will need to read an article [there to see how to bypass the logic behind the guessed source code of the program](https://gogo2464.github.io/gogo-s-blog-cpe/from-0-to-crypto-by-projects/episode-2-proof-demonstration/).


## III - Reversing the result (checksum), method with only reverse engineering


We could see where the password is used:

```bash
/r method.Util.decryptType7_char_const__char__unsigned_int_

...
method.CommandSet::CTerminalLine.getDecryptedPassword_abi:cxx11____const 0x2f242ca [CALL] call 0x38fdede
method.Device::CCiscoDevice.getPasswordOfUser_std::__cxx11::basic_string_char__std::char_traits_char___std::allocator_char____ 0x2ff8d2f [CALL] sub dword [rax], 1
```

It seems that this is where the checksum is compared.

### 1 - Understanding the code

While looking for in the code we found an unbelievable finding. After executing the command: `pd 310 @ sym.Util::decryptType7_char_const__char__unsigned_int_`.

With the commands:
```
f~decryptType7
0x038a9d90 310 sym.Util::decryptType7_char_const__char__unsigned_int_
0x038a9d90 1 method.Util.decryptType7_char_const__char__unsigned_int_
```

We realize that the function does not compare two hashes together but reverse itself the hash and then compare 2 unmodified text values (plain text). We are now going to rewrite the hashing function, decompile it, recompile it in C.

The program seems to have been coded by hand in assembly language or maybe has been modified by automatic tools after compilation to make the reading harder to copyright infringers.

We are then going to position at the position in the file (offset) at `0x038a9e04` bytes from the start of the file and to define there in order to allow radare2 to print a graph with the command: `af @ 0x038a9e04`. We are now in capacity to understand the flow with the command `VV` at this offset.

Let's rewrite the function with name `method.Util.decryptType7_char_const__char__unsigned_int_` at offset `0x038a9e04` in C.

After a raw conversion from assembly to C, the exact assembly logic is:

```c
#include <cstring>
#include <ctype.h>
#include <cstdio>

char x_lat[41] = "dsfd;kfoA,.iyewrkldJKDHSUBsgvca69834ncxv";

int decrypt_with_vigenere(const char * hash, char * out_original_password, int size) {
    /* Reverse the vigenere cisco hash using the decompilation of Packet Tracer.
     *
     * Returns 0 (error) if the hash size is 1 character.
     *
     *
     *
     * Argument 1, hash: input of the hash that needs to be reversed.
     * Argument 2, out_original_password: reversed password output.
     * Argument 3, size: size in bytes of the output password from the variable out_original_password. 
     *
     * Returns: 0 in case of error:
     *  - Returns 0 (error) if the hash size is 1 character.
     *  - Returns 0 (error) if the second character - 0x30 has the value 9.
     * Returns 1 in case of success.
    */
    
    

    int s = strlen(hash);
    
    if (s == 1) {
        return 0;
    }
    
    if ((hash[1] - 0x30) > 0x9) { // strcpy(var, atoi(hash+1)); if (var > 0x9) {} // or if (hash[1] > '8') {}
        return 0;
    }
    
    if ((hash[0] - 0x30) > 0x9) {
        return 0;
    }
    
    
    
    if (((hash[1] + ((hash[0]*5)*2)) - 528) > 0xf ) {
        return 0;
    }
    
    /*if ((hash[1] + (((hash[1] - 0x30) + (hash[1] - 0x30)) * 4 ) * 2 - 0x210 ) > 0xf ) {
        return 0;
    }*/
    
    
    
    int edx = hash[1];
    
    int esi = edx - 0x30;
    
    if (esi > 9) {
        return 0;
    }
    
    
    
    int edi = hash[0] - 0x30;
    
    esi = esi + esi * 4;
    esi = edx + esi * 2;
    int r15 = edx + esi * 2;
    r15 -= 0x210;
    
    r15 += 1;
    r15 = r15 >> 1;
    r15 -= 2;
    
    
    if (r15 >= 0xf) {
        return 0;
    }
    
    
    r15 = 8;
    
    
    s ++; // weird
    s = s >> 1;
    s -= 2;
    if (s > size) {
        return 0;
    }
    
    s = strlen(hash);
    
    if (s < 2) {
        out_original_password[size] = '\0';
        return 1;
    }
    
    char char_output = '\0';
    char mysterious_variable2;
    char mysterious_variable3 = r15;
    
    for (int i = 2, j = 8, k = 0, o = 0; i < s + 1; i++, k ++, o ++) {
       if (i != 2) {
            if ((i & 1) == 0) { //OOP: i % 2 == 0 ?
                char_output ^= (x_lat[j]);
                out_original_password[((i+1)>>1)-2] = char_output;
                char_output = 0;
                
                j ++;
            }
        }
        
        char_output = char_output << 4;
        mysterious_variable2 = toupper(hash[i]) - 0x30;
        
        if (mysterious_variable2 <= 0x9) {// if toupper(hash[i]) - 0x30 <= '9'
            char_output += mysterious_variable2;
        } else {
            mysterious_variable3 = (hash[i] - 0x41);
            if ((mysterious_variable3 > 0x5)) { // if (hash[i] == 'E') {}
                if (i != s) {
                    return 0;
                }
            } else {
                char_output += hash[i];
                char_output -= 0x37;
            }
        }
    }
    
    out_original_password[s] = '\0';//[((s+1) >> 2) - 2] = '\0';
    return 1;
}
```

Let's refactor the code with real variable names and clean structures such as: if/while/for/etc.. (control flow).

We now are able to clean the code. It outs mainly that the first code part is always equals to 8. We are then now refactoring the source code with register `r15` in favor of `j`.

```c
r15 = 8;
```

Becomes `j = 8` in:

```
for (int i = 2, j = 8, k = 0, o = 0; i < s + 1; i++, k ++, o ++) {
```

In the same way that following the code logic:

```c
    if (((s+1)>>1)-2) > size) {
        return 0;
    }
```

Becomes:

```c
    s ++; // weird
    s = s >> 1;
    s -= 2;
```


We are now going to proceed to a sementical analysis in order to refactor deeper the code. For this, let's understand deeply the code:

The next line:

```c
if ((i & 1) == 0) {
```

Seems to serve to return 0 one time on 2. It is a kind of half counter that is true only 1/2 times when the while is iterating. This comes either from an optimization by the compiler, from an obfuscation by an anti-piracy/industrial espionage/reverse engineering method in general or simply from an inexperienced developer who would have coded this function manually in assembly. This is an unnecessarily complicated logic to read for developers and cryptanalysts. We will then replace it with a modulo such as:

```
if ((i % 2) == 0) {
```

There subsist trough the risk that the code logic is altered by the reverse engineeing refactoring. Let's create some unit tests:


### 2.3.1 Coding Unit tests:

```c
#include <stdio.h>
#include <stdlib.h>
#include <cstring>
#include "../includes/cisco_decryptor.h"

int main() {
    char *hash = "08701E1D5D4C53404A52";
    char *password = (char *) malloc (strlen(hash)/2);
    if (decrypt_with_vigenere(hash, password, 0x400) != 1) {
        printf("error\n");
    }
    
    printf("original password: %s\n", password);
    
    char *hash2 = "08701E1D5D4C53404A5254537C7E707B616472465243";
    char *password2 = (char *) malloc (strlen(hash2)/2);
    if (decrypt_with_vigenere(hash2, password2, 0x400) != 1) {
        printf("error\n");
    }
    
    printf("original password2: %s\n", password2);
    
    
    char *hash3 = "0829494205163A0E1D1E";
    char *password3 = (char *) malloc (strlen(hash3)/2);
    if (decrypt_with_vigenere(hash3, password3, 0x400) != 1) {
        printf("error\n");
    }
    
    printf("original password3: %s\n", password3);
    return 0;
}
```
main.c

```
$ ./src/main 
original password: 123456789
original password2: 123456789876543210555
original password3: hello_you
```

Excellent! Since we have this output, we know that the code has not been altered by the analysis. Let's continue!


### 2.4.2 refactoring `((i+1)>>1)-2`

Let's rewrite `((i+1)>>1)-2` to more readable way.


An mysterious iterator `((i+1)>>1)-2` is present at the line:

```c
out_original_password[((i+1)>>1)-2] = char_output;
```

Thanks to debugging, we know that it corresponds to the iteration between 0 to the full checksum size typed.

It calculate `i` and iterate from 0 starting from the 3 of `i`. We are then now going to replace it by:

```c
out_original_password[o] = char_output;
```

As:

```c
    int o = 0;
    
    for (int i = 2, j = 8, k = 0; i < s + 1; i++) {
       if (i != 2) {
            if ((i % 2) == 0) {
                char_output ^= x_lat[j];
                out_original_password[o] = char_output;
                char_output = 0;
                
                j ++;
                o ++;
            }
        }
```

Decompiled line by line the code is:

```c
mysterious_variable3 = (hash[i] - 0x41);
if ((mysterious_variable3 > 0x5)) {
```


This is then a code verification to ensure that the entry hash `hash[i]` is superior to `0x46` that provide F for a computer when the computer compares a number of characters (in ascii).

It then means that the program verifies that the hash has characters in the interval `0123456789abcdef`, then any characters in the previous list (in hexadecimal) because the `type7` cannot contain non-hexadecimal characters. Inputting an entry outside of these values would trigger an error and the program exits in such case.

I am also removing the:
```c
	if (i != s) {
	    return 0;
	}
```

This code is useless because the buffer always has the same size...

We still could split this code and the loop in two functions:

```c
    char char_output = '\0';
    char mysterious_variable2;
    int o = 0;
    
    for (int i = 2, j = 8, k = 0; i < s + 1; i++) {
        if (i == 2) {
            char_output = char_output << 4;
        
            mysterious_variable2 = toupper(hash[i]) - 0x30;
            
            if (mysterious_variable2 <= 0x9) {
                char_output += mysterious_variable2;
            }
            
            if (mysterious_variable2 > 0x9 && (hash[i] > 'F')) {
                return 0;
            }
            
            if (mysterious_variable2 > 0x9 && (hash[i] <= 'F')) {
	            char_output += hash[i];
	            char_output -= 0x37;
            }
        } else if (i != 2) {
        
            if ((i % 2) == 0) {
                char_output ^= x_lat[j];
                out_original_password[o] = char_output;
                char_output = 0;
                
                j ++;
                o ++;
            }
            
            char_output = char_output << 4;
            
            mysterious_variable2 = toupper(hash[i]) - 0x30;
            
            if (mysterious_variable2 <= 0x9) {
                char_output += mysterious_variable2;
            }
            
            if (mysterious_variable2 > 0x9 && (hash[i] > 'F')) {
                return 0;
            }
            
            if (mysterious_variable2 > 0x9 && (hash[i] <= 'F')) {
	            char_output += hash[i];
	            char_output -= 0x37;
            }
        }
    }
```

The code becomes more readable!!

After analysis, the code part `mysterious_variable2 = toupper(hash[i]) - 0x30;` achieves to convert ascii to hexadecimal. A bit like `atoi`. We are now going to rename `mysterious_variable2` to `hexified_hash_ascii`.

I was not really understanding easily the behavior of  `out_original_password[((s+1) >> 2) - 2]` and I was not able to make it run. 

Anyway, from the global program internals, I see I should replace it with `out_original_password[o] = '\0';`.

`if ((hash[1] - 0x30) > 0x9) {` becomes `if (hash[1] > '9')`. And `if ((hash[0] - 0x30) > 0x9) {` becomes `if (hash[1] > '9')`.

We remove `s = strlen(hash);`.

And... We finally have:

```c
#include <cstring>
#include <ctype.h>
#include <cstdio>

char x_lat[41] = "dsfd;kfoA,.iyewrkldJKDHSUBsgvca69834ncxv";

int decrypt_with_vigenere(const char * hash, char * out_original_password, int size) {
    /* Reverse the vigenere cisco hash using the decompilation of Packet Tracer.
     *
     * Shift left to 4 each character, then converts each ascii hash to hexadecimal number. If the result is a number between 0-9, then the number is added to the output buffer. Else, we add this number and remove 7. 1 time on 2, the result is xored with the `x_lat` plain text password.
     *
     * Argument 1, hash: input of the hash that needs to be reversed.
     * Argument 2, out_original_password: reversed password output.
     * Argument 3, size: size in bytes of the output password from the variable out_original_password. 
     *
     * Returns: 0 in case of error:
     *  - Returns 0 (error) if the hash size is 1 character.
     *  - Returns 0 (error) if the first character is a letter.
     *  - Returns 0 (error) if the second character is a letter.
     *  - Returns 0 (error) if the hash is not an even number.
     * Returns 1 in case of success.
    */

    int s = strlen(hash);
    
    if (s == 1) {
        return 0;
    }
    
    if (hash[0] > '9') {
        return 0;
    }
    
    if (hash[1] > '9') {
        return 0;
    }
    
    if (((hash[1] + ((hash[0]*5)*2)) - 528) > 0xf) {
        return 0;
    }
    
    if ((((s+1)>>1)-2) > size) {
        return 0;
    }
    
    char char_output = '\0';
    char hexified_ascii_hash;
    int o = 0;
    
    for (int i = 2, j = 8; i < (s + 1); i++) {
        if (i == 2) {
            char_output = char_output << 4;
        
            hexified_ascii_hash = toupper(hash[i]) - 0x30;
            
            if (hexified_ascii_hash <= 0x9) {
                char_output += hexified_ascii_hash;
            }
            
            if (hexified_ascii_hash > 0x9 && (hash[i] > 'F')) {
                return 0;
            }
            
            if (hexified_ascii_hash > 0x9 && (hash[i] <= 'F')) {
	            char_output += hash[i];
	            char_output -= 0x37;
            }
        } else if (i != 2) {
        
            if ((i % 2) == 0) {
                char_output ^= x_lat[j];
                out_original_password[o] = char_output;
                char_output = 0;
                
                j ++;
                o ++;
            }
            
            char_output = char_output << 4;
            hexified_ascii_hash = toupper(hash[i]) - 0x30;
            
            if (hexified_ascii_hash <= 0x9) {
                char_output += hexified_ascii_hash;
            }
            
            if (hexified_ascii_hash > 0x9 && (hash[i] > 'F')) {
                return 0;
            }
            
            if (hexified_ascii_hash > 0x9 && (hash[i] <= 'F')) {
	            char_output += hash[i];
	            char_output -= 0x37;
            }
        }
    }
    
    out_original_password[o] = '\0';
    return 1;
}
```

Excellent! We are now able to study it! We are going to see the mathematical implication that follows that in [the next article that focuses on mathematical proofs and logic a](there).


The full code for the solution that reverse the vigenere cisco hashing algorithm is present [there](https://codeberg.org/gogo/vigenere_cisco_decryptor).


## 2.5

By digging, we realize, it is not mandatory the recode ourself the code of the new algorithm. In effect, the command `pd 310 @ sym.Util::decryptType7_char_const__char__unsigned_int_` of radare2 already contains the revesing of the checksum of Vigenere Cisco in Packet Tracer...

The reverse of the hash `08701E1D5D4C53404A5254537C7E707B616472465243` is `123456789876543210555`. The soluce is [at](https://codeberg.org/gogo/vigenere_cisco_decryptor).


## Ressources

- [vigenere cisco hash algorithm decompiled](https://codeberg.org/gogo/viegenere-cisco-proprietary-algorythm-decompiled-from-packet-tracer)
- [viegenre cisco reverser](https://codeberg.org/gogo/vigenere_cisco_decryptor)