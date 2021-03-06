# mmkparser Tutorial

MMK files can import tuning data directly from wav2brr. To do that, place `build.cmd` in the same folder as `convert_brr.cmd` and `tuning.yaml`.

```cmd
@echo off
set folder=..\addmusick-1.1.0-beta
set NAME=zmm-final

    EITHER:
@call convert_brr.cmd
    OR:
python -m amktools.wav2brr wav ..\addmusick-1.1.0-beta %NAME%

python -m amktools.mmkparser %NAME%.mmk || goto :eof
copy %NAME%.txt %folder%\music\%NAME%.txt
```

Rename your .txt song to `%NAME%.mmk` (in this case `zmm-final.mmk`).

## Automatic Tuning

In the .mmk file, under the #instruments section, locate each sample generated by wav2brr and remove the last 2 (tuning) bytes:

```
"strings.brr"   $8F $E0 $00 $02 $1b
```

And add `%tune` to the beginning of the line:

```
%tune "strings.brr"   $8F $E0 $00
```

And tuning data will be automatically extracted from `tuning.yaml`.

- Only one sample per line. Avoid `%tune "strings.brr" $8F $E0 $00 %tune "piano.brr" $8F $E0 $00`
- Do not add comments after ADSR. Avoid `%tune "strings.brr" $8F $E0 $00 ;slow strings`
    - The tuning values will be commented out. This is a mmkparser bug and may be fixed later.

## ADSR

Use the `%adsr` command to automatically compute ADSR. It takes 4 parameters: `attack speed`, `decay speed`, `sustain level`, `release speed`.

- Attack speed
    + 0 is slowest, 15 or -1 is fastest.
- Decay speed
    + 0 is slowest, 7 or -1 is fastest.
- Sustain level
    + 0 to 7 correspond to `decay` ending at (1/8 to 8/8).
    + `full` and -1 map to 7, or "decay envelope is mostly ignored".
- Release speed
    + 0 is slowest (infinite release, 0 speed), 31 or -1 is fastest.

`%adsr` works both within `#instruments` and in music data.

- Within #instruments:
    + `%tune "strings.brr"   %adsr -1,0,full,0`
    + results in a fast attack (2 milliseconds) and no decay over time.
- Outside #instruments:
    + `%adsr -1,0,full,0    c4`
    + replaces the current instrument's ADSR envelope.

**NOTE:** The AddmusicK readme's ADSR calculator is inaccurate and misleading. `Attack` is a linear rise, while `decay` and `release` are exponential decay with adjustable slope/τ, and `sustain level` controls when the `decay` envelope ends and `release` begins.

## GAIN

Use the `%gain` command to automatically compute GAIN. It takes 2 parameters: `curve`, `rate`. `curve` can be one of the following:

- `direct`
    + `rate` determines the loudness of the sample. 0 is silent, {$7f 0x7f 127} is loudest.
- `down`, `exp`, `up`, `bent`
    + `rate` determines the speed of the curve. 0 is no change (constant envelope), {$1f 0x1f 31} is fastest.
    + `down` and `exp` are decreasing envelopes.
    + `up` and `bent` are increasing envelopes.

## Volume and pan scaling

All lowercase `v` and `y` are interpreted as volume and pan commands, and volumes can be rescaled using `%vmod factor`, (deprecated: `%isvol` `%notvol` `%ispan` `%notpan`).

**All "word=something" where the word contains lowercase `v` or `y` will lead to errors when used.**

I may fix this by ignoring all `v` or `y` commands when rescaling is disabled, and/or ignoring all letters not followed by a number.

## Remote Commands

MMK has no special support for remote commands. Use this template instead:

```
"clear=0"
"kon=-1"
"after=1"
"before=2"
"koff=3"
"now=4"
```

Example of use:

```
(!1)[$f4$09]
(!2)[%gain exp $0c ]
"LONG_DECAY=(!1, kon)(!2, koff)"    ; restore on keyon, decay on keyoff
```

I included a space after `%gain exp $0c`. At the moment, this is necessary for my parser to work. (I could add brackets/parentheses as delimiters if necessary.)

Do not define or use (!0), it will not work in AddmusicK (this is an undocumented AMK restriction).

## Result

Run `build.cmd`. This will compile `zmm-final.mmk` to `zmm-final.txt` and copy it into the AddmusicK folder. You can edit `build.cmd` to automatically build the song, but this varies by version.

AMK 1.0.x (unsure):

```
cd %folder%

addmusick -norom -noblock %NAME%.txt
start SPCs\%NAME%.spc
```

AMK 1.1 beta April 2017:

```
cd %folder%

addmusick -m -noblock %NAME%.txt
start SPCs\%NAME%.spc
```

AMK 1.1 beta May 2017:

```
cd %folder%
echo 01 %NAME%.txt> Trackmusic_list.txt

addmusick -m -noblock
start SPCs\%NAME%.spc
```

AMK Beta may sometimes corrupt samples (happens in May 2017, possibly April 2017).
