---
layout: post
title:  "Fixing a Corrupt Git Index"
date:   2024-3-22 13:20:00 -0400
categories: jekyll update
---
I've been building a git implementation in Ruby called **[Jit]((https://github.com/Garrett-Bodley/Jit))** as part of a **[Recurse Center](https://www.recurse.com/)** study group that's reading **[Building Git](https://shop.jcoglan.com/building-git/)** by James Coglin.

<!-- Part of the fun (and challenge) of the book is that you use your own implementation to version control itself. The early stages of your implementation cannot handle nested subdirectories, so all your files must be kept in a flat directory. This limitation is frustrating, but makes the victory of successful adding subdirectory support all the more significant!. Similarly, early versions of -->

I finished building a feature and was ready to commit. I staged the modified files via `jit add` and ran `jit status` when I encountered this output:

{% highlight txt %}
% jit status
fatal: Checksum does not match value stored on disk
{% endhighlight %}

Uh oh, had I corrupted my index file somehow?

## Inspecting the Index on disk

The checksum refers to the final 20 bytes stored in `.git/index` which is a `sha1sum` of the everything in the index that precedes those final 20 bytes.

Okay, let's compute the checksum via the command line and inspected the last 20 bytes of my `.git/index` file

{% highlight zsh %}
% wc -c .git/index

    3526 .git/index

% cat .git/index | head -c 3506 | sha1sum

fdcf9b375d7f21c2a90aa68440925d5976ae2e54  -

% cat .git/index | tail -c 20 | hexdump -C

00000000  fd cf 9b 37 5d 7f 21 c2  a9 0a a6 84 40 92 5d 59  |...7].!.....@.]Y|
00000010  76 ae 2e 54                                       |v..T|
00000014

{% endhighlight %}

What the heck? The hashes match!

Let's throw a breakpoint in my code and inspect what's going on:

{% highlight pry %}
% jit status

From: jit/lib/index.rb:87 Index#load:

    80: def load
    81:   clear
    82:   file = open_index_file
    83:   if file
    84:     reader = Checksum.new(file)
    85:     count = read_header(reader)
    86:     read_entries(reader, count)
 => 87:     binding.pry
    88:     reader.verify_checksum
    89:   end
    90: ensure
    91:   file&.close
    92: end

