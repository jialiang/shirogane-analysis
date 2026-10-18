# Shirogane no Soleil: extracting and repacking the script text

How the text in 白銀のソレイユ -Successor of Wyrd- (2007, ExHIBIT engine)
is stored, how to get it out and how to put edited text back so the game
runs. Recorded 2026-10-18.

Terms used throughout: CP932 is Microsoft's Shift-JIS, the Japanese
encoding the game uses. u32 is an unsigned 32-bit integer and i32 a signed
one, always little-endian here. An op is one script instruction.

## Where the files are

1. The game reads its scripts from the `rld\` folder next to
   `ExHIBIT.exe`: 418 `.rld` files, 5,453,265 bytes in all. Most hold one
   scene each, plus support files such as `defChara.rld` (the character
   definitions). Nothing else holds dialogue. Menus and system screens are
   images, not text.
2. The files are not encrypted or compressed. Keep a copy of the original
   folder before changing anything: it is the only way back.

## The file format

### Header

| bytes          | content                                       |
| -------------- | --------------------------------------------- |
| 0-3            | magic `00 44 4C 52` (`\0DLR`)                 |
| 4-7            | u32 version: 1 in every file                  |
| 8-11           | u32 offset of the first op: 276 in every file |
| 12-15          | u32 number of ops                             |
| 16 to first op | not needed for text work: copy it unchanged   |

### Ops

The ops follow the header back to back until the end of the file. Each op
is:

1. A u32 op word, split into four fields (bit 0 is the least significant):
   1. `word & 0xFFFF`: the opcode,
   2. `(word >> 16) & 0xFF`: how many i32 values follow,
   3. `(word >> 24) & 0xF`: how many strings follow the values,
   4. `word >> 28`: flags of unknown meaning (0, 1 or 2 seen): keep them.
2. That many i32 values.
3. That many strings, each CP932 text ended by a NUL byte.

A parse is correct when it ends exactly at the end of the file. The game
uses 107 different opcodes. Only five carry text a player sees.

### Text-carrying ops

| opcode | what it is                   | strings, in order                                | ops (all files) |
| ------ | ---------------------------- | ------------------------------------------------ | --------------- |
| `0x1C` | a dialogue or narration line | 2: the speaker label, then the line              | 39,213          |
| `0x0C` | a scene title                | 1: the title as shown in save slots              | 291             |
| `0x15` | a choice menu                | 10: one option each, `*` for an unused slot      | 71              |
| `0x71` | text drawn over the screen   | 1: shouts and effects, or an ASCII resource name | 87              |
| `0x30` | a character definition       | 1: comma-separated fields, one of them the name  | 37              |

1. **`0x1C` lines** carry 7 i32 values before their two strings. The
   speaker label is the name shown above the text box. Narration has the
   label 記述 ("narration"), which is also shown. Each line carries its
   own copy of the label, so the same label appears thousands of times:
   an editor will usually want to edit each distinct label once and write
   it back to every op that uses it.
2. **`0x0C` titles** read `<number> <title> [<counter>]`, for example
   `001-1 <title> [1]`. The number may hold digits, capitals, `-` and `_`.
   The separator before `[` can be an ASCII or a fullwidth space. Keep the
   number and the counter when editing a title.
3. **`0x15` choices** hold 169 real options in their 71 menus. Leave the
   `*` placeholders as they are.
4. **`0x71` overlays:** of the 87, 71 hold Japanese text and 16 are ASCII
   resource names: leave those alone.
5. **`0x30` definitions** live in `defChara.rld`, one per character. The
   string has 14 comma-separated fields, and the name is field 3 counting
   from 0 (the fourth value): `<id>,0,1,<name>,,,,,,,,,,`. These names are
   independent of the `0x1C` speaker labels: 22 of the 37 also appear
   among the 69 distinct labels, but editing one never changes the other.

### Why lengths can change

No op stores the byte position of another: jumps refer to label numbers,
not file offsets. A string can therefore grow or shrink freely, and the
file needs no other change. A build in which almost every one of the
78,994 text strings changed length (two per `0x1C` line, plus the titles,
choice options, Japanese overlays and names) runs and branches correctly
in the game.

## Extracting

For each file, parse the ops and keep every string of the five opcodes
above, with:

1. the file name, the op's index in the file and the string's index within
   the op, both counting from 0 (for a `0x1C` op, string 0 is the speaker
   label and string 1 the line). These three are all that repacking needs.
2. the kind of string (line, speaker, title, choice, overlay or name),
3. the text itself. For a `0x30` definition, keep only field 3.

Skip empty strings, the `*` choice placeholders and overlays that are
ASCII-only (every byte below 0x80). ASCII-only strings of the other
opcodes are kept.

## Repacking

1. Parse each **original** file again.
2. For every edited string, replace the string at its op and string index.
   For a `0x30` definition, split on commas, replace field 3 and join
   again.
3. Encode each string as CP932. Report a string that fails and keep the
   original there (see "Writing new text"). Never write a comma into a
   character name: it shifts every later field.
4. Write the file back: every byte from 0 up to the first-op offset
   unchanged, then each op as its op word (unchanged), its i32 values
   (unchanged) and its strings, each followed by a NUL. The op count and
   the string counts need no update: repacking replaces text but never
   adds or removes an op or a string.
5. Copy the rebuilt files into the game's `rld\` folder.

With no edits, the rebuilt files match the originals byte for byte (all
418). That round trip is the check that the parser is right.

## Testing in the game

On a Western (non-Japanese) Windows locale the game starts, but its
branching logic misbehaves. A locale-fix `version.dll` placed next to
`ExHIBIT.exe`, which the game loads from its own folder, makes it behave
as on a Japanese locale without changing the system's settings. The
in-game checks described here were run with it in place.

## Writing new text

### Encoding

1. Text must encode to CP932. ASCII is safest: anything else is drawn as a
   fullwidth character and looks oddly spaced beside ASCII text.
2. Must convert, because CP932 lacks them: the em dash U+2014 (write `--`,
   or CP932's ― U+2015) and accented letters (write the plain letter).
3. Should convert, because CP932 draws them fullwidth: curly quotes (write
   `'` and `"`), `…` (write `...`) and the en dash (write `-`).
4. `♪` is in CP932 and works, but is drawn fullwidth.

### Line breaks and the text box

A line break inside a string is a single LF byte (0x0A). 549 of the
Japanese lines contain one. The engine word-wraps ASCII text by itself, so
a break is only needed where one is wanted. A line too long for one text
box carries on into the next box with nothing cut off (checked on lines
over 200 characters).

### Markup in the Japanese

1. **Reading hints** are written `《漢字:よみ》`: a word in Chinese
   characters, a colon, then its pronunciation in Japanese syllables. The
   engine treats them as ordinary characters with no special meaning, so
   an edited line can simply leave them out.
2. **Thoughts** are a whole line wrapped in fullwidth parentheses
   `（…）`. This is a writing convention only: the engine does nothing
   special with them, so ASCII `( )` serve as well.

### Custom symbols

Some lines contain characters from CP932's user-defined area (lead byte
0xF0), which the game's own font draws as symbols. Use a CP932 codec that
maps that area to the Unicode private-use code points below (Python's
`cp932` does), or keep the raw bytes: either way they repack unchanged.

| code point | CP932 bytes | occurrences | drawn as        |
| ---------- | ----------- | ----------- | --------------- |
| `U+E000`   | `F0 40`     | 593         | ♥ filled heart  |
| `U+E001`   | `F0 41`     | 79          | ♡ outline heart |
| `U+E009`   | `F0 49`     | 8           | sweat drops     |
| `U+E00B`   | `F0 4B`     | 8           | dizzy swirl     |
| `U+E006`   | `F0 46`     | 2           | anger mark      |
| `U+E02F`   | `F0 6F`     | 2           | shock spike     |
| `U+E01A`   | `F0 5A`     | 1           | sweat drop      |

Whether they render correctly next to ASCII text is untested.

## Going back to Japanese

Copy the original `.rld` files back over the game's `rld\` folder. Don't
expect saves made with an edited build to load with the original scripts
(untested).
