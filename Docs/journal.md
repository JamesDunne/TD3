Accolade Test Drive 3: The Passion - data decoding journal
James Dunne
https://github.com/JamesDunne/TD3

2013-10-27 10:32 PM CST
-----------------------
After several unsuccessful attempts to gleam any raw information from the *.DAT and *.LST files, I gave that up and tried to disassemble the TD3.EXE program itself to give me a clue as to how it decodes the data itself. I tried several disassembler programs after giving up in frustration due to their complete lack of quality. Even IDA Pro 5.0 (free version) let me down. It was nice when it worked but I just could not make heads or tails of what it was trying to tell me. Also, it seemed to not handle the segmented addressing of DOS real-mode programs well at all. I was able to use its disassembler to find INT 21h calls and it provided helpful comments about the AH value commands for that interrupt, such as 3Fh to read a file and 3Eh to close a file.

A quick google revealed some sufficient DOS INT 21h documentation: http://spike.scu.edu.au/~barry/interrupts.html#ah3f

Giving up on IDA, I remembered that DOSBox has its own debugger, so off I went in search of a debugger-enabled build. Luckily I happened across this crusty old forum: http://www.vogons.org/viewtopic.php?t=7323 . Its download link for the 0.74 version of the debugger-enabled build of DOSBox 0.74 was still in tact on this date. So I grabbed it, thankful that I didn't have to compile the damn thing myself, and gleefully traded away a small amount of security for the convenience.

I fired up DOSBox with its built-in debugger and slowly acclimated myself to its awkward UI, familiarizing myself with necessary keystrokes. The HELP command showed me the only command I was really interested in: `BPINT`.

