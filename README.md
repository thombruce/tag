# Thom's Guide to Dual-booting Arch Linux on a Windows 11 Machine

## Step 0: Are you Thom?

0. Is your name Thom? If not, this guide probably isn't for you!

No, seriously, I'm writing this guide before I've even begun the process - sort of a _measure twice, cut once_ sort of thing.
The steps here are going to be super specific to my current machine.
This guide will be outdated and redundant for even myself in the future.
Might update later, who knows!

## Step 1: READ THE FUCKING WIKI!

[Dual boot with Windows - ArchWiki](https://wiki.archlinux.org/title/Dual_boot_with_Windows)

Everything else I do here will branch off from this guide.
We might visit other pages on the Arch wiki for specific setup or problem solving,
and we're definitely going to visit other sites for extremely niche information too.
But this wiki page is our Bible; from here on out, we are zealots and _"Dual Boot with Windows"_ is our religion!

## Step 2: What the fuck is an EFI system partition?

### 2.1: Are we EFI yet?

First, per the docs, we need to figure out whether our system boots in UEFI mode or BIOS mode.

Long story short, mine is UEFI. Older systems are likely booted in BIOS mode.

This was straightforward to discover. We can simply open the Run dialog in Windows (<kbd>Win+R</kbd>) and run `msinfo32`.

In the "System Summary" tab, we can find an entry labelled "BIOS Mode". Mine reads: "UEFI". So that's...

1. <kbd>Win+R</kbd>
2. `msinfo32`
3. "System Summary"
4. "BIOS Mode" (this will read either "UEFI" for UEFI systems or "Legacy" for BIOS systems)

**The rest of this guide will concern itself only with the UEFI boot mode.**

### 2.2 Fucking around (Carefully! Carefully fucking around!) with partitions

Okay, so we know we have an UEFI boot. The guide now tells us that Windows creates only a 100 MB system partition for this...
and ideally we need that to be bigger.

To verify the size, we need to look at our disk partitions in the Disk Management utility.

1. <kbd>Win+R</kbd>
2. `diskmgmt.msc`

What we ought to expect to see here are THREE partitions. One "EFI System Partition", one large partition labelled as our "C:" drive, and one smaller "Recovery Partition".

I see FOUR partitions. Fuck! And the fourth is unlabelled. It occupies 16.60 GB of space to the left of the disk... and its purpose is a mystery!

Let's get more detail. We're going to open up a dialog with DiskPart and examine the partitions more closely. Do...

1. <kbd>Win+R</kbd>
2. `diskpart`
3. `list disk`
4. `select disk 0` (replace `0` with the number of the disk you want to examine)
5. `list partition`
6. `select partition 1` (replace `1` with the number of the partition you want to examine)
7. `detail partition`

When we list our partitions at step 5 above, we'll notice we have at least one additional partition that Disk Management did not show us.
This is a 16 MB "Reserved" partition, and we're not gonna worry about that just now...

Completing the steps above reveals that my mystery 16 GB partition is a "Recovery" partition with an NTFS filesystem.
_This is the same filesystem used by our C: drive. In theory, if we could assign this mystery drive a letter we could easily go exploring its contents..._

This honestly doesn't tell me much, and at this point I discover via searching online that the partition is a factory reset partition created by my machine's manufacturer.

I am fucking around with a Razer Blade Stealth 13" laptop, and DuckDuckGoogling this along with "16 GB partition" revealed another confused owner asking the same question.
This is how I discovered that it is a factory reset partition.

But look, it's about the journey, right? We may have gleaned little of use just yet from DiskPart, but we're glad we used it - that utility is gonna be handy later!

The good news is we can delete that partition by right-clicking on it in our Disk Management panel (we don't need a factory reset; we're installing an OS here, we can absolutely restore a near-factory state manually should we ever want to).

The bad news...

| | | **Blade Stealth (C:)** | |
| 16.60 GB | 100 MB | 459.19 GB | 1.04 GB |
| | Healthy (EFI System Partition) | Healthy (Boot, Page File, Crash Dump, Basic Data Partition) | Healthy (Recovery Partition) |

This table represents what I'm seeing in Disk Management. The unlabelled partition occupying 16.60 GB of disk space we're going to be happy to delete, but...

Other partitions can only be grown into adjacent space. Moving the partitions is a lot more complicated (your computer's data is physically stored somewhere and every byte might need to be moved).

What we have here is 16.60 GB of reclaimable space on the **WRONG SIDE** of the EFI System Partition.

I mean, if we weren't planning to dual-boot, it would be nice to reclaim that space for C.
Since we are dual-booting, we actually want to claim that space for our new Arch Linux partition, but we also want to claim some additional space by shrinking C:. _Incidentally, this runs into another problem that we will get to..._

For now though, precisely what we want to do is delete the unlabelled factory reset partition and then increase the size of the EFI system partition.

**And also...** _MOVE_ the EFI system partition all the way to the left, placing the 16.60 GB of newly unallocated space next to C: (actually next to that 16 MB reserved partition I told you not to worry about - here we start to worry about it).

So let's go over what we would like to do at this point:

1. Delete the unlabelled volume occupying 16.60 GB of space
2. Extend the EFI System Partition volume to around 1 GB
3. **MOVE** the EFI System Partition all the way to the left (**DANGEROUS!**)
4. **MOVE** the hidden, reserved 16 MB partition leftwards too (**ALSO DANGEROUS!**)
5. _We then, in theory, have 475 GB of adjacent space (unallocated and C:) to play with_

> _And when I say **DANGEROUS!**, I mean it.
> Some DuckDuckGoogling informs me that attempting to move my EFI partition could result in my computer becoming unbootable.
> Isn't that a terrifying prospect?
> Definitely worth pausing for thought at this point...

