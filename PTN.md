The SP404sx binary pattern format
=================================

This document is largely based on an earlier 
[blog](http://byteflip.club/sp-edit/roland-sp404sx-ptn-format) post. Other
reverse engineering efforts have been recorded in threads on the SP404 
forum
[here](http://sp-forums.com/viewtopic.php?p=60635&sid=820f29eed0f7275dbeaf776173911736#p60635)
and
[here](http://sp-forums.com/viewtopic.php?p=60693&sid=820f29eed0f7275dbeaf776173911736#p60693).

The SP404sx stores its rhythm patterns in a binary format. The files
are located on the SDs in `/SP-404SX/ROLAND/SP-404SX/PTN/`. The file names
are constructed as `PTN00xxx.BIN` where `xxx` is the number of the pad as
counted from A1 to J12, i.e. 001..120. Because the patterns are stored in
binary, the files need to be accessed with a file handle in binmode (e.g. 
`rb` in Python). The data is then read in hexadecimal, with the following
interpretation having been developed by the community:

- next_sample
- pad_code
- bank_switch
- unknown1
- velocity
- unknown2
- length (note: 2 bytes)

## next_sample

A value between 0x00 (0) and 0xff (255) that indicates when the next 
sample in the pattern is to be triggered. When multiple samples are 
triggered at the same time, this value stays 0 until the last of the 
simultaneous samples, which has the time till next. The value is 
interpreted as ticks, where the SP404sx has a resolution of 96 PPQ, 
resulting in a bar length of 384 ticks. Because the value can't cover
an entire bar, it seems there are 'spacer' events (with 0 velocity and
0 length) that are introduced to span gaps (?)

## pad_code

Identifies the midi note pitch to be triggered. The values that can
be emitted correspond with those shown on page 47 of the manual, i.e.
ranging from `0x2f / 47 / note B2` to `0x6a / 106 / note A7#`. Values 
outside this range are emitted for the 'spacer' events, where the value 
is `0xff / 128`.

## bank_switch

A value that indicates whether the event is a 'spacer' (`0x00 / 0`), 
or a note event that accesses one of the lower banks A..E 
(`0x40 / 64`) or one of the upper banks F..J (0x41 / 65). The 
SP404sx uses a coding where the midi pitches (`pad_code`) are reused,
such that 0x2f is both bank/pad A1 as well as F1, which are 
distinguished by the `bank_switch` being 64 or 65 respectively. For
conversions to midi, this switch can be used to identify the midi
channel over which to transmit: the 'base' channel as configured
on the device for the lower banks, or 'base + 1' for the upper
banks.

## unknown1

The value seems to always be 0x00. Possibly this is reserved by
Roland to add functionality in firmware upgrades that never 
materialized? (Would be interesting to experiment with and see if
anything happens.)

## velocity

Stores the raw value for the velocity. The pads of the SP404sx
are not velocity sensitive, and the default value for their triggers
is set to `0x7f / 127`, i.e. the maximum as per the midi standard.
Sequencing the SP from a DAW allows for lower values. A value of
`0x00 / 0` is used for the 'spacer' events.

## unknown2

The value is either `0x00 / 0`, for 'spacer' events, or `0x40` for
note events.

## length

The length of the event. Because this uses two bytes, this gives
a theoretical range from `0x0000 / 0` to `0xffff / 65535`. Because
the SP uses 384 ticks per bar and can store patterns up to 99 bars,
the two byte value has enough headroom.

# Pattern file footer

PTN files contain a footer of two rows, like so:

```
0x00 0x8c 0x00 0x00 0x00 0x00 0x00 0x00 
0x00 0x04 0x00 0x00 0x00 0x00 0x00 0x00 
```

The value `0x8c / 140` seems invariant (no, it's not the BPM). The
other value, in this example `0x04 / 4` is variable, and specifies
the length of the pattern.