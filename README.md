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
2. ~~Extend the EFI System Partition volume to around 1 GB~~ (_Bad info! See "Step 4"._)
3. **MOVE** the EFI System Partition all the way to the left (**DANGEROUS!**)
4. **MOVE** the hidden, reserved 16 MB partition leftwards too (**ALSO DANGEROUS!**)
5. _We then, in theory, have 475 GB of adjacent space (unallocated and C:) to play with_

> _And when I say **DANGEROUS!**, I mean it.
> Some DuckDuckGoogling informs me that attempting to move my EFI partition could result in my computer becoming unbootable.
> Isn't that a terrifying prospect?
> Definitely worth pausing for thought at this point...

## Step 3: I can ride my bike with no handlebars! (We're deleting the factory reset partition)

Okay, so we know we can free up 16.60 GB of space by simply deleting that unlabelled partition.
We know that it is intended for performing a factory reset... but we don't need that.
We're clever little things; we don't need stabilisers on our bicycle.

So, in Disk Management, let's right click and select "Delete volume...".
This brings up a window telling us that the volume was not created by Windows and
may contain data recognised by other operating systems.
But we get a "Do you want to delete this partition?", "Yes" or "No" dialog.

_I expect hitting "Yes" here will delete the partition and free up the space without issue._
_But right now we're still "measuring twice"; we will take this step when we commit to the whole process._

## Step 4: One does not simply walk the EFI partition to Mordor...

Look, the _Lord of the Rings_ analogy is gonna fall apart instantly but it looks like we can't just "move" the EFI partition into the now free space.
We have got to throw it into the fires of Mount Doom (delete it) instead. **NOT BEFORE** creating a new EFI partition, obviously.

So... we'll create a new partition all the way to the left in the newly freed space.
That part's easy. We'll make it 1 GB. And our computer should have no issue with these two EFI partitions both existing.
The issue comes when we delete the original partition, which is... _oh boy,_ it's responsible for booting our computer.

Yeah, this part is scary. But in theory, once we have the new EFI partition setup appropriately, and we have copied the data
from the old partition to the new one...
Well then our system should recognise the new EFI boot partition as though nothing's happened.
Assume that it's formatted correctly, assume that all the data has been copied, and it should be functionally identical to the original partition (just in a different place and an order of magnitude bigger).

We're going to be so careful about the steps we take here though.
I'm advised that we may need to boot Windows from a USB? We'll see.
Because here's the thing: We **CANNOT** delete a critical system partition while our system is running and dependent on it.
I don't know what happens if we do? System crash? Does it become unbootable? Terrifying stuff.
So let's get this properly figured the fuck out before we do anything.

### Step 4.1: The layout we desire

I guess it makes sense to describe what I want...

Ideally, our newly unoccupied 16.60 GB would be claimable by my new Arch filesystem.
But that's not all I want. I want to steal additional space from the C: drive (right now, I can shrink it by 40 GB even though there are 202 GB free - we'll come back to this).
Let's say that that's all I want... I want to increase the size of my EFI partition to 1 GB and alott the other 15 GB plus, let's say, 40 GB from C: to my Arch drive...
This should look like this:

|  | **_Arch_** | **Blade Stealth (C:)** |  |
| 1 GB | 55.6 GB | 419.19 GB | 1.04 GB |
| EFI System Partition | _Arch filesystem_ | Boot, Page File, Crash Dump, Basic Data Partition | Recovery Partition |

The two (and a half) problems with achieving that right now are...

- The EFI system partition is badly positioned (and must be recreated and deleted)
- We don't know how much the C: partition will shrink by on either side (that's how this works, right?); we might not get the full 40 GB on the left side
- There exists that 16 MB reserved partition just before the C: partition and not shown (this too is in the way of claiming space, I would assume)

There are two groups of steps towards achieving this result...

#### Step 4.1.1: The easy part

1. Delete the unlabelled factory reset partition, creating 16.60 GB of unallocated space
2. Shrink the C: partition, creating 40 GB of unallocated space (falling either side of C:, we presume)

#### Step 4.1.2: The hard part

1. Create a new 1 GB EFI System Partition and copy data from original
2. Delete old 100 MB EFI System Partition
3. Unify the unallocated spaces (now roughly 15.60 GB and 40 GB = 55.60 GB)

_Why is this hard? Because step 2 above is dangerous, and step 3 has potential blockers in how these partitions had been arranged._

---

