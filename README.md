
Debugging FreeBSD with VS Code and GDB
====
FreeBSD is a free and open-source operating system which the PS4 Orbis OS is based on (9.0-RELEASE). By debugging the OS directly and reading the source code, you can learn a lot about OS internals. Applying that knowledge to the PS4 can be fun if you're interested in homebrew or understanding how a potential "jailbreak" works. There are several articles detailing how some researchers used their FreeBSD knowledge to develop exploits on the console such as: [(1)](https://cturt.github.io/ps4.html) [(2)](https://fail0verflow.com/blog/2017/ps4-namedobj-exploit/) [(3)](https://github.com/Cryptogenic/Exploit-Writeups/blob/master/PS4/%22NamedObj%22%204.05%20Kernel%20Exploit%20Writeup.md).

Getting started debuging an entire OS can seem like a lot of work but there are a lot of great tools that can help make it easier to get debugging.

# Setup

[VS Code](https://code.visualstudio.com/) is a cross-platform free and [open-source](https://github.com/Microsoft/vscode) code editor that has suport for many types of [debugger extensions](https://marketplace.visualstudio.com/search?target=vscode&category=Debuggers&sortBy=Downloads).

## Extensions

You can use the [official C\C++ extensions](https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools) or webfreak's [Native Debug](https://marketplace.visualstudio.com/items?itemName=webfreak.debug) extension to setup GDB debugging.

## Instructions

Thanks to [fail0verflow](https://fail0verflow.com/blog/2012/cve-2012-0217-intel-sysret-freebsd/#kernel-debugging) and [Austin Hanson](http://austinhanson.com/vscode-gdb-and-debugging-an-os) we can break our requirements down to:


1. GDB targeting FreeBSD (`amd64`)
2. FreeBSD sources and kernel symbols
3. FreeBSD VM
4. VS Code

This repo contains a pre-built GDB binary for 64-bit Windows, but if you've built a gdb for another OS, it should still work. The version/release is [7.10.1](https://ftp.gnu.org/gnu/gdb/gdb-7.10.1.tar.gz) and it for use with a 64-bit FreeBSD.


1. Install [VS Code](https://code.visualstudio.com/).
2. Install the debugger extension [here](https://marketplace.visualstudio.com/items?itemName=webfreak.debug) or press `Ctrl+Shift+X` inside VS Code and search for `Native Debug`.
3. Enable debugging for your VM. VMware defaults to port `8864`. We enable it by opening the virtual machine's `.vmx` file and adding the line: 

```
debugStub.listen.guest64 = "TRUE"
```
4. Create a new folder that will hold your source files(`debug-root`), kernel, and symbols. Copy `/boot/kernel/` and `/usr/src/` into this folder. You can copy these directories from your VM machine using ssh or you can get the `/boot/kernel/` directory from the FreeBSD ISO image by mounting it or extracting it. You can also get the source files for a specific FreeBSD release using git:
```
git clone -b releng/9.0 --single-branch https://github.com/freebsd/freebsd.git ./usr/src
```
* Running the command in the `debug-root` will create the `/usr/` folder by checking out the 9.0 git branch files. You can also specifiy different releases by changing `releng/9.0` to a different [release](https://www.freebsd.org/releases/), such as `releng/11.0`.

5. Now, right click inside the `debug-root` folder and select `Open with Code` to launch VS Code. Select the `Debug` menu and then `Add Configurations...`. You will see a list of debugger types and you should select `GDB`. This will generate a `.vscode` folder with a `launch.json` file. We will configure this file to attach to our VM.

The current layout of your folder should now look something like this:

```
[+].vscode/
    --launch.json
[+]kernel/
    --acc.ko
    --aac.ko.symbols
    --accf_data.ko
    ...
    --kernel
    --kernel.symbols
    ...
[+]usr/
    [++]src/
        --bin
        --cdll
        ...
```

Create a `bin` folder inside the main directory `debug-root` and place your `gdb` there (if you're using the repo binaries, place the `bin/gdb-7.10.1/gdb.exe` file inside your `bin` folder), then and set (or copy a file from the `examples` directory) your `launch.json` to one of the following:

For the official C/C++ extensions:
```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Attach to FreeBSD",
            "type": "cppdbg",
            "request": "launch",
            "program": "${workspaceRoot}/kernel/kernel",
            "stopAtEntry": false,
            "cwd": "${workspaceRoot}",
            "miDebuggerPath": "${workspaceRoot}/bin/gdb.exe",
            "MIMode": "gdb",
            "miDebuggerServerAddress": "localhost:8864"
        }
    ]
}
```
For Native Debug:
```json
{
    "version": "0.2.0",
    "configurations": [
    {
        "name": "Attach to FreeBSD",
        "type": "gdb",
        "request": "attach",
        "executable": "kernel/kernel",
        "target": "localhost:8864",
        "remote": true,
        "gdbpath": "bin/gdb",
        "cwd": "${workspaceRoot}"
    }
    ]
}
```

`debug-root` should look like this:
```
[+].vscode/
    --launch.json
[+]bin/
    --gdb.exe
...    
```

Save the configuration and you should be able to start a debugging session by running your VM and then inside VS Code, press `Ctrl+Shift+D` and selecting the `Attach to FreeBSD` configuration in the top left.  You can press `F5` or press the green arrow next to configuration list to attach to the VM. 

## Usage
When entering commands using the official extensions, you need to prefix the command with `-exec` but with Native Debug(ND) you do not. You can set source line breakpoints with VS Code and you can also display the assembly code directly inside the Debug Console by entering the `display /{n}i $pc`. With this command you can display *`n`* instructions at each step and each breakpoint. For example, to display the next four instructions, including your current address, you can enter `display /4i $pc` for ND or `-exec display /4i $pc` for the official extensions in the Debug Console.


## Notes
If you disconnect the debugger and connect again, you might receive this message when using Native Debug if it fails to break/stop:

```
Not implemented stop reason (assuming exception): undefined
```

To resolve this, use VS Code to set a breakpoint inside the file `/usr/src/sys/amd64/acpica/acpi_machdep.c` at the function `acpi_cpu_c1` at the line 

```
__asm __volatile("sti; hlt");
```
then press `Ctrl+Shift+F5` or the `Restart` button. It should now break and let you debug as usual.

## Official Extensions Preview
![vscodegdbofficial]

## Native Debug Preview
![vscodegdbnd]


# GDB TUI 
If you prefer the native GDB interface, you can also use the TUI version in repo's `bin/gdb-7.10.1-tui/` directory directly. To do so, use the following commands to start and attach to the VM:

```
gdb-tui.exe -q -tui kernel/kernel
(gdb) target remote localhost:8864
```

You can switch TUI views by pressing `Ctrl+X+2`, for example to see the assembly instructions. You can switch back to source view with `Ctrl+X+1`.

## Preview
![gdbtui]

[vscodegdbnd]: images/vscode-gdb-1.png "GDB Debugging with Native Debug"
[vscodegdbofficial]: images/vscode-gdb-2.png "GDB Debugging with Official Tools"

[gdbtui]: images/gdb-tui-fbsd.png "GDB TUI"
