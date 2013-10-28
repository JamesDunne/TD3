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

Offset `0x10240` was read from `SCENE01.DAT`; haven't figured out how that offset was calculated yet, but I'm sure I'll get to that soon enough. Opening this file up in HxD shows me a big pile of what looks like pairs of 8-bit values with some sort of alternating pattern. No idea what this could be yet... keep diggin.