Since Step 4.1.1 is "easy" it is worth doing straight away, I think.
Following this, we will have a visual indication of... where we're at and what we have to work with.

Certainly there's no harm in deleting the factory reset partition. I'll do that... now?

**_Now?_** _Fuck!!_ Wish me luck!

...

Okay, we're good. We now have 16.60 GB of unallocated space over there.

I'm going to hold off on the also "easy" action of shrinking C: by 40 GB.
Here's my thinking: Maybe we can work with this 16.60 GB of space initially.
We partition off 1 GB of space for a new EFI System Partition which will...
It'll contain GRUB and be used to boot Linux (eventually also Windows).
And we just install Arch in the other 15.60 GB of available space.
We can increase it later...

That might be an approach. Ah, but no...

## A thought occurs!

We interrupt these numbered titles because I've stepped away from my computer for a bit and a couple of thoughts came to mind:

1. The reason I can only get 40 GB from C: is because of some immovable files, but I came across some information earlier that might help me reclaim more...
2. Even though there is adjacent space, the option to extend the EFI System Drive into it isn't available. Since the ArchWiki recommends resizing it, perhaps the docs go into doing that in such a way where we could successfully move it too?

To point one, we can do a few things to address these "immovable files". We can...

1. Disable hibernation (I never use it anyway)
2. Disable Pagefile (it's virtual memory that extends the physical RAM - not strictly necessary, but Windows might depend on it in some circumstances)
3. Disable system protection/system restore

_All info per this article here: https://www.thewindowsclub.com/shrink-volume-with-unmovable-files-in-windows_

Note that there is some info at the beginning of the article too about uncovering the unmovable files blocking us in Windows Event Viewer.
That will be worth a look.

1. <kbd>Win+R</kbd>
2. `eventvwr`
3. Windows Logs
4. Application
5. Actions -> "Filter Current Log"
6. In "Filter Current Log" window: "Event Sources" -> Defrag
7. Explore the logs looking for "The last immovable file appears to be: [...]"

In my case, this is `\$Mft::$DATA` ... which does not seem to be consistent with any of the above disabling suggestions but... well, we step through the problem bit by bit, byte by byte!

**IMPORTANT:** Re-enable system protection and Pagefile after shrinking the drive. Reenable hibernation too, if you want. I might, just 'cos.

## Step 5: Turning down the volume...

So, let's just try it. We're just gonna disable hibernation, disable Pagefile and disable system protection,
and then we're going to restart my computer and see if we can grab any more than 40 GB.

If not, we'll just reenable those features and take another look at this later.
_Because we have that 16.60 GB free now to play with anyway, right?_

### Step 5.1: Disabling hibernation

Per documentation found here: https://learn.microsoft.com/en-us/troubleshoot/windows-client/setup-upgrade-and-drivers/disable-and-re-enable-hibernation

We're going to open Command Prompt with administrator privileges and we're simply going to run:

```
powercfg.exe /hibernate off
```

To re-enable this, we will repeat the same steps and run the command with an `on` argument instead:

```
powercfg.exe /hibernate on
```

### Step 5.2: Disabling Pagefile

1. Open system properties
2. "Advanced" tab
3. Open "Settings..." under the "Performance" heading
4. "Advanced" tab
5. Select "Change..." under the "Virtual memory" heading
6. We're gonna deselect "Automatically manage [...]"
7. We're gonna select "No paging file" and hit "Set"
8. Restart computer

To re-enable it, we should just be able to re-select "Automatically manage [...]"

### Step 5.3: Disabling system protection

1. Open system properties
2. "System Protection" tab
3. Select the C: drive and hit "Configure..."
4. Select "Disable system protection" and hit apply

To re-enable, we should just have to re-select "Turn on system protection" at that final step.
Should we need to re-apply the setting under "Disk Space Usage", it currently shows as at 2% (9.18 GB) for "Max Usage".

### Step 5.4: So, it didn't work...

We still only have the capability to shrink the C: partition by 40 GB.
`\$Mft::$DATA` continues to stand in our way... I presume. I'll check!

I does say this "appears to be" the blocking file though... and I did see advice online earlier saying...
basically, shrink the volume anyway. You might be able to continue shrinking after.
So I'm gonna do that. The volume is now being shrunk.

This has created 39.77 GB of unallocated space to "the right" of the C: drive. And...

Unfortunately, we can shrink it no further. Let's re-enable those features then, and then we'll...

Then we'll deal with the spaces we're looking at.

### Step 5.5: Defrag, maybe?

I've run defrag on the C volume to see if this will consolidate and free up the space being blocked by that weird entry.

As of now, I'm still getting 0 MB as available shrink space... Okay...

I am going to look at the shrinking option again after one more restart, but I anticipate it will still show 0.
In which case... the next recommendation is for third-party partitioning tools, per this answer on Microsoft Learn: https://learn.microsoft.com/en-us/answers/questions/3909152/cannot-shrink-volume-due-to-unmovable-file-mft-dat

_Why third-party? Well, according to "Dave" Windows just had limited partitioning capabilities. This third-party tool... should apparently do a little more?_ 🤔

So at this point I have downloaded AOMEI Partition Assistant Standard Addition. It's freeware that wants to upsell you the pro version whenever you open it, but it's already yielding some good information...

It won't let me resize or move (_wait, move?_ yeah, I know!!) any of my partitions unless I disable BitLocker Device Encryption.

Totally happy to do that temporarily, but right now I need to take a break... Then again...

Disabling BitLocker encryption might take an amount of time. 🤔 I could certainly start it while taking a break.
I just need to turn it back on tonight or early tomorrow. Fuck it, yeah. Let's disable BitLocker and see if we can't
do a bit more with our partitions after that.

### Step 5.6: Disabling BitLocker

Takes a minute, but it's a straightforward process.

### Step 5.7: Third-party partition management to the rescue!

With BitLocker encryption disabled, AOMEI Partition Assistant was able to resize C: down to 300 GB (an arbitrary number, chosen because it was the closest round number - there will be more resizing/moving in future).

Again, this is a straightforward process not much worth documenting...

Just be careful and understand that AOMEI can have several jobs queued at once before applying the changes.
In this case, I only wanted to shrink C so... read the information provided as it comes up, sit back, and AOMEI will achieve the result for you.

...

After probably not enough thought... I decided to go ahead and move all of my partitions using that AOMEI tool.
I trusted that if I was about to do anything particularly scary, it would warn me... and err, well, it didn't.

I now have a 1 GB EFI System Partition to the far left hand side of the disk image.
This is followed by the 16 MB Microsoft Reserved Partition.
This is followed by a 300.0 GB partition for my C: drive and... hey, the fact that I'm writing this means this is all still functional. Hell yeah! 🔥
Then we have 174.88 GB of unallocated space. This will be the space for my new Arch Linux filesystem.
And finally, there's the 1.04 GB recovery partition; this is at the end of the disk and is the only partition not moved or resized at all during this process.

Okay, big recommendation for AOMEI here. That all kind of just worked like magic.

## Step 6: Windows Settings

I've come across two things that need to be altered while rambling through the wiki pages...

1. We need to disable UEFI Secure Boot; this is an essential step, I believe. I don't know if we can turn it back on later either as we're going to be installing a new boot manager... Must investigate.
2. We should set Windows' system time to UTC

To point two above, there is more information here: https://wiki.archlinux.org/title/System_time#UTC_in_Microsoft_Windows

Unlike Unix systems, Windows uses localtime by default instead of UTC.
Fixing this is straightforward though. One command that we can run in a command prompt with administrator privileges:

```
reg add "HKEY_LOCAL_MACHINE\System\CurrentControlSet\Control\TimeZoneInformation" /v RealTimeIsUniversal /d 1 /t REG_DWORD /f
```

We're fucking with registry values here, which I don't know a lot about so a lot of that is foreign to me, but I can see the gist of it.
May as well run that now.

**Done!!** Nothing visible changes. All it says is "The operation completed successfully." I'm going to trust that it did.

I won't disable secure boot yet. I want to be ready to move forwards with the Linux installation before I do that.

Also... can I reenable BitLocker encryption at this point?

... I think not. And I think ultimately we will be encrypting via a different method.

I have at this point, per the wiki, disabled hibernation again:

```
powercfg /H off
```

The one Windows setting I still need to amend is... I need to disable UEFI Secure Boot.
The wiki describes this as... rather complicated.

### Step 6.1: Rather complicated (disabling UEFI Secure Boot)

Info: https://wiki.archlinux.org/title/Dual_boot_with_Windows#UEFI_Secure_Boot

We actually need to do this from the BIOS menu, which means restarting my computer and entering the BIOS rather than allowing Windows to boot.

This means no access to this Neovim, or my browser... I'm gonna have to bring the information up on my phone.

We're going to follow the most authoritative source for this, Microsoft themselves: https://learn.microsoft.com/en-us/windows-hardware/manufacture/desktop/disabling-secure-boot?view=windows-11&preserve-view=true#disable-secure-boot

However... we only need to do that when we're ready to install Linux. As of now, we aren't. I haven't prepared a disk image.

Just know that that is **IMPORTANT!** and needs to be done prior to Arch Linux installation.

In fact, it is the last thing we need to do before Arch Linux installation. So...

We should prepare for that step next.

## Step 7: Do you even Arch Linux, bro? (Installing Arch)

> **IMPORTANT!** Per the above section, do not forget to disable Secure Boot in the system BIOS prior to installing Arch Linux.

We're ready to begin installation, but there appear to be a few ways about this.

I'm also still unsure how Arch will go about knowing which disk partitions to use.
That is, how will I inform the installation to use the pre-existing EFI System Partition for boot installation
and the 174.88 GB of now "unallocated" space for my Arch Linux filesystem.

Do I simply... tell it to use these spaces? If so, how? Is it by ID? How do I even obtain the ID of the EFI partition? 😕

Documentation online talks about mounting to `/boot` and `/`. These are familiar patterns to me.
They look like filesystem directories. But um... that's obviously quite alien compared with what I've been doing these past few hours.

_Another thing that's concerning me is the presence of "Boot" in my "Blade Stealth (C:)" partition._
_But I'm going to relax about that, as screenshots of others' Disk Management results show a "Boot" in their C drives too._

This is just... I mean, it's expected. I spend most of my time in Linux filesystems anyway. Windows is the alien here.
I'm just confusing myself in the translation. I won't be installing Arch tonight. But we're close.
My next big run at this is all about installing Arch (probably manually too, rather than using `archinstall`).

> **IMPORTANT!** I really also need to ensure that I back up anything critical. Most stuff already is; I just need to double-check it.

---

Also keep note of this: https://wiki.archlinux.org/title/EFI_system_partition#Replace_the_partition_with_a_larger_one
It's ArchWiki's approach to creating a new EFI System Partition. USEFUL!!!!

### Step 7.1: Flash, Aaah!

So, I've downloaded Rufus (apparently I already had this installed, I've just downloaded a newer version)
and I've used this to create write an Arch Linux ISO onto a spare USB flash drive I had lying around.