[1] pry(#<Index>)> reader.digest.digest.bytes.map { |byte| byte.to_s(16) }.join
=> "ef7552992140e6169ca2cf2ba9db2bc47115d96"
{% endhighlight %}

Well, that's definitely a different checksum. So what's happening here?

## A Higher Level Perspective

It might help to explain what the status command is doing.

The first thing the `status` command command does is read `.git/index` into memory before using that in-memory representation to determine if any files in your workspace have changed and if anything in your index differs from your previous commit. The checksum error I'm getting happens in that very first step, reading the index into memory.

## Examining the Code

{% highlight ruby %}
# lib/index.rb

def load
  clear
  file = open_index_file
  if file
    reader = Checksum.new(file)
    count = read_header(reader)
    read_entries(reader, count)
    reader.verify_checksum
  end
ensure
  file&.close
end
{% endhighlight %}

The first 12 bytes of `.git/index` constitute a header that specifies (amongst other things) how many entries are stored in the index. That's the return value of `read_header` which we store in `count`.

Looking at the `read_entries` method shows this:

{% highlight ruby %}
# lib/index.rb

def read_entries(reader, count)
  count.times do
    entry = reader.read(ENTRY_MIN_SIZE)

    until entry.byteslice(-1) == "\0"
      entry.concat(reader.read(ENTRY_BLOCK))
    end

    store_entry(Entry.parse(entry))
  end
end
{% endhighlight %}

Each index entry is null terminated, so `read_entries` reads from the index until it finds a null byte. It then stores that entry in memory, repeating that process `count` times.

What's that reader object doing?

{% highlight ruby %}
# lib/index/checksum.rb

def initialize(file)
  @file = file
  @digest = Digest::SHA1.new
end

def read(size)
  data = @file.read(size)

  raise EndOfFile, 'Unexpected end-of-file while reading index' unless data.bytesize == size

  @digest.update(data)
  data
end

def verify_checksum
  sum = @file.read(CHECKSUM_SIZE)

  unless sum == @digest.digest
    raise Invalid, 'Checksum does not match value stored on disk'
  end
end
{% endhighlight %}

The `reader` object reads a specified number of bytes from a file, updates the `sha1sum` of everything it's read, and returns the read chunk of data. `verify_checksum` reads 20 bytes from the file and compares that the current digest value, throwing an error if they don't match.

## A Hypothesis

After looking at my code, it seems that we're blindly trusting the value of `count` that's returned from reading the index header. If that `count` value is wrong then `verify_checksum` will return the wrong 20 bytes and everything will break. It's possible that we're reading `count` incorrectly, or the value in the header is incorrect, and both are pretty easy to check.

## Hexdump the Header

First let's see what code is saying the value of `count` is:

{% highlight irb %}
From: jit/lib/index.rb:87 Index#load:

    80: def load
    81:   clear
    82:   file = open_index_file
    83:   if file
    84:     reader = Checksum.new(file)
    85:     count = read_header(reader)
    86:     read_entries(reader, count)
 => 87:     binding.pry
    88:     reader.verify_checksum
    89:   end
    90: ensure
    91:   file&.close
    92: end

[1] pry(#<Index>)> count
=> 38
{% endhighlight %}

According to the documentation, the index header contains a 4-byte signature "DIRC", followed by a 4-byte version number, and 32-bit number of index entries.

{% highlight hexdump %}
% cat .git/index | head -c 12 | hexdump -C

00000000  44 49 52 43 00 00 00 02  00 00 00 26              |DIRC.......&|
0000000c
{% endhighlight %}

`0x26` is 38 in decimal, so it seems my code and the index header agree on the number of entries to expect.

## Manually Count the Index Entries

Each index entry contains the pathname of the entry in uncompressed ascii format, making it relatively easy to count the number of entries by hand via hexdump.

I hexdumped the whole index and started counting the entries. After the 38th entry I found something weird:

Here's the last 400 bytes of my `.git/index` file:

`index_test.rb` is the 38th entry, so I should expect to find the checksum immediately after it.
{% highlight hexdump %}
% cat .git/index | tail -c 400 | hexdump -C

00000000  00 11 04 5b 84 1e 00 00  81 a4 00 00 01 f5 00 00  |...[............|
00000010  00 14 00 00 05 2e b1 45  46 a8 0f 64 93 b6 20 10  |.......EF..d.. .|
00000020  aa 3b 6e 41 bf 8e c2 8e  a4 02 00 12 74 65 73 74  |.;nA........test|
00000030  2f 69 6e 64 65 78 5f 74  65 73 74 2e 72 62 00 00  |/index_test.rb..|
00000040  00 00 00 00 00 00 54 52  45 45 00 00 01 2e 00 33  |......TREE.....3|
00000050  38 20 34 0a e6 a2 e7 72  4e 4d b7 97 cd e9 6a e2  |8 4....rNM....j.|
00000060  b9 bc 4d 7b ae 98 3e 6c  62 69 6e 00 32 20 30 0a  |..M{..>lbin.2 0.|
00000070  d2 f1 e0 40 39 09 27 01  a4 eb 00 a8 fb 64 b6 4f  |...@9.'......d.O|
00000080  47 63 9e b1 6c 69 62 00  32 34 20 34 0a c8 1d 17  |Gc..lib.24 4....|
00000090  0e ff 81 4c a5 69 4c e6  8f 53 bb 79 e8 01 db 62  |...L.iL..S.y...b|
000000a0  f5 64 69 66 66 00 31 20  30 0a f5 69 d3 e8 5d 57  |.diff.1 0..i..]W|
000000b0  2a d4 af e6 aa 8a 0a 4f  b2 cc 6e 6b 61 71 69 6e  |*......O..nkaqin|
000000c0  64 65 78 00 32 20 30 0a  15 0a b2 e4 86 42 7e de  |dex.2 0......B~.|
000000d0  90 8e b4 0c 88 d6 62 d8  5f a0 52 c2 63 6f 6d 6d  |......b._.R.comm|
000000e0  61 6e 64 00 35 20 30 0a  09 56 b3 03 77 e5 ec 93  |and.5 0..V..w...|
000000f0  1e ad 3a f5 a3 57 2a ee  fe a0 d8 c1 64 61 74 61  |..:..W*.....data|
00000100  62 61 73 65 00 35 20 30  0a 84 0d cd 7d 0b 7c 42  |base.5 0....}.|B|
00000110  7a 7a 57 aa c3 d2 1b 6c  8c 32 e3 cc 72 74 65 73  |zzW....l.2..rtes|
00000120  74 00 34 20 31 0a 2a 2d  76 65 d6 35 ee cb a1 df  |t.4 1.*-ve.5....|
00000130  71 07 7d 7f 29 aa b3 5f  e9 13 63 6f 6d 6d 61 6e  |q.}.).._..comman|
00000140  64 00 32 20 30 0a 2d dd  06 b9 ec ce 35 fd a6 62  |d.2 0.-.....5..b|
00000150  ca 7b c9 d9 d9 b9 22 99  cc 06 2e 72 75 62 79 2d  |.{...."....ruby-|
00000160  6c 73 70 00 34 20 30 0a  98 d0 54 74 23 af 4a 47  |lsp.4 0...Tt#.JG|
00000170  d2 c2 0d 46 7f 9e 24 8a  0d 78 f1 67 fd cf 9b 37  |...F..$..x.g...7|
00000180  5d 7f 21 c2 a9 0a a6 84  40 92 5d 59 76 ae 2e 54  |].!.....@.]Yv..T|
00000190
{% endhighlight %}

