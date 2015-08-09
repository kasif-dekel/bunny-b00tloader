# The Bunny Challenge

A [DC9723](http://dc9723.org/) (DEFCON IL) member posted a RE challenge to the facebook group this weekend: 

![alt tag](http://up415.siz.co.il/up1/od4izm3jinvi.png)

Well, I love challenges so i decided to try and solve this one.

I downloaded the zip archive and there were two files: 

1. Bunny.txt - 
> The Backstory:
> ==============
> You are given a floppy image found in a suspect's house
> Can you extract the secret password from it?
> 
> The Rules:
> ==========
> 	1. This challenge should not be modified by any way. No patching!
> 	2. A bruteforce solution will NOT be accepted.
> 	3. Completing the challenge means that you got a "Well done" message. Contact details will also appear.
> 	4. The creator of this challenge shall not be held responsible in any way - direct or indirect,
> 		to any damange done as a result of this challenge.
> 
> 
> Good luck!

2. bunny.img (which is the floppy image)

Running **"file bunny.img"**:

```bunny.img: DOS/MBR boot sector```

For those of you who doesn't know what a MBR is, from wikipedia:

> A master boot record (MBR) is a special type of boot sector at the very beginning of partitioned computer mass storage devices like fixed disks or removable drives intended for use with IBM PC-compatible systems and beyond.
> The MBR holds the information on how the logical partitions, containing file systems, are organized on that medium. Besides that, the MBR also contains executable code to function as a loader for the installed operating system—usually by passing control over to the loader's second stage, or in conjunction with each partition's volume boot record (VBR). This MBR code is usually referred to as a **boot loader**.

# First Look In IDA

[IDA](https://www.hex-rays.com/products/ida/) can try to produce code by loading the ```bunny.img``` file but this code will be messy and hard to understand since there is no specific format that IDA will use to analyze the binary file.

After some googling i found a nice tool named ```bochs```. For those of you which are not familiar with bochs:

> Bochs is a portable IA-32 and x86-64 IBM PC compatible emulator and **debugger** mostly written in C++ and distributed as free software under the GNU Lesser General Public License. It supports emulation of the processor(s) (including protected mode), memory, disks, display, Ethernet, BIOS and common hardware peripherals of PCs.

The IDA Bochs debugger plugin allows researchers to debug code in a emulated environment. This is implemented using the open source x86 emulator Bochs.

You need to install supported Bochs version (v2.3.7 or above) from [http://bochs.sourceforge.net/](http://bochs.sourceforge.net/).

Next, we gonna have to create a bochsrc file in order to properly load the boot loader into IDA. bochsrc files are handled by the ```bochsrc.ldw``` (IDA Loader). The loader parses the bochsrc file looking for the first "**ata**" keyword then it locates its "**path**" attribute.

An example would be (adjust the attributes accordingly):

```

romimage: file=$BXSHARE/BIOS-bochs-latest
vgaromimage: file=$BXSHARE/VGABIOS-lgpl-latest
megs: 16
ata0: enabled=1, ioaddr1=0x1f0, ioaddr2=0x3f0, irq=14
ata0-master: type=disk, path="bunny.img", mode=flat, cylinders=20, heads=16, spt=63
boot: disk

```

So, first look (as you can see, now it is more easy to understand the code):

![alt tag](http://up415.siz.co.il/up3/jjmmfgnjhnmf.png)

Okay so lets run this boot loader inside bochs and see what happens:

![alt tag](http://up415.siz.co.il/up3/o5tnmkg3diyn.png)

# Looking For A Solution

After exploring the code a little bit you can notice this call:

```asm

7C65                 mov     cx, 0Bh
7C68                 mov     di, 7D84h
7C6B                 call    loc_7CF5

```

And by looking at the procedure that gets called, we can see that this is the procedure that gets the input from the user:

![alt tag](http://up415.siz.co.il/up1/zywheznynxwz.png)

From the x86 interrupts manual:

> INT 16h - Wait for Keypress and Read Character

> 	on return:

> 	-  AH = keyboard scan code
> 	- AL = ASCII character or zero if special function key


So we know that this function reads 0x0B (11) characters inside 0x7D84.

After that, the following code gets executed:

```asm

7C6E                 mov     cx, 0Bh
7C71                 mov     si, 7D84h
7C74                 call    sub_7CC9

```

This is what happens inside the procedure 0x7CC9: 

![alt tag](http://up415.siz.co.il/up3/tzbyknmi2wyk.png)

This code basically ```XOR``` our provided input with a dynamically-calculated key (as you can see in the picture above) and stores the output back.

And then there's a call to 0x7C85 which basically does:

```asm
push    7C93h
retn
```

Which is equal to JMP, so the code at 0x7C93 gets executed: 

```asm

7C93                 mov     si, 7D84h
7C96                 mov     cx, 0Bh
7C99                 repe    cmpsb
7C9B                 jnz     short loc_7CB7

```

This code compares our XORed input with the ```correct password``` and then JuMPs accordingly.

So we can reveal the correct password simply by doing the reverse operations (since ```XOR``` is a symmetric operation).

![alt tag](http://up415.siz.co.il/up2/dktbzgymnewd.png)

Good Luck (All your bases belongs to us -*evilface*-).
