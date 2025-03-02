# Background
Project Gotham Racing 2 is a racing game developed by Bizarre Creations in 2003 for the Microsoft Xbox. The game was written in C/C++ and was likely compiled in Microsoft Visual Studio. The Xbox uses a modified Intel Celeron CPU and uses the x86 instruction set. There are various writeups available online detailing how the Xbox runs on a very stripped down, custom version of the kernel from Windows 2000. Due to this, a lot of APIs and system calls work very similarly to those found in Windows, which themselves are very well documented.\
\
PGR 2 is one of my favourite games of all time, and as a chronic tinkerer I've found myself fiddling with its files and code at various points, trying to make my own mods for it an extend functionality in various ways. Partially wanting to test myself as a challenge and partially wanting to monitor its file reads to see if I could find anything of interest (e.g. load order, requested files that the game looks for that weren't included in retail, etc.), I set out to see if I could make it log file reads.\
\
# Patching CreateFile
As mentioned above, most Xbox titles utilise fairly standard NT-like APIs to interact with the console's hardware. One of these standard calls is `NtCreateFile`, which takes various parameters including a file handle and access mode. Usually these file handles are retrieved from another function that takes the file's path as a string - knowing this, we also know that `NtCreateFile` must be working with, or near, the path to each file that the game tries to read. This is useful because there's a strong chance we can override part of `NtCreateFile` or a nearby function to jump to a code cave, do something new, and jump back.\
\
In my case, I wanted to pipe every file request through OutputDebugString - a standard Xbox API function that logs any given string to the debug console - such that when I monitor my Xbox through Xbox Debug Monitor, I can see what it's accessing whenever I load a race or click on a certain menu. Finding out how to do this wasn't easy for a beginner, but in case it's useful (and possibly applicable to other Xbox games), I'm documenting it here.\
\
In default.xbe in IDA, at offset `0x18F081` (`0x17F081` in the XBE file), the original code has just finished moving the address for what I'll refer to as filePath into the EDI register, and is about to push it and some other information from EAX so that it can call RtlInitAnsiString. I figured this was a good place to hijack the original code, so I did the following:
## 1: Identify the bytes to replace with a jump
A jump instruction is usually 5 bytes - 1 for the instruction itself, and 4 for the address offset to jump to - so in this image, I'm picking the `push edi` as my starting point:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/42bc7e8f-3744-4607-acb8-e010b0c33683)\
\
I replace the `57 8D 45 F4 50` with my own jump. I identified a space at `0x28BAA4` (XBE offset `0x278824`) that looks unused and large enough to add some new code, so I used IDA's Assemble function (Edit->Patch Program->Assemble), typing `jmp 28BAA4`, and it handily calculates the offset for me:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/638cde4e-4a43-44ba-9f81-46ae1218443a)

## 2: Adding our new code
After consulting some x86 register documentation, discussing with others, and playing around in IDA, I gradually became more familiar with the concept of the stack, and how calls to functions will pass in N elements popped from the stack, where N is however many parameters that function takes.\
\
From past experience I already know that the Xbox API's OutputDebugString takes one parameter, a pointer to a string. Also through digging I've established that OutputDebugString resides at `0x192298`. As mentioned above, we're at a point where EDI contains the pointer to filePath, so we want to push EDI onto the stack then call the function, which will both use and pop EDI from the stack.\
\
Again I used IDA's assembler for this, simply inputting `push edi` to get another copy of it on the stack, and `call 192298h`, which resulted in the following with the nicely calculated offset:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/975d72be-4ae2-4df0-9e70-f318b76e51b2)\
\
At this point, the changes are actually almost done! I've diverted CreateFile to a new chunk of code and passed what it had already set up into a debug output function. Right now I can probably boot the game and see it log the first file it encounters - then fall down and crash the console. This is because I'm missing two things: restoring/replicating those 5 bytes I destroyed, and jumping back to where I left off. The former leads us to...

## 3: Cleaning up
I am eventually going to jump back to the code I started off from, but first I need to clean up the mess I've made. Those bytes I overwrote to make my jump were _probably_ setting up important variables on the stack for the following function call to RtlInitAnsiString, and right now I'm missing them entirely. Luckily, I've already jumped to a fairly empty point of the executable, and I've still got the current stack and register setup with me, so all I have to do is put them back in, after my call to OutputDebugString, exactly as they were.\
\
In this case IDA was a little finicky trying to assemble the exact typing of the `mov` I wanted to recreate, so knowing I only had 5 bytes to replicate I simply went to Edit->Patch Program->Change Byte, and replaced the first 5 filled-in bytes with the original `57 8D 56 F4 50`:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/8dc9f073-66fb-480b-8b99-fc2b12788d41)\
(You can usually tell when you've done this right as IDA tends to fill in whichever labels it or yourself have given the variables)

## 4: Jumping back
There's only one thing left for me to do at this point: go back to where I came from. Because I put my jump at `0x18F081`, and my instruction took up 5 bytes, I simply need to jump back to the next instruction, `0x18F086`. IDA will happily assemble this with calculated offsets for me, so I simply put `jmp 18F086h`, and voila - my new code is complete.\
\
Because I made sure to do things carefully, such as pushing EDI to the stack again (as OutputDebugString will pop it and the original code still needs it), by the time RtlIniAnsiString is executed it has no idea that anything has changed, and everything following will execute as expected. The only difference is the one that I can observe myself - the Xbox is now printing every accessed file path to my xbWatson instance:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/e38bf84b-efff-475f-8db1-6f366a2c4ac7)

## 5: Extra bits
As you might've noticed above, the file paths are simply being spat out one after another, with no line breaks. While it works perfectly fine, it's not great to read, so let's fix that. My method of doing this was quite simple - call OutputDebugString immediately after the one that outputs the file path, but this time simply outputting a pointer `0A`, a linebreak character. I couldn't identify any strings that contained nothing but the text representation, `\n`, so I ended up taking a C++ runtime string that I believe the game never uses itself, at `0x28C240`:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/88bbe968-9b1b-4c0a-ad97-a3256a2c9837)\
\
Replacing the 'T' at the start with `00` in hex terminates the string after the `0A` character, so the Xbox won't bother reading the rest of the string:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/30db7b41-5b15-4a9c-8906-5340793888ee)\
\
From there, I went to the line after my original OutputDebugString call, added `push 28C240h` to get that memory address on the stack, and another call to OutputDebugString assembled by IDA. It's probably not the cleanest solution, but it's simple enough for me and I have the space to make the extra push and call.\
I then re-implemented everything that was below my original call as I've now overwritten them, again manually writing most in Change Byte, but using Assemble for the returning jump so that IDA would re-calculate the offset from the shifted line. The final chunk of new code looks like this:\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/cf77d617-4443-4410-82db-e3d3f98e30e9)\
And my xbWatson output is now nice and readable, like this:\
\
![image](https://github.com/AJB-Tech/PGR2Doc/assets/12451453/1cb6aaf8-7c26-4587-ab56-f193b087d464)







