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
