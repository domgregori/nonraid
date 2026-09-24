# The nonraid driver and tool fork changes

15 commits the fork (`domgregori/nonraid`) has added on top of upstream
(`qvr/nonraid`), as of the merge at `28abc59` (tagged `v0.4.0`).

## Group 1: Status reports were wrong

**1. A paused check showed as "not active."** ([commit](https://github.com/domgregori/nonraid/commit/4cab140772825ac371fab2ec560cdbc5451785af))
The tool can pause a parity check. Before, the status report did not mark a paused check as
active. A paused check looked the same as no check at all. Now the report shows a paused
check as active.

**2. The status command gave bad output on a new system.** ([commit](https://github.com/domgregori/nonraid/commit/dddcbfb834fbddf931534907e267e91fe4516b47))
On a system with no array yet, the `status` command should give clean JSON, or clean text for
other output types. Before, the command gave plain text with a success code, even for JSON
mode. This broke any tool that reads the JSON. Now the command gives correct, readable output
for each mode, with a failure code.

**3. A pending disk operation showed size zero.** ([commit](https://github.com/domgregori/nonraid/commit/3fc7e499a4aa7532fd9b240725b655cec4f3e0df))
When a clear or rebuild operation is waiting to start, the tool should show its real size.
Before, the report showed zero until the operation started. This made a real pending job look
the same as a stuck, fake one. Now the report shows the real size right away.

**4. The array stayed "DEGRADED" after a fix.** ([commit](https://github.com/domgregori/nonraid/commit/b3365549fd71cc8f4b08995a138ac11b24233845))
A parity check can find and fix errors. Before, the tool kept reporting the array as DEGRADED
after a successful fix. There was no way to clear this status without running a second,
multi-hour check that found nothing. this was tested live: a real check fixed 595 errors, but
the array stayed DEGRADED. Now the tool checks if the errors were already fixed. If so, it
reports the array as healthy.

## Group 2: The system did not start the array correctly at boot

**5. The system did not load the disk driver with the correct data file before starting the
array.** ([commit](https://github.com/domgregori/nonraid/commit/b7aecade9e2c348c46aa2dd7270b4c384161ea30))
The driver needs a specific data file (the superblock) at the moment it loads. Before, nothing
made sure this happened in the right order. This could leave the data file empty after a real
disk rebuild. Now the boot process loads the driver with the correct file first, every time.

**6. A new system with no array failed the whole startup service.** ([commit](https://github.com/domgregori/nonraid/commit/b77f622dfcc7cf54d0f31bde78b1f9147c548251))
On a truly new system, there is no array yet. The start command correctly reports this and
stops. Before, this counted as a full failure of the whole boot service. Now the boot service
accepts this as a normal, expected case.

**7. The system used a variable that could break.** ([commit](https://github.com/domgregori/nonraid/commit/012e79b59618bc0bf8e319d802c236a1ed5bca7e))
The boot service used a stored variable to find the data file. If this variable was ever empty
or broken, the driver loaded with a wrong superblock file (literally named `$SUPER`). The variable was removed. The boot
service now uses the fixed, correct file path directly. `/nonraid.dat`

**8. The tool did not check for a valid file path.** ([commit](https://github.com/domgregori/nonraid/commit/b0cf6517e8506ae3b1fce5be04a6813083c1972d))
Before, giving the tool a broken file path (not starting from the top of the file system)
caused strange, hidden errors. Now the tool checks this and shows a clear error message right away. Not really needed now that superblock is a fixed path, but still good to check.

**9. The system did not save its live status before shutting down.** ([commit](https://github.com/domgregori/nonraid/commit/4de99d0df6865a055200267ff1193b36d91f6dd8))
Before, nothing made sure the true, current disk status was saved to disk before the driver
could unload. Now the system saves this status first, unless the computer is shutting down for
good (a special case, since the disk may already be unreachable then).

## Group 3: Disks were not read correctly after certain steps

**10. A disk added without a partition could not restart properly.** ([commit](https://github.com/domgregori/nonraid/commit/7148a18e4db6a114fbdd61e48628c2a5fbfcec42))
Some disks are added to the array as a whole, plain disk, with no separate partition. Before,
the tool could not find these disks again after a stop and restart. This left the array stuck,
even though the disks were healthy. We found this while testing a backup restore. Now the tool
correctly finds these plain disks again.

**11-14. The array reported the wrong health status after fixing a disk. (Four related
fixes.)**
Commits: [82ebf37](https://github.com/domgregori/nonraid/commit/82ebf37abda569fe19ba74b99d0b608b25376039),
[0aa0f5e](https://github.com/domgregori/nonraid/commit/0aa0f5e2dcb2d3928ef5cddd7fb7b5dd97ebfc04),
[844a58d](https://github.com/domgregori/nonraid/commit/844a58d597ca05ef1ce6b674bf93aeec98c1b50e),
[683438f](https://github.com/domgregori/nonraid/commit/683438f1db95fb8512ab940be6046e1722446b96).
These four changes all fix the same root problem, one layer at a time.

- After a successful disk rebuild, the system used old counts of "how many disks are OK,
  missing, or broken." These counts did not update correctly. This made a fully healthy array
  show as DEGRADED.
- Added a function that recalculates all disk counts from scratch, using the real current
  state.
- Found and fixed two smaller bugs inside that new function (it miscounted present disks,
  and it miscounted empty slots as broken).
- Used this same recalculation in two more places (starting the array, and handling a
  write error), not just after a rebuild. This closes the door on the same kind of
  miscounting happening in those other places too.

Net result: array health status is now correct in more situations, especially after replacing
or rebuilding a disk.

## Group 4: A new fix for LUKS-encrypted disks (future feature)


**15. Stopping the array could fail if a LUKS-encrypted disk was still closing.** ([commit](https://github.com/domgregori/nonraid/commit/de1f7d1908bc78b516a7ccef692cebc083b12ca1))
When a disk uses LUKS encryption, closing it takes a small amount of extra time. Before, if
the array tried to stop during this short window, the stop command failed outright. Now the tool waits a moment and tries again automatically, up to five
times, before it reports a real failure.

---
Thank you qvr for all your work on this driver and tool, and thank you everyone else for coming to my ted talk.