This `BPINT` command enables you to set a BreakPoint on an INTerrupt call (an operating system call in DOS's case) so that you can stop the program when it, say, reads from a file! Awesome. Now we're in business. From here it's just a "simple" matter of tracing program execution after reading interesting-looking files, like `SCENE01.DAT`. I may end up having to `MEMDUMPBIN CS:IP` the code to get a better handle on it in a more friendly UI. I also foresee many `MEMDUMPBIN DS:DX`s in my future, dumping the decoded data (somehow, somewhere) after it's been read from the file.

Oh yeah, now I remember why I started writing this journal... to actually document binary data formats and interesting points in the code. Duh. Let's get to it.

I fired up the game in the debugger and made my way through the intro screens and menu system to get to the default course/race/whatever. Immediately following the copy protection screen and passing it with flying colors (thank you, whoever hacked that), I stopped execution at the first `BPINT` trap on `INT 21 3F` (read file). Got here from a `CALL 01A5:0DAE`.

`0x2137` bytes were read from offset `0x10240` in `SCENE01.DAT`; haven't figured out how that offset was calculated yet, but I'm sure I'll get to that soon enough. Opening this file up in HxD shows me a big pile of what looks like pairs of 8-bit values with some sort of alternating pattern. No idea what this could be yet... keep diggin.

After stepping through a few more instructions and jumps, I found what appears to be a LZ (Lempel-Ziv) decompression loop at `08FE:104D`:

    08FE:104D  8B76FE              mov  si,[bp-02]             ss:[F93E]=0003
    08FE:1050  D1E6                shl  si,1
    08FE:1052  81C697B2            add  si,B297
    08FE:1056  813C0F0F            cmp  word [si],0F0F         ds:[B29B]=6464
    08FE:105A  7604                jbe  00001060 ($+4)         (down)
    08FE:105C  812C0202            sub  word [si],0202         ds:[B29B]=6464
    08FE:1060  FF46FE              inc  word [bp-02]           ss:[F93E]=0003
    08FE:1063  817EFE0001          cmp  word [bp-02],0100      ss:[F93E]=0003
    08FE:1068  7CE3                jl   0000104D ($-1d)        (up)
    08FE:106A  5E                  pop  si

I'm not sure yet if that is an actual LZ decompressor, but the `cmp` to `0100h` at `1063` there is a strong hint in that direction because of the way LZ compression works. You don't bother storing the first 256 (100h) entries in the dictionary because they're redundant. Then again, I could be totally wrong here and this code is likely something entirely different.

Hmm, I strongly suspect all of the above is nonsense. I notice after playing a while that when I crash, a read-file call is made to `SCENE01.DAT` every time. That seems a better indicator of where to start looking for the decode calls.

An LSEEK (`INT 21h AH=42h`) command is issued immediately after `SCENE01.DAT` is opened and the file position is moved to `17D75` each time. Then `E29` bytes are read from that location.

I've now found an interesting point in the code where the function `1F94:0049` is called with stack arguments pointing to the `DS:DX` where the data was loaded from the file, in this case `81DE:4E5A`.

2013-10-28 09:30 AM CST
-----------------------
I won't be able to make any progress today due to work schedule and prior commitments for the rest of the day.

That function at `1F94:0049` I found yesterday I suspect is strongly related to decoding the scene data. Which scene data, and how it decodes, both remain a mystery as yet. I really need a decent C decompiler to make sense of this code. I can step through it in the DOSBox debugger but reading pages of raw bytes and seeing registers get modified is not exactly revealing of the true purpose of the code there. I wish the debugger would put a little extra effort in and highlight bytes in the memory view when they get read or written. Also trapping on memory reads would be wonderful. Having a call stack view in DOSBox debugger wouldn't be so bad but I suspect that's a non-trivial thing to implement. It's very easy to get lost in the sea of CALLs and RETFs.

While watching the function execute I did see some semblance of decompression work: lots of bit shifting, memory indexing, and counting, but let's not fool ourselves here; those operations are the basic building blocks of virtually all useful code lol.

I should return to this tomorrow evening after work.

2013-10-30 04:40 PM CST
-----------------------
Well yesterday was a wash. I had a headache the whole damn day... but enough excuses; let's get back to business!

Every time I start up dosbox debugger I have to remember to set up the `BPINT` traps:

    BPINT 21 3D  ; file open
    BPINT 21 3F  ; file read
    BPINT 21 42  ; file seek

Let's be methodical about this now. I'll fire up TD3 and start recording traps just after I pass the copy protection screen with the ENTER key.

...

Nope, that didn't happen. I find using DOSBox debugger too painful to use for reading and traversing code at a high level. I need a real tool that lets me analyze procedures and rename stack labels, etc. I ended up digging deep into IDA Pro 6.1 to try to figure out how to disassemble all this code.

As it turns out `TD3.EXE` is packed using some obscure EXE packer tool. If you try to naively disassemble it with any disassembler, all you'll get is the start method which copies data from data segments and jumps to it. Not very useful.

I don't know what EXE packer tool was used and there aren't any obvious signatures for it to advertise itself with. What I did to get a useful disassembly is I used DOSBOX's `debug td3.exe` to breakpoint at the very first instruction of the program and then single-stepped until the real program was fully unpacked in memory and far-jumped to. I could tell which was the final jump-to-unpack because the instruction to do so was a `JMP FAR DS:DX` or something to that effect. One doesn't dynamically jump to random locations in memory all that often so it sticks out in the debugger view like a sore thumb.

Anyway, once I found the entry point (CS:IP) in conventional memory, I took advantage of DOSBOX debugger's `MEMDUMPBIN 0:0 100000` to just dump the entire 16M of memory out to disk in a single BIN file. This is probably overkill but I don't want the analysis to miss any potential code. I also wrote down the CS:IP, which in this case was `1AF0:001C`. I think that seg:offs might actually be hard-coded or at least derived statically from the packed EXE data. It seems to always end up as that value.

I then fed this `MEMDUMP.BIN` file to IDA Pro who then told me to pick a start offset of my choosing and press C to start code analysis at that point. Great! Now we're in business... Yes I've said that before, but I'm getting more serious about it lol.

**NOTE**: The `MEMDUMP.BIN` file is actually saved in your VirtualStore directory for your user account on Windows, e.g. `C:\Users\USERNAME\AppData\Local\VirtualStore\Program Files (x86)\DOSBox-0.74\`. I had to use sysinternals Process Explorer to find that path lol. DOSBOX: you should know better than to try to write files to system folders like program files. Windows: you should pick a slightly less VERY STUPID path for your end-user trickery here. Alright, moving along...

Knowing that DOS segments are exactly `0x10` bytes apart (4 bits), it's just a simple matter to convert a seg:offs pair into a linear offset into the binary file: `(segment << 4) + offset` or `(segment * 16) + offset` for you non-bit-twiddlers out there. It's best to keep figures in hexadecimal in order to maintain sanity, so that offset ends up as `0x1AF1C` (`1AF0 << 4 = 1AF00 + 001C = 1AF1C`). Left-shifting (`<<`) by 4 bits in hexadecimal just adds another zero to the right because each hex digit is represented by exactly 4 bits.

Anyway, before we press the big C key and let IDA go nuts, we have to start defining segments for IDA to decode into. This is sort of an oddball manual process and I'm not sure why IDA doesn't do it itself. Essentially what we do is start with our initial CS segment (`1AF0`) and define that, then search for `CALL FAR PTR` instructions and read their immediate data to find out what other segment values are expected and/or commonly used.

IDA's Segments subview is extremely non-intuitive, but once you figure it out it's rather easy to use. Use `shift-F7` to open the Segments subview. A default `seg000` should already exist which represents the entire 16M binary image as a single segment. This is flawed from the start because a DOS segment can only address `0x10000` bytes, an order of magnitude shy of the total memory available of `0x100000` bytes (1MB exactly). This is why segments exist in the first place heh.

Let's create a new segment to represent `1AF0`. Press `Ins` or right-click and pick the Add menu option. Enter the following values:

    Segment name:  seg1AF0
    Start address: 0x1AF00
    End address:   0x100000
    Base:          0x1AF0
    Class:         CODE
    16-bit segment

Essentially, the `Base` value is the CS register value assumed in the ASM code that falls within that segment declaration. The `Start address` value is the linear offset in the binary file of where the code can be found for that segment. This is calculated with our trusty `(segment << 4) + offset` where `offset` in this case is always 0, so in other words we always just add an extra 0 to the right of the segment value in hex to determine its starting linear offset in the binary file. The `End address` will come in handy later when we start to add more segments, but for now we just set it to the end of the whole file.

I've already done much of the grunt work here in ripping through the disassembled code from the start point and finding all `CALL FAR PTR` instructions and creating explicit segments for their target FAR addresses.
