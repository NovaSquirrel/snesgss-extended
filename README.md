# snesgss-extended
This is an enhanced version of the SNESGSS music engine for SNES homebrew games.
Because the original tracker's exporter is hardcoded for the orginal SNESGSS driver, I made a new one in Python 3 that's based on the original export code.

New features include
* Local samples - have per-song samples that get loaded in alongside the songs that use them
* Commands to enable and configure echo
* Ability to play sound effects on music channels
* Take multiple SNESGSS module files as input and combine them
* Acknowledge SPC driver commands in a safer way

The faster SPC700 loader from [libSFX](https://github.com/Optiroc/libSFX/) is included. The music compression could also use some improvements - it seems to be particularly slow in Python, and doesn't try to compress as aggressively as it could have in order to try taking less time.

# SPC700 Driver
The driver has been converted to use a ca65 macro pack, allowing it to integrate with a ca65 program more easily, and display debugging information in Mesen-S. It's interfaced with similarly to the original driver, but the command IDs have been changed to make room for more commands.

The way you send commands has also been changed - you're supposed to read APU3 ($2143), wait for APU0 ($2140) to be nonzero, write the new command to APU0 (as well as any parameters to APU1-3), and then wait until APU3 has a different value than it had before.

# Exporter
The `exportgss.py` script handles taking SNESGSS modules and turning it into data the driver can take. Syntax is `exportgss.py data.s enum.s List.gss Of.gss Songs.gss`

The exporter outputs a data file (ca65 format) as well as a file containing a `Music` enum, and a `SoundEffect` enum, giving you an easy way to get the ID of the song or sound effect you want to play.

Local samples are specified by prefixing the instrument name with `L_` but you can change the prefix in `exportgss.py`, as well as decide whether you want the exporter to accommodate streaming or not.

# BRR conversion tool
BRR conversion is split into a separate program, just using the original BRR conversion code from SNESGSS. It could be converted into Python later if desired, but it was a lot easier to just leave as-is than everything else.
It should be easy to build with any C compiler - `gcc gssbrr.c -o gssbrr` for example.

The BRR conversion tool is used by calling it with the command-line parameters `length loop_start loop_end loop_enable loop_unroll souce_rate source_volume eq_low eq_mid eq_high resample_type downsample_factor ramp_enable source` in order, and it outputs three text lines, consisting of the BRR size (in bytes, decimal), BRR loop point (in bytes, decimal), and the converted sample as a long hexadecimal string.

# ld65 linker config
You'll want to add something like this to the MEMORY section of your linker configuration:
```
  SPCZEROPAGE: start =    $0004, size = $00EC;
  SPCRAM:      start =    $0200, size = $FDC0;
```

And something like this to your SEGMENTS section (replacing ROM15 with wherever you want to store the data):
```
  SPCIMAGE:   load = ROM15, run=SPCRAM, align = $100, define=yes;
  SPCZEROPAGE:load = SPCZEROPAGE, type=zp, optional=yes;
  SPCBSS:     load = SPCRAM, type = bss, align = $100, optional=yes;
```

The exporter will also use a segment named `SongData` and will currently put all of the music data (including local samples) in that segment.

# 65816-side API

65816-side functions are in `blarggapu.s`. You can call `spc_boot_apu` to load the driver and global music data, then you want to send the `INITIALIZE` command. Commands are sent by calling `GSS_SendCommand` with the command ID in the accumulator. For commands that take parameters, `GSS_SendCommandParamX` will use X, with the low byte going to APU2 and high byte going to APU3.

# SPC driver commands

## GLOBAL_VOLUME
* APU2 = Volume
* APU3 = Change speed

## CHANNEL_VOLUME
* APU2 = Volume
* APU3 = Channel mask

## SFX_PLAY
* APU1 = Volume 
* APU2 = Effect number
* APU3 = Pan

Starts a sound effect. The channel to play on is given in the upper nybble of the command ID - see the following example:
```
lda #255
sta APU1 ; volume
lda #SoundEffect::Jump
sta APU2 ; effect
lda #128
sta APU3 ; Pan
lda #GSS_Commands::SFX_PLAY|$80
sta APU0
```

When a song starts, it reserves a number of channels for music playback, starting from the first one. Sound effects cannot play on reserved channels, but you can specify "channel" 8, 9 or 10, and it will play on a virtual channel corresponding to channels 7, 6 and 5 respectively. When a sound effect is playing on a virtual channel, the music stops using the corresponding real channel and the sound effect plays instead.

## INITIALIZE
* No parameters
Must be called after loading the driver in - initializes variables and DSP registers.

## STEREO
* APU2 = Stereo, if nonzero
The driver defaults to mono - change it here if needed.

## MUSIC_START
* No parameters
Starts playing whatever song has been loaded into audio RAM.

## MUSIC_STOP
* No parameters
Stops all music channels and marks all channels as free for sound effects.

## MUSIC_PAUSE
* APU2 = Pause if nonzero

## STOP_ALL_SOUNDS
* No parameters
Stops all channels, including any sound effects.

## FAST_LOAD
* APU2 = Number of pages (256 byte units) to send over
Starts the faster loader, with the destination hardcoded to the place music should be loaded. `GSS_LoadSong` can use this.

## LOAD
Jumps to the built-in SPC bootloader at `$ffc9`.

## ECHO_VOLUME_CHANNELS
* APU2 = Volume
* APU3 = Channel mask

## ECHO_ADDRESS_DELAY
* APU2 = High byte of the start of the echo buffer 
* APU3 = Echo delay (and buffer size) in 2KB chunks. 0 is invalid, and represents just 4 bytes.

## ECHO_FIR_FEEDBACK
* APU2 = FIR set
* APU3 = Feedback volume (the volume at which the echo-output is mixed back to the echo-input).

From FullSNES:
* Volume (-128..+127) (negative = phase inverted) (sample=sample*vol/128)
* Medium values (like 40h) produce "normal" echos (with decreasing volume on each repetition). Value 00h would produce a one-shot echo, value 7Fh would repeat the echo almost infinitely (assuming that FIRx leaves the overall volume unchanged).

To change the FIR sets available you can go to `fir_table` in the SPC driver. The preset options are 0=simple echo, 1=multi echo, 2=low pass echo, 3=high pass echo.
