I'm terribly annoyed by the fact that grep(1) cannot look for binary
strings. I'm even more annoyed by the fact that a simple search for 
"binary grep" doesn't yield a tool which could do that. So I ~~wrote one~~ *gratuitously forked one*.

**Original work by [tmbinc/bgrep](https://github.com/tmbinc/bgrep)**

Feel free to modify, branch, fork, improve. Re-licenses as BSD.

### Building
First, you need to install `gnulib`.  On Debian and derivatives:
```
sudo apt-get install gnulib
```

Once you have that, it is the normal autotools build process.
```
./configure
make
make check
```
And optionally: (*note: this part is untested*)
```
sudo make install
```

### How it works

```
$ src/bgrep -h
bgrep version: 0.3
usage: src/bgrep [-hfc] [-s BYTES] [-B BYTES] [-A BYTES] [-C BYTES] <hex> [<path> [...]]

   -h         print this help
   -f         stop scanning after the first match
   -c         print a match count for each file (disables offset/context printing)
   -s BYTES   skip forward to offset before searching
   -B BYTES   print <bytes> bytes of context before the match
   -A BYTES   print <bytes> bytes of context after the match
   -C BYTES   print <bytes> bytes of context before AND after the match

      Hex examples:
         ffeedd??cc        Matches bytes 0xff, 0xee, 0xff, <any>, 0xcc
         "foo"           Matches bytes 0x66, 0x6f, 0x6f
         "foo"00"bar"   Matches "foo", a null character, then "bar"
         "foo"??"bar"   Matches "foo", then any byte, then "bar"

      BYTES may be followed by the following multiplicative suffixes:
         c =1, w =2, b =512, kB =1000, K =1024, MB =1000*1000, M =1024*1024, xM =M
         GB =1000*1000*1000, G =1024*1024*1024, and so on for T, P, E, Z, Y.
```

### Examples
**Basic usage**
```
$ echo "1234foo89abfoof0123" | bgrep \"foo\"
stdin: 00000004
stdin: 0000000b
```
**Count findings option**
```
$ echo "1234foo89abfoof0123" | bgrep -c \"foo\"
stdin count: 2
```
**Find-first option**
```
$ echo "1234foo89abfoof0123" | bgrep -f \"foo\"
stdin: 00000004
```
**Overlapping matches**
```
$ echo "oofoofoofoo" | bgrep \"foof\"
stdin: 00000002
stdin: 00000005
```
**Wildcard matches**
```
$ echo "oof11f22foo" | bgrep '66????66'
stdin: 00000002
stdin: 00000005
```

### Extreme example

Suppose you have a corrupt bzip2-compressed TAR file called `backups.tar.bz2`.  You want to locate TAR headers within this file.  You do know that all the paths start with "home", since it was your /home volume you had backed up.

Looking [here](https://www.gnu.org/software/tar/manual/html_node/Standard.html), you learn that a filename is typically in the `name[100]` field of the TAR file header.  There is a magic number `magic[6]`, starting 257 bytes after the start of `name[100]`.
The magic number for GNU TAR files is "ustar  ".

Let's get to work breaking down and reconstructing that archive!  First we take advantage of bzip2's block CRC property to find non-corrupt blocks.

```
$ bzip2recover backups.tar.bz2
```
This produces many small files, each called "rec00001backups.tar.bz2", "rec00002backups.tar.bz2", etc.  We can then string them together until we hit a bad one, searching for TAR headers...

```
$ bzcat rec*backups.tar.bz2 | bgrep -f  '"home"??????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????????"ustar  "'
```
This will locate the first place in the stream where "ustar  " is preceeded by "home", 257 bytes earlier (257-4==253 wildcard bytes).  This is a strong indication that you've found the first GNU tar header.  We can then split the archive at the tar header (tar can expand any files if you start it at a header), then reconstruct files up to the next bad bzip2 block.  After the bad block, we can resume this process as needed until we squeeze all we can out of that corrupt archive.
