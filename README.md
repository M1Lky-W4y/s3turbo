# s3sysex
MIDI System Exclusive interface for the General Music S3 Turbo synthesizer

## About
The General Music/GEM S2/S3 Turbo Music Processor is a synthesizer workstation from the early 90s.  The Turbo version (larger ROM and updated functionality) has support for MIDI System Exclusive which is not well-documented.  The manual includes only a rudimentary description of the available commands and data structures but the C development files are not available.

This tool provides an implementation of most of the commands from the manual.  It is also possible to send and receive sample dumps using SDS to the synthesizer (not tested with other instruments).

## Functionality
* Send commands to the synthesizer (see `S3Turbo.py`)
* Monitor MIDI messages and decode S2/S3 specific data
* Send and receive sample dumps
* Convert samples to PCM (`convert_sample.sh`)

## Usage

The MIDI device numbers can be given on the command line using `--indev` and `--outdev`. `--listdev` lists all available MIDI devices. Defaults for command line arguments can be set using a configuration file `~/.s3sysex.conf`, e.g. containing:
```
indev:  2
outdev: 3
```
Sending a basic status request to the synthesizer. The `--exit` flag causes the program to terminate after one second of inactivity from the synthesizer.
```
./s3sysex.py --command STAT_REQUEST --exit
  Data:
    ActBankPerf: (2, 1)
    FreeMem: 678632
    FreeSampleMem: 1649774
    ReadyFor: (255, 255)
    TotalMem: 1650688
    iClass: 1
    iRelease: 2
    iSubClass: 2
  checksum: 0x57 (calculated 0x57)
```

Change to Bank 1, Performance 1 and print debugging information. Additional command line arguments are appended to the message body.
```
# Bank and Performance values in ASCII (0x30-0x39)
./s3sysex.py --debug --command BANK_PERF_CHG 0x30 0x30 --exit
```

Send a directory request to the synthesizer. Uses the custom python functions `str2file` and `str2hex` to convert a string to lists of integers. `str2file` creates fixed 11 character filename padded with whitespace.  `str2hex` creates a 0-terminated list.

```
# List everything
./s3sysex.py --command DIR_DRQ 'str2file("*")' 'str2hex("A:\\")'
```

The first argument is the filename, second is the path.
A: corresponds to RAM.
B: corresponds to the current disk if loaded, to the RAM disk otherwise.
C: is either empty if no disk is loaded or corresponds to the RAM disk.

The general directory layout is
```
A:-|
   |-BANKS                 (type 32, format 32)
   | |-'BANKNAME  0'       (type 32, format 32)
   | |-'BANKNAME  1'       (type 32, format 32)
   | :
   | |-'BANKNAME  9'       (type 32, format 32)
   |   |-PERF              (type 32, format 32)
   |   | |-'PERFNAME  0'   (type  2, format  2)
   |   | |-'PERFNAME  1'   (type  2, format  2)
   |   | :
   |   | |-'PERFNAME  9'   (type  2, format  2)
   |   |
   |   |-Song              (type  3, format  2)
   |
   |-DATA                  (type 32, format 32)
   | :_(e.g. screenshot from hardcopy program in HARDCPY/FIG_001 IMG)
   |
   |-PROGRAM               (type 32, format 32)
   |
   |-SETUP                 (type 32, format 32)
     |-GENERAL             (type  5, format  2)
     |-EFFECT1             (type  6, format  2)
     |-EFFECT2             (type  7, format  2)
     |-SOUNDMAP            (type  9, format  2)
     |-SAMPLES             (type 32, format 32)
     | |-'SAMPNAMETXL'     (type  8, format  2)
     |
     |-SOUNDS              (type 32, format 32)
       |-'SOUNDNAME F'     (type  1, format  2)
       |-'SOUNDNAME E'     (type  1, format  2)

Sound suffixes:
  E - RAM sound using RAM sample
  F - RAM sound using ROM sample

File types:
  1 - Sound
  2 - Performance
  3 - Song
  4 - ?
  5 - General Setup
  6 - Effect1 Setup
  7 - Effect2 Setup
  8 - Sample
  9 - Soundmap Setup
 32 - Directory
```