Rufus also allows us to check the signatures of the file we're writing, by the way. Good for verifying the
authenticity of the Arch ISO.

### Step 7.2: Disable Secure Boot, I guess...

This _might_ be the very next thing to do. 🤔

We're otherwise kind of ready to begin an attempt at installing Arch Linux...

_...after considerably more consumption of the wiki docs, I mean. I still need to know, step-by-step,_
_what I'm going to be doing to install and configure the system._

---

It's a brand new day!

Disabling Secure Boot was straightforward, but you do have to be a ninja.
The BIOS options flash by so quickly on startup, you barely have a chance to read the keybindings.

<kbd>F12</kbd> got me into the BIOS menu on my machine. <kbd>F10</kbd> or <kbd>DEL</kbd> would have gotten me straight into the BIOS config.

In there, I have disabled both "Secure Boot" and "Fast Boot".
No guidance was given suggesting that "Fast Boot" ought to be disabled here, but I DuckDuckGoggled it and some people reported having issues when this was left enabled.
Seemed like the safest bet just to disable that.

With that done, we are ready to initiate a boot from the Arch Linux USB I already prepared and actually start setting up Arch.

### Step 7.3: Managing partitions (HARD MODE)

I know that the first couple of things I need to do when I'm setting up Arch are...

1. Assign the UEFI System Partition appropriately for the Linux boot...
2. Assign the unallocated space I freed up yesterday to the Arch Linux filesystem

This is another part that scares me, because the docs aren't super clear about how to achieve this.

In fact, the ArchWiki at this point appears to assume either a lot of pre-existing knowledge or simply that this might be handled differently per user.

That's not great. I just need clear steps.

At this point then... hmm... I'm going to need to again read and re-read some docs, some blogs.
Make sure I have access to these from a secondary device for reference.

There is a terrifying risk at this point that we accidentally overwrite the UEFI System Partition or my C: drive, I think.
Every step I take here, I need to be **SO... FUCKING... SURE OF!**

Should be fine. Should be good.

