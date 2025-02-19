---
layout: post
title:  "Recovering Deleted PS3 Executables: Blur Edition"
date:   2025-02-18 17:43:00 -0500
author: burninrubber0
categories: recovery
---
# Part 1: Initial efforts
## Preface
In September 2024, I was sent an HDD image from a Bizarre Creations PS3 devkit. This has since been [publicly released](https://mega.nz/file/avRkBDaK#nD3NuBVi7I8N8GeU58NSnuEBcU11aytP3oDhlRzGcXU) (or perhaps rereleased; it's unclear). Several builds had already been recovered by this point, but it was requested that I look over it and attempt recovery of anything that previously proved too difficult to retrieve. (This has also been requested of several other people, and on other Blur HDD images, but I've chosen to focus on this specific case here.)

## Investigation
As with other kits I've encountered, the first—and likely only, at least for a while—order of business was executables. Checking the ones that have already been recovered, there are a few interesting points. For one, and rather unusually, the 2009-12-15 build uses ELFs rather than SELFs. There are also a couple hundred intact smaller ELFs in the `dontship` folders attached to the 2010-01-11 and 2010-03-19 builds. I made sure to map the positions of the main executables in the drive before continuing.

Looking through the search results for both SELF and ELF headers and filtering by the size field, it became clear that not much had been left on the table. However, there's one SELF header at offset 0x40624A000 that doesn't match up with anything previously recovered. This fragments quite quickly (0x1C000), so just to make sure there's actual build data instead of going on a wild goose chase, I searched for the build date.

![2009 search results](/images/blur/2009_search_results.png)

936 of the 988 hits for 2009 are from the one map file still remaining on the drive. The remaining 52 are mostly false positives, filesystem-related paths, and results from the already-recovered 2009-12-15 build, but there are a few unknown sections like these:

![2009 result 1](/images/blur/2009_result_1.png)
![2009 result 2](/images/blur/2009_result_2.png)

Despite their build dates and such making it seem like they're from an executable at first glance, it turns out they're actually the level assets, and all other levels contain a similar plaintext date of creation. This is quite unusual in my experience; normally a FILETIME or time_t would be stored instead. At least this could be useful in asset recovery, but that's not the focus here.

Moving on to 2010, the result count of 260 matches is skewed by symbols and level files again, but this time there's something else:

![2010 result 1](/images/blur/2010_result_1.png)
![2010 result 2](/images/blur/2010_result_2.png)
![2010 result 3](/images/blur/2010_result_3.png)

Alright, so there were 2010-03-13 and 2010-03-17 builds on here at one point. Not that different from the already-recovered 2010-03-19 build, but anything potential builds are worth trying for. It looks like only the 2010-03-17 data still exists, and since there's only one instance of it it's pretty much guaranteed that this drive contains either one or zero potentially recoverable executables. Now it's time to find out whether this data matches the SELF header from earlier.

## Beginning defragmention
The first step in figuring out whether or not this is recoverable is to defragment it. Unfortunately, this section has no easily searchable data. More fortunately, there's a build from two days later to compare with. First, though, which of the configurations is it? Searching a string that exists in every build but master should give up the answer:

![2010-03-17 build config](/images/blur/20100317_build_config.png)

Okay, it's a speed executable. Does any unique data from just before our fragment point (0x1C000) in 2010-03-17 exist in 2010-03-19 amax_speed?

![2010-03-17 fragment 1 search](/images/blur/frag1_search.png)

It does indeed, with a -0x64 offset when going from 3-17 to 3-19. There's some searchable data at +0x30. which means if the next fragment exists, the correct search result's address should end in 030.

![2010-03-17 fragment 2 search](/images/blur/frag2_search.png)

There it is at 40A... exactly as hoped. Or is it? See, I didn't notice at first, but there's a second result at 414... and to save some time writing this, suffice to say I chased the wrong fragment (40A...) for a long time when 414... was the actual correct fragment.

At this point I was demoralized and took a break. A very *long* break. And then I came back.

# Part 2: Recovering the right thing
## More defragmentation
In February 2025, I returned to work on the 3-17 recovery effort. I managed to forget about the earlier defragmentation debacle but correctly deduced that the 414... fragment was the correct one this time because this time I'd decided to properly check certain executable-related things, including the SELF/ELF metadata (possible thanks to some beautiful [certified file](https://www.psdevwiki.com/ps3/Certified_File) and [SELF/SPRX](https://www.psdevwiki.com/ps3/SELF_-_SPRX) documentation on psdevwiki). This wasn't strictly necessary for defragmentation but proved valuable later.

Just one problem: the data appeared complete but wasn't long enough. Using the segment extended headers, I did a lot of comparing offsets and sizes but just couldn't figure out where the problem was. 0x8000 bytes were simply missing. From experience, I knew this was extremely unlikely and figured it was probably something like an 0x20000 fragment being somewhere in the data and throwing things off, and there being more missing data than that.

In the end, I had to figure out a way to know the function locations in 3-17 without having an intact executable so I could compare those locations with what was actually in the file. And luckily, there was a way.

## Comparing via .opd
![OPD section comparison](/images/blur/opd.png)

In builds like these, the .opd (official procedure descriptors) section isn't very interesting under normal circumstances. You can get significantly more info from DWARF debugging symbols. However, thanks to previous searches and knowing the size of the file from SELF metadata, I knew the end of the file (and thus most or all symbols) had been destroyed. And needing function locations, this was the best section to turn to.

By comparing this with analyses of both 3-19 and the broken 3-17 executables in Ghidra, I was able to determine the exact location of the fragmentation. It turned out to be 0x20000—the fragment found previously was a mere 0x4000 long. Worse still, the data after it didn't seem to exist in the image anymore, but I'll circle back to that problem later.

Armed with .opd's function locations and Ghidra, it was possible to figure out what was valid and where it needed to be. In the end, there was an 0x28000-wide gap (0x20000 to 0x48000), with valid data extending to 0x2030000. This finally gave me some confidence that it had been recovered correctly since 0x2030000 is an extremely common size to encounter when recovering from the PS3's filesystem. But now there was a new problem, that of the missing data, and I wasn't sure I could fix it.

# Part 3: Regenerating the code
Remember how in part 1, it turned out the assembly in 3-17 and 3-19 was identical but shifted by -0x64? Well, after comparing the data before and after the missing 0x28000, it turns out both parts are offset by that amount, which means the missing middle section is identical between both. Of course, it's not as easy as copy-pasting, but at least it can be overcome.

Or can it? I hadn't done this before, and after some searching around I can confidently say nobody else had either. In other words, the method of fixing this was purely theoretical. Either way, it was worth a shot.

![Ghidra divergence](/images/blur/ghidra_divergence.png)

Above is where the two executables diverge. I ended up taking sample data from (offsets are file-relative) 3-17's 0x7800 and 3-19's 0x779C with a length of 0x18800, and 3-17's 0x48000 and 3-19's 0x47F9C for 0x1714C. I referred to IBM's [PPC instruction set pages](https://www.ibm.com/docs/en/aix/7.3?topic=reference-instruction-set) and [PPC instructions list](https://www.ibm.com/docs/en/aix/7.3?topic=reference-appendix-f-powerpc-instructions) for what came next.

The sample data proved to be a perfect choice for testing. Using it, I could easily see what would need changing to recreate the missing 3-17 data from 3-19's data. This effectively ended up being two things:

1. The load instructions (`lwz`, `lhz`, `lbz`, `lfs`) that use r2 as the source register
2. The branch instructions (`b`, `bl`) that branch to functions after the executables diverge

Technically there could be more, but these are the ones present in the missing data.

I built a small program that breaks the PPC instructions in a given input file down into their constituent parts. The tool then makes the necessary changes to the relevant instructions. The r2-based load instructions proved easy since at least in this block, the change to the value of `D` was a constant 0x3E8.

Branch instructions were harder to handle. I ended up having to build a map of branch target addresses and the change to `LL` required for the address to be correct in 3-17. Thunks aside, this worked quite well, and only one instruction in the sample data was changed (again because of thunking). If the function in question isn't in the map, the tool just uses the value for whatever the one after it is (assuming the one before it is the same).

![Tool output](/images/blur/tool_output.png)

After some trial and error, I got it working and even discovered that the missing 0x8000 from earlier was still on the image and was completely identical to the tool's output, which I was delighted about and took as confirmation that the tool had done its job. And now, only testing is needed.

![Blur asserts](/images/blur/blur_assert.png)
![Blur running](/images/blur/blur_running.png)

I was only able to get it to assert, but people in the Blur community ran it fine. Either way, the executable is clearly working perfectly, because if it wasn't it would've died instantly rather than display anything. I know this because I messed it up a couple times and that's what it did.

In any case, the recovered executable is available here: https://mega.nz/file/dMNhUBZB#ZLkTbtGMeIFYqN4dChwUtQsM82t8gM_GvQcTPwMrgv8

There's more that wasn't discussed here but it's already long enough. I'll likely be putting this newfound knowledge and experience to use on recovering Burnout executables, to the extent it's possible.

If anyone actually read this, I hope you found it interesting, or at least liked the end result.