## Dumps
```
TYPE = [ 0x0, # Sound
         0x1, # Sample
         0x2, # Soundmap
         0x3, # Effect1
         0x4, # Effect2
         0x5, # General
         0x6, # Song
         0x7, # Performance
         0x8, # Global       (untested)
         0x9, # StylePerf    (untested)
         0xa, # RealtimePerf (untested)
         0xb, # Riff         (untested)
]

BANK = ASCII Bank number (0x30-0x39), only used for TYPE 0x6, 0x7
PERF = ASCII Performance number (0x30-0x39), only used for TYPE 0x7

# List directory contents
./s3sysex.py --command DIR_REQUEST $TYPE $BANK $PERF 'str2file("*")'

# Dump 'file'
./s3sysex.py --command DATA_REQUEST $TYPE $BANK $PERF 'str2file("*")'

# Dump Sound ("SoundName" from DIR_REQUEST)
./s3sysex.py --command DATA_REQUEST 0 0 0 'str2file("SoundName")'

# Dump Sample ("SampleName" from DIR_REQUEST)
./s3sysex.py --command DATA_REQUEST 1 0 0 'str2file("SampleName")'

# Dump SOUNDMAP (filename argument ignored)
./s3sysex.py --command DATA_REQUEST 2 0 0 'str2file("*")'

# Dump EFFECT1 (filename argument ignored)
./s3sysex.py --command DATA_REQUEST 3 0 0 'str2file("*")'

# Dump EFFECT2 (filename argument ignored)
./s3sysex.py --command DATA_REQUEST 4 0 0 'str2file("*")'

# Dump GENERAL (filename argument ignored)
./s3sysex.py --command DATA_REQUEST 5 0 0 'str2file("*")'

# Dump Song 1 (Bank 0x30-0x39)
./s3sysex.py --command DATA_REQUEST 6 0x30 0x30 'str2file("*")'

# Dump Performance (Bank 0x30-0x39 Perf 0x30-0x39)
./s3sysex.py --command DATA_REQUEST 7 0x30 0x30 'str2file("*")'

# File dump request
./s3sysex.py --command F_DREQ 'str2file("SampNameTXL")' 'str2hex("A:\\SETUP\\SAMPLES")'

# Trigger RAM dump (probably bug)
./s3sysex.py --command STAT_REQUEST --exit
./s3sysex.py --command F_ACK --exit
```

## Formats

```
# Sound map format, variable length
 6 bytes header     01 02 02 09 02 02
18 bytes map entry
   bytes  1-10      Sound Name
   byte  11         unknown, c0 for layer, c6 for user, else c2 or 00
   byte  12         unknown
   byte  13         Bank
   byte  14         Program
   bytes 15-18      unknown, 1e6ea562 for user, else 1b237000

# Effect1 map format, variable length
 6 bytes header     01 02 02 06 02 01
34 bytes map entry
   bytes  1-10      Effect name
   bytes 11-12      zero
   byte  13         Effect number
   byte  14         unknown (7f)
   bytes 15-16      Date (18 c8)
   bytes 17-18      Time (70 00)
   bytes 19-21      unknown (00 00 00)
   byte  22         Effect type
   bytes 23-24      Level
   bytes 25-26      Effect param 1
   bytes 27-28      Effect param 2
   bytes 29-30      Effect param 3
   bytes 31-32      Effect param 4
   byte  33-34      Effect param 5

# Effect2 map format, variable length
 6 bytes header     01 02 02 07 02 01
34 bytes map entry, see Effect 1 map

# General format, possibly constant length 1127
 6 bytes header     01 02 02 05 02 01
 remaining unknown

# Performance format, possibly constant length 252
 6 bytes header     01 02 02 03 02 01
 remaining unknown
```

### Sample format
The S2/S3 exports 14-bit encoded mono samples. When receiving a sample dump, the program creates five files:
* sample_TIMESTAMP.sds: original stream of MIDI messages
* sample_TIMESTAMP.txt: sample information (loops, samplerate, etc)
* sample_TIMESTAMP.dmp: sample data dump (7-bit stream)
* sample_TIMESTAMP.raw: raw sample data (16-bit unsigned Big Endian)
* sample_TIMESTAMP.wav: PCM sample data (16-bit signed Little Endian )

The RAW file can be played back with:
```
# try samplerate*10
play  -b 16 -c 1 -e unsigned -B -t raw -r SAMPLERATE sample.raw
```

## Dependencies
* `python-pypm` Portmidi wrapper for Python
* `python-progress` Progress bar for Python
