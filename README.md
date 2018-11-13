# Human Archive (.hrx)

This is a specification for a plain-text, human-friendly format for defining
multiple virtual text files in a single physical file, for situations when
creating many physical files is undesirable, such as defining test cases for a
text format.

[multipart/mixed]: https://www.w3.org/Protocols/rfc1341/7_2_Multipart.html

Here's a sample HRX that contains two files:

```hrx
<===> input.scss
ul {
  margin-left: 1em;
  li {
    list-style-type: none;
  }
}

<===> output.css
ul {
  margin-left: 1em;
}
ul li {
  list-style-type: none;
}
```

HRX files are always encoded in UTF-8.

## Goals

The HRX format is intended to make it easy to represent multiple chunks of
plain-text data in a single physical file. It's intended to be easy for humans
to read, edit, and create with simple text editors. It's intended to be easy to
read and modify programatically, while generating easy-to-understand diffs for
version control. It's intended to integrate well with tooling for
syntax-highlighting or providing other language-specific support for individual
files it contains.

### Non-Goals

The HRX format is not intended as a wire format. The [multipart/mixed][] format
defined by MIME is already well-suited to that need.

The HRX format is not intended to faithfully represent any arbitrary directory
structure. In order to ensure simplicity and human-readability, it intentionally
lacks support for binary data or complex permissions.

## Syntax

The syntax for a HRX file is as follows:

<x><pre>
**archive**        ::= **entry*** **comment**?
&#32;
**entry**          ::= **comment**? (**file** | **directory**)
**comment**        ::= **boundary** **newline** **body**
**file**           ::= **boundary** " "+ **path** **executable**? **newline** **body**?
**directory**      ::= **boundary** " "+ **path** "/" **newline**+
**boundary**       ::= "<" "="+ ">" // must exactly match the first boundary in the file
**executable**     ::= " "+ "+x"
**newline**        ::= U+000A LINE FEED
**body**           ::= **contents** **newline** // no newline at the end of the file
**contents**       ::= any sequence of characters that does not include U+000A
&#32;                  LINE FEED followed immediately by **boundary**
&#32;
**path**           ::= **raw-path** | **quoted-path**
**raw-path**       ::= **path-component** ("/" **path-component**)*
**quoted-path**    ::= '"' **path-component** ("/" **path-component**)* '"'
**path-component** ::= (**path-character** | '\"')+ // not equal to "." or "..", not a **drive**
**path-character** ::= any character other than U+0000 through U+001F, U+007F DELETE, U+0022
&#32;                  QUOTATION MARK, U+002F SOLIDUS, or U+005C REVERSE SOLIDUS; or U+0020 SPACE
&#32;                  if part of a **raw-path**
**drive**          ::= [A-Za-z] ":"
</pre></x>

Each `path` in the file must be unique, not including surrounding quotation
marks. It's invalid for an initial subsequence of a path's components to be the
same as another `file`'s path.

> The length of a `boundary` must be consistent throughout a given HRX file.
> Longer or shorter `boundary`s may be included in `content` blocks, which means
> a boundary may be chosen for the HRX file that will allow its `content` blocks
> to contain arbitrary text.
>
> `content` blocks don't contain the trailing newline in `body`. However, the
> last `body` in the file doesn't have a trailing newline, so if the file ends
> with a newline it *is* included in its `content` block.
>
> Directories are distinguished from empty files by the trailing "/" in the path
> name.

## Semantics

A HRX file, represented as an `archive`, contains a sequence of files and/or
directories, each represented as an `entry`. Each entry has an optional
`comment` that's intended to allow an archive author to include documentation or
notes relating to that entry. This comment must not affect the contents or
behavior of that entry.

Each entry has a `path` which represents the relative path from the root of the
archive to that entry. Each `path` is divided by U+002F SOLIDUS characters into
separate path components. Each component except for the last one represents a
nested series of directories; the last component represents the entry itself,
which is either a file (for a `file`) or a directory (for a `directory`). Parent
directories are implicitly assumed to exist if they have entries beneath them,
even if they aren't explicitly written as `directory`s.

Each path component is a string that represents the name of that component. Path
component names are interpreted exactly as they appear in the archive file, with
the exception that the sequence `\"` is interpreted as U+0022 QUOTATION MARK.

File entries with an associated `body` have their contents given by the `body`'s
`contents` block. File entries with no `body` have empty contents. If a file has
an `executable` marker, that indicates that the file is meant to be marked
executable when extracted.

### Extracting

Although HRX files are primarily intended to be used as a replacement for
multiple physical files, they can be extracted to the physical filesystem. When
a HRX file is extracted, the extraction process should (by default) create a
directory named after the HRX file, with the extension `".hrx"` removed.

The permissions of extracted files should match those of the original HRX file,
except that any `file`s with an `executable` flag should be marked executable on
disk if the operating system supports such a distinction.
