There is a [TL;DR at the bottom](#tl-dr).

# Think Before You Print

I recently had the chance to play around with a Brother QL-720NW label printer. Our goal was to use this printer on a Linux system to print labels in a fully automatic fashion as part of some particular process. The paper we had lying around was a 12mm continuous tape roll (Brother DK-22214/DK-2214).

Now, when you buy a 12mm continuous tape roll from Brother you will actually get a 27-ish mm roll that is pre-cut at 12mm height so that you essentially have a 12mm and a 15mm label on top of each other on the same roll. (I'm guessing this is because the printer may have difficulties handling a very narrow roll of paper.) For our use case this was very convenient, because we actually needed two labels associated with the same data in different places. Awesome!
Unfortunately, the Linux driver will only allow you to print on the 12mm part of the roll. I suppose this is totally fine, because you bought a 12mm roll after all and that is what you get. However, I (and others, according to many product reviews) found it a bit wasteful to discard over 50% of the paper and I knew that, at least under OS X, the software will allow you to print on the entire space if you insist on it. This means the printer hardware/firmware is capable of doing so. 


## Setting up

Brother provides a Linux driver for this printer and a [Stack Overflow](http://stackoverflow.com/a/19700576) post contains some hints how to get it working with a modern 64-bit system. On my Ubuntu 16.04 system I was able to install the driver properly with

    # apt-get install libc6-i386
    # dpkg -i ql720nwlpr-1.0.2-0.i386.deb
    # dpkg -i ql720nwcupswrapper-1.0.2-0.i386.deb


## Test page

That was easy enough; let's print a test page. Navigate to `localhost:631`, select the printer, and click on "Print test page." You will notice that the printer does not print anything at all but flashes a red LED. According to the manual there are several meanings to this, but the most likely here is a paper size mismatch. Indeed, the default media size is 29mm x 90mm. After setting this to 12mm you should be able to print the test page (and other documents).

However, it turns out that it doesn't matter what the dimensions of your document are, a 12mm label is going to be cut at about 100mm in length. At least we are covered in this regard—the driver includes a tool called `brpapertoolcups` that will allow you to set a custom paper size. So by running, say,

    # brpapertoollpr_ql720nw -P QL-720NW -n 12x30mm -w 12 -h 30

you can add a custom paper size and thus ensure that the printer cuts the label after 30mm (don't forget to restart the CUPS service after running that command). It is of course tempting to add a paper size of 29mm x 30mm and hope that this will allow you to use the entire space of your 12mm (27mm) roll. Well, the printer is simply going to reject your job (LED is red and flashing) in this case.


## Digging deeper

This tool was an important starting point, though. I ran it through `strace` to see what is actually going on when you add custom paper sizes:

    [...]
    open("/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1", O_RDONLY) = 3
    open("/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1.$$$", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 4
    fstat64(3, {st_mode=S_IFREG|0666, st_size=993, ...}) = 0
    read(3, "paper type:\twidth\theight\n29X1/29"..., 4096) = 993
    fstat64(4, {st_mode=S_IFREG|0644, st_size=0, ...}) = 0
    read(3, "", 4096)                       = 0
    close(3)                                = 0
    write(4, "paper type:\twidth\theight\n29X1/29"..., 1027) = 1027
    close(4)                                = 0
    unlink("/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1") = 0
    rename("/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1.$$$", "/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1") = 0
    chmod("/opt/brother/PTouch/ql720nw/inf/paperinfql720nwpt1", 0666) = 0
    [...]

Looks like there are some files of interest in `/opt/brother/PTouch/ql720nw`! Besides two expected files that store the default label sizes and the custom ones you add there is `/opt/brother/PTouch/ql720nw/lpd/filterql720nw`. This shell script implements a filter chain that receives documents from CUPS and prepares them in such a way that the printer can understand. At the end of this chain is a proprietary binary format that includes a rasterized version of the document. Without really knowing what to expect, I extended the filter chain by `[...] | tee /tmp/dump` and inspected its contents. Luckily, the data looked fairly bitmap-y and rather sparse, uncompressed, and unobfuscated. Good news.

My first question was: what is the difference between a, say, 29mm x 30mm label versus a 12mm x 30mm label? Two dumps later I got

    -rw------- 1 root root 26566 Sep  4 21:59 /tmp/dump12mm
    -rw------- 1 root root 26566 Sep  4 22:00 /tmp/dump29mm

Huh, same file size?!

This is *very* good news. My suspicion at this point was that the bitmap has a fixed width of the maximum label size (62mm) and the printer doesn't actually care about where the paper ends—the driver simply makes sure that it only sets those pixels which are within the bounds of the configured paper size. But if this is the case, why does the printer flash the red LED when we print a 29mm label?

Comparing the first 256 bytes of those two dumps, I noticed only one difference at offset 211:

                      |                                                 |
                      v                                                 v
        0000000 0000 0000 0000 0000 0000 0000 0000 0000   0000000 0000 0000 0000 0000 0000 0000 0000 0000
        [...]
        00000c0 0000 0000 0000 0000 401b 691b 0161 691b   00000c0 0000 0000 0000 0000 401b 691b 0161 691b
    --> 00000d0 867a 0c0a 1b00 0001 0000 1b00 4d69 1b40   00000d0 867a 1d0a 1b00 0001 0000 1b00 4d69 1b40
        00000e0 4b69 1b08 4169 1b01 4d69 1b40 6469 0000   00000e0 4b69 1b08 4169 1b01 4d69 1b40 6469 0000
        00000f0 691b 014a 004d 0067 005a 0000 0000 0000   00000f0 691b 014a 004d 0067 005a 0000 0000 0000
        0000100 0000100

Well, 0x0c is 12 and 0x1d is 29. Apparently, the printer checks if the expected roll that the driver computes from the media size (not document size!) matches the actual roll size installed in the printer. If this doesn't match, red flashing LED.

A quick fix to the filter chain later we are able to use the whole space—by telling the driver to raster on 29mm but intercepting its output to signal the printer to expect a 12mm roll instead of 29mm. This piece of middleware logic is a matter of one line of Python:

    ... | python -c 'import sys; list(sys.stdout.write(chr(29) if i == 211 and ord(c) == 12 else c) for i, c in enumerate(sys.stdin.read()))'

Instant tree saving!


## TL;DR

Add custom media size with 29mm width and desired height (30mm in this example).

    # brpapertoollpr_ql720nw -P QL-720NW -n 29x30mm -w 29 -h 30

Grab [force-12mm-paper.diff](files/force-12mm-paper.diff) and patch driver to select 12mm media when using this custom size.

    # patch /opt/brother/PTouch/ql720nw/lpd/filterql720nw < force-12mm-paper.diff
