# Symbols Memory Dump

It uses `KD`, which is a command line version of `WinDbg` (GUI version).
```c
kd is a kernel-mode debugger based on a console user interface.

WinDbg can be used as a user-mode or kernel-mode debugger, but not both at the same time.
It provides a GUI for the user.
```
> [Windows Internals E7](https://github.com/nohuto/windows-books/releases)  

Preview:

https://github.com/user-attachments/assets/78e67add-6a25-4d49-b60e-d743f9ce492d

Modules are kernel files or drivers loaded into memory with code and data, you can get detailed information about a module with:
```ps
!lmi Module
```
> [lmi.md | windows-driver-doc](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/-lmi.md)  

Symbols map module memory addresses to names of functions and variables. `dd l1` shows a single `32-bit` value at a symbol's address, revealing its current memory data. Before the GUI gets displayed, it checks whether kernel debugging is enabled and KD is installed. It currently searches for `kd.exe` in:
```ps
"$env:ProgramFiles\Windows Kits",
"${env:ProgramFiles(x86)}\Windows Kits",
"$env:ProgramFiles\WindowsApps\Microsoft.WinDbg_*_x64__8wekyb3d8bbwe"
```
If one of them isn't present/active:
```ps
bcdedit -debug on # bcdedit -enum

# Gets installed via winget, you can also install it in the MS Store if you want to
iwr get.scoop.sh -OutFile "$env:temp\Scoop.ps1"; powershell -File "$env:temp\Scoop.ps1" -RunAsAdmin -Wait
scoop install winget
winget install Microsoft.WinDbg --accept-package-agreements --accept-source-agreements
```
> [Winget Install | windows-dev-docs](https://github.com/nohuto/windows-dev-docs/blob/docs/hub/package-manager/winget/install.md)

The tool was partly used for [wpr-reg-records](https://github.com/nohuto/wpr-reg-records#kernel-values) (`nt` module).

## GUI Buttons
| Button            | Description                                                                                           |
|-------------------|-------------------------------------------------------------------------------------------------------|
| `New KD Session` | Starts a new Kernel Debugging (KD) session with the `-kl` parameter                                  |
| `Remove Dumps`   | Removes all folders in `$env:localappdata\Noverse\Symbols`                                           |
| `Reload Modules` | Reloads all modules using `.reload /f`, then lists the loaded modules with `lm`                      |
| `Phase Folder`    | Opens `$env:localappdata\Noverse\Symbols`<br>Each `.txt` file is saved in its respective module folder |
| `Dump`           | Runs through all dump phases using the currently selected module                                     |
| `1`           | Specifies the range of length size, default is `1`, it might not work properly with a length of `8 <`.                                 |
| `dd`           | Displays the current [`Display Memory`](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/d--da--db--dc--dd--dd--df--dp--dq--du--dw--dw--dyb--dyd--display-memor.md) command |

## Dump Procedure

The tool starts a local kernel debugging session for each phase, which means that it has to run `.reload /f` (the `.reload` command deletes all symbol information for the specified module and reloads these symbols as needed. In some cases, this command also reloads or unloads the module itself) in each new session, which is time consuming. I may change it in the future. Dumping a module that includes many symbols it can take some time to go trough each phase. E.g., the `nt` module can take up to `50` minutes.
```ps
-kl # Starts a kernel debugging session on the same machine as the debugger.
.reload /f # Forces the debugger to immediately load the symbols. This parameter overrides lazy symbol loading. For more information, see the following Remarks section.
```
> [reload--reload-module-.md | windows-driver-docs](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/-reload--reload-module-.md)  

I'll use `ReservedCpuSets` (`nt` module, symbole name != value name) for the following examples ([discord notes channel](https://discord.com/channels/836870260715028511/1397387718874501120/1397531587985543268)). 
```c
ValueName.Buffer = L"ReservedCpuSets";
memset_0(&KiReservedCpuSets, 0, 0x100uLL);
memmove(&KiReservedCpuSets, Src, 8LL * v0);
```
Firstly it reloads all modules and lists them in the GUI, after the user selection (`Dump`) it gets all symbols of the module using `x /1 module!*` (`module-Symbols.txt`).
```ps
x [Options] Module!Symbol
/1 # Displays only the name of each symbol.
```
> [x--examine-symbols-.md | windows-driver-docs](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/x--examine-symbols-.md)  

Output will now looks like:
```c
fffff804`d13c66e0 nt!KiReservedCpuSets = <no type information>
```
Now `module-Symbols.txt` gets filtered (output `module-Filtered.txt`), removes header/footer lines, removes symbols `('$mod!(?:_xmm|write_char|write_string|write_multi_char|chunkset_core|ReadString|\s\?\?)'`, `<no type information>`, `parentheses` (not needed anymore since it uses `/1`, but I'll leave it for safety) and it places the symbol into the command:
```c
dd symbol l1
```
```c
dd - Double-word values (4 bytes). The default count is 32 DWORDs (128 bytes).

There are two other ways to specify the value (the L Size range specifier):
- L? Size (with a question mark) means the same as L Size, except that L? Size removes the debugger's automatic range limit. Typically, there is a range limit of 256 MB, because larger ranges are typographic errors. If you want to specify a range that is larger than 256 MB, you must use the L? Size syntax.
- L- Size (with a hyphen) specifies a range of length Size that ends at the given address. For example, 80000000 L20 specifies the range from 0x80000000 through 0x8000001F, and 80000000 L-20 specifies the range from 0x7FFFFFE0 through 0x7FFFFFFF. // l1 = display one unit of data at the specified address
```
If you want to display memory differently with a increased range of length Size, use `New KD Session`.
> [address-and-address-range-syntax.md | windows-driver-docs](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/address-and-address-range-syntax.md)  
> [display-memory | windows-driver-docs](https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/d--da--db--dc--dd--dd--df--dp--dq--du--dw--dw--dyb--dyd--display-memor.md)  

![](https://github.com/nohuto/sym-dump/blob/main/images/dismem.png?raw=true)

`module-Filtered.txt` gets pasted into a KD session, output gets dynamically saved in `module-KD.txt`). The KD window gets opened in the background (minimized), you can open it whenever you want to see the output:
```c
lkd> dd nt!KiReservedCpuSets l1
fffff804`d13c66e0  00000000
```
`nt!KiReservedCpuSets` output is already "correct", but not all symbols will have such a output, which is why a second filter phase is needed. It removes the memory address (`fffff804d13c66e0`), `parentheses` (still not needed, same reason as above), the KD text (`lkd> dd module!`), `l1`, errors (`Couldn't resolve error at 'module!X'`), miscellaneous leftovers. At the end it moves the data (`32-bit` DWORD of data read from memory) to the corresponding symbol line with a `<>` in between:
```c
KiReservedCpuSets <> 00000000
```
The final output is `module-Dump.txt`.

Miscellaneous references:
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/symbol-path.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/debugger-reference.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/kd-command-line-options.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/commands.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debuggercmds/lm--list-loaded-modules-.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/symchk-command-line-options.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/deferred-symbol-loading.md  
> https://github.com/nohuto/windows-driver-docs/blob/staging/windows-driver-docs-pr/debugger/symbols.md  