What the heck? There's the word `TREE` followed by a bunch of binary data. That's some extra data, but it's definitely not the format of a valid index entry.

It almost looks like I've written the content of a `TREE` database object directly into my index. But I've been making commits and staging files in this project for a while. If I was writing arbitrary data into the index I surely would have encountered an issue sooner.

Also, looking closer at the hexdump we see that mixed amongst the binary are the strings `bin`, `diff`, `command`, `database`, `test`, `command`, and `.ruby-lsp`. These correspond to all the subdirectories in my workspace, but interestingly there's no reference to any actual files in this rogue `TREE` object. The trees stored in the database contain references to both files and directories, so there's no way that I've written a database tree's content into my index.

```
% find . -type d -not -path "*.git*" -not -path .

./test
./test/command
./bin
./.ruby-lsp
./lib
./lib/database
./lib/diff
./lib/index
./lib/command
```

## Documentation to the Rescue

I had pulled up the index format documentation for git while debugging. There are a lot of references to trees in that page, but **[one in particular stands out](https://git-scm.com/docs/index-format#_cache_tree)**:

```manpage
Cache tree

Since the index does not record entries for directories, the cache
entries cannot describe tree objects that already exist in the object
database for regions of the index that are unchanged from an existing
commit. The cache tree extension stores a recursive tree structure that
describes the trees that already exist and completely match sections of
the cache entries. This speeds up tree object generation from the index
for a new commit by only computing the trees that are "new" to that
commit. It also assists when comparing the index to another tree, such
as `HEAD^{tree}`, since sections of the index can be skipped when a tree
comparison demonstrates equality.

The recursive tree structure uses nodes that store a number of cache
entries, a list of subnodes, and an object ID (OID). The OID references
the existing tree for that node, if it is known to exist. The subnodes
correspond to subdirectories that themselves have cache tree nodes. The
number of cache entries corresponds to the number of cache entries in
the index that describe paths within that tree's directory.

The extension tracks the full directory structure in the cache tree
extension, but this is generally smaller than the full cache entry list.
When a path is updated in index, Git invalidates all nodes of the
recursive cache tree corresponding to the parent directories of that
path. We store these tree nodes as being "invalid" by using "-1" as the
number of cache entries. Invalid nodes still store a span of index
entries, allowing Git to focus its efforts when reconstructing a full
cache tree.

The signature for this extension is { 'T', 'R', 'E', 'E' }.

A series of entries fill the entire extension; each of which
consists of:

...
```

That must be it! In fact, if I run `git status` it seems `git` has no issue reading my index file. That would make sense seeing as this is an "index extension" and therefore unsupported by my own implementation.

## How did this happen?

Digging around my `.git` folder I found the following in `.git/logs/HEAD`

```
09bda1a2eabf18826997349410032f3d61229345 5704864b6c45d0991a1671fb196a51aa581b4398 Garrett-Bodley <garrett.bodley@gmail.com> 1710967835 -0400	commit: Stub out the Myer's Diff algorithm.
```

That's my most recent commit! But my code doesn't have any logic pertaining to `.git/logs/HEAD`.

I must have been on autopilot and used `git commit` instead of my own `jit commit` command. Git must have mutated my index and added on the "Cache Tree" extension when making the commit. My own implementation has no concept of a "Cache Tree" so that addition caused everything to break.

## Fixing the problem

Thankfully the solution to all this is straightforward: delete the index file.

Running `jit add .` will generate a new index file that is tracking the entire workspace. After deleting and recreating the index, `jit status` works entirely as expected.

## Post Mortem

I learned that while `git` works in `jit` repositories, the reverse is not true. Git is a complicated tool, and the pared down version that I've implemented doesn't cover all the edge cases that exist in the main tool.

It's rather unclear to me when a Cache-Tree gets generated. Clearly one is made when making a commit, but not when calling `git status`. I did a quick google search and haven't been able to find any posts on the subject. It seems like it exists to speed up the status command for large repositories. I tried searching the *Building Git* book for any references, but it doesn't seem to mention cache trees at all. Julia Evans, who has been digging deep into git lately, doesn't have any blog posts about cache trees either.

I also learned about `.git/logs`, which provided me concrete evidence of what had gone wrong. Logging is super useful, and I'm glad the main `git` tool keeps internal records for moments like these.

I'm tempted to try to add support for Cache Trees in my own implementation, or at least learn to skip past it to find the real checksum if one is present. But trying to match all of Git's functionality 1 to 1 feels like it could lead to an endless series of yak shaves.

All in all I'm left with greater respect for the amount of engineering that goes into a system like Git. There are so many edge cases, and small details that have been built up over the year, and trying to build my own version has given me perspective on just how much work has gone into this ubiquitous tool (though I'll try to remember to avoid `git` commands in my Jit repo)