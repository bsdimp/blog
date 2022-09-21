# Subtle ways that C has evolved since the 7th Edition

I've been working on reconstructing Venix source for my DEC Rainbow
100B, as long term readers of this column are painfully aware.

Recently, I became aware that the TK Chia and friends has managed to
get their gcc 6.x fork that adds support for the 8088 16-bit (ia-16)
and friends. I thought I'd compile the V7 sources with it to see if
this can help me get past some subtle bugs in my Venix emulator so I
don't have to use the MIT compiler that came with Venix, or port it to
a recent version of C. K&R can still be compiled with gcc and clang
(shocker!), but it's painful and isn't always right.

As part of this, I was trying to configure a kernel for the 7th
Edition kernel. I discovered one of those subtle things that I'd never
known.

## Structure of V7 kernel

There are three parts to a V7 kernel. There's the `sys/sys`
direcotry. This directory contains the core of the kernel. All the
system calls, file system code, etc. Next, there's `sys/dev` which
contains the device drivers and bits of the block I/O (bio) system for
the kernel. And finally there's `sys/conf` which builds all the glue
that holds the kernel together (the cdevsw table, the bdevsw table,
information about the root, what drivers to include, etc).

So the first two bits aren't interesting for this story, so I'll delve
a little into the configuration. The kernel is built with l.s, mch0.s,
mch.c (all of which we'd call locore.s or machdep.c today), c.c and
LIB1 and LIB2 (corresponding to the .o files in `sys/sys` and
`sys/dev` respectively. c.c has the cdevsw, bdevsw and other tables in
it. It's generated by a program that lives in `sys/conf` called
`mkconf` which is a C program. You feed it a config file that looks
something like:
```
hp
root hp 0
swap hp 1
swplo 0
nswap 8778
ht
2dh
dhdm
3kl
```
The format of this file is simple enough. There's an optional count,
followed by either a device driver name or a directive. So in the
above file we are configuring the ho, ht, dh, dhdm and kl drivers. And
there's two instances of dh and 3 of kl (in this time frame, there
were fixed addresses for devices, especailly when there were multiple
of them. DEC established the convention, and Unix just used the
convention. That's why there's no way to configure an address.

The `root` directive sets what the root device is for the kernel. The
swap the swap device. swplo and nswap are relics of the
pre-partitioning era that survived into the paritioning era. Sometime
there weren't enough partitions provided, so you had to swap to areas
between the partitions. Sometimes the partions overlapped so it was
only safe to use part of the partition. smplo was the first block you
could swap to, and nswap was the size. It's one of the tricky things
around disk partitioning that I've explored elsewhere...

## You said obscure, so far this is all straight forward

Oh, right obscure. So, to parse the above, there's a loop that starts like:
```
        count = -1;
        n = sscanf(line, "%d%s%s%ld", &count, keyw, dev, &num);
        if (count == -1 && n>0) {
                count = 1;
                n++;
        }
        if (n<2)
                goto badl;
```
with all the variables properly declared (which I've omitted for space).

Now, when I tried to run this against the above config file, it
declared all the lines were bad execpt '2dh' and '3kl'. Now why is
that? I was baffled at first, but then I read through the scanf code
in V7. There's a bug where an empty number matches %d, but doesn't
count it and no number is returned into the pointer. Unlike a modern
scanf, it doesn't stop the scan. This has since been fixed and my
modern FreeBSD system failed to run this relic. So, what's going on
here is that count is set to a known value we'd never encounter in a
real config file (-1) and if it is still -1 and we matched something
the code compensates for that bug by bumping the count and setting
count to a sane value.

The fix is relatively straight forward:
```
	if (fgets(line, 100, stdin) == NULL)
		return(0);
	if (isdigit(*line)) {
		n = sscanf(line, "%d%s%s%ld", &count, keyw, dev, &num);
	} else {
		count = 1;
		n = sscanf(line, "%s%s%ld", keyw, dev, &num);
		n++;
	}
	if (n<2)
		goto badl;
```
Here, the code peeks at the line. If it starts with a number, then it
will scan it in using the format that starts with a number. Otherwise,
it scans w/o the number and bumps the count.

One of the things that I was hoping when messing around with the new
gcc ia-16 compiler was to make as few changes as possible. It can
build `sys/sys` and `sys/dev` with only one change to add an 'extern'
to a declaration that's now considered ambiguous. So the above changes
are consistent with that. I didn't go through it all bringing it up to
modern standards.

So, there it is. The weird obscure bug that I had to debug on my way
to the crazy Venix thing I've been tilting at for a few years, on and
off... People often idealize V7 in many ways, but when you stumble
over things like this you have to really wonder...