---
layout: post
title: Excellent Stack Buffer Overflow tutorial
---

This is the only tutorial anyone would ever need to handle the BOF machine during the OSCP exam:

<a href="https://github.com/justinsteven/dostackbufferoverflowgood">justinsteven
/
dostackbufferoverflowgood</a>

- Easy mode: no ASLR, no DEP, no stack canaries (party like it’s 1999!)
- WoW64 (Windows 32-bit on Windows 64-bit) 
- Covers stack return pointer overwrite in particular
<br>
<h2>TL;DR</h2>

**1. Trigger the bug:**
- Send lots of A’s
- Expect a crash at `0x41414141`

**2. Discover offsets:**
- `pattern_create.rb`
- `!mona findmsp`

**3. Test offsets**

**4. Discover bad characters:**
- Educated trial and error
- `!mona cmp`

**5. Settle on a spot to stick some shellcode**
- `ESP` often points to right after Saved Return Pointer overwrite – good spot

**6. Use control over EIP to divert execution to shellcode location:**
- Overwrite Saved Return Pointer with a pointer to a `JMP ESP`
- `!mona jmp -r esp -cpb "\x00\x0a"`
- Use an `INT3` breakpoint to test for execution

**7. Generate calc-popping shellcode:**
- `msfvenom -p windows/exec -b '\x00\x0A' -f python --var-name shellcode_calc CMD=calc.exe EXITFUNC=thread`

**8. Account for the decoder stub's GetPC destroying your shellcode:**
- Easy mode: `NOP` sled, `"\x90" * 16`
- Pro mode: `"\x83\xec\x10"` (`SUB ESP, 16`)

**9. Run exploit, pop calc**

**10. Replace the payload, pop shell**
