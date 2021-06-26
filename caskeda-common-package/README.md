# Caskeda Package Format

> A application container format directly inspired by HTTP multipart-requests/responses.
>
> The name is formed by the first 4 letters of `caskeda`: `cask`.
>
> **MIME-Type:** `application/caskeda-cask`  
> **File Extension:** `*.cask`  
> **Text Encoding:** `UTF-8 without BOM`, unless otherwise noted.  

## Introduction

The `cask`-format (first `k` is silent) is intended to be a human-readable definition of a virtual filesystem (henceforth called the `CVFS`), that serves as a **container for applications**, running inside the Caskeda app-browser.

When a `cask`-file without content is created, the CVFS will initially look like this:

```
CVFS
┆
├╴/.keda: The root script as a virtual file (empty).
╰╴/.cask: The container as a virtual file (empty).
```

There are *two things* an application container is made of, that make up the individual files within it: **Headers** and **Content**.

- **Headers** are a set of key-value pairs that contain a bunch of metadata regarding the file.
- **Content** is, well, the *content* of each file, encoded/linked as defined by the headers.
- **Boundaries**, separating the individual files.

> **Note:** All files that are put in the CVFS are, unless otherwise noted, *read-only* at runtime.

## Syntax & Semantics

### Headers

Every individual file in the container definition is preceded by a set of headers, that define how the content is encoded and to be decoded.

The header syntax follows these rules:

- Headers **ALWAYS** end with the next line-break (`\n`).
- Headers **MUST** be ignored, *if* they start with a `#` (eg: comments).
- Headers **MUST** be of the form `KEY: VALUE`, unless they're ignored.
  - The `KEY` is case-**in**sensitive, the `VALUE` is *not*.
  - Both `KEY` and `VALUE` individually have any whitespace trimmed.
- Headers **MUST** be prefixed by two dashes (`--`) if they're non-standard.
- Headers **MUST** have any whitespace trimmed off their start & end.
- Headers have their values joined (by line-breaks) when their keys are the same.
  - Thus, headers *may* occur multiple times.
- The headers section ends with a triple-dash delimited by line-breaks (`\n---\n`).

### Boundaries

The very first header of the container (and thus the entire file), should be the `Boundary` header, that specifies how the files in the container are to be kept apart.

```keda
Boundary: "-----EMBED-BOUNDARY-----"
```

If the header is not set, the string `-----EMBED-BOUNDARY-----` will be used as the default boundary.

After any headers-section ends (again, by a triple-dash delimited with line-breaks: `\n---\n`), the content-section *begins* and keeps going *until* the boundary occurs, at which point another file (with its own headers and contents) begins.

### Content

The content of a file follows immediately after the headers end (eg: after the `\n---\n`) and is to be treated as a *binary blob*.

To decode this blob into meaningful bytes, the following applies, as defined per header...

- `Content-Path`: An absolute path that defines where, in the VFS, the file will be placed.
- `Content-Type`: Defines the MIME-type of the file. If undefined, the file extension from `Content-Path` will be used to automatically select one.
- `Content-Radix`: How the bytes of the content were written; `base2`, `base16`, `base32`, `base62`, `base64`, `base85` and `base256` (the default). All but `base256` ignore any whitespace.
- `Content-Compression`: The decompressor to use once the bytes are read; `deflate`, `zstd` or `brotli`.
- `Content-Encoding`: The text encoding to use when the file is to be read as text; `utf-8` is the default.
- `Content-Identity`: A hashing algorithm name (eg: `sha2`), followed by a hash (eg: `sha2 abc123`), that the file *must* be tested against before it is made readable trough the CVFS. If it is streamed, the identity check will happen once the stream *ends* instead.

Resulting in the following pipeline:

```
RADIX-DECODE -> DECOMPRESS -> TEXT-DECODE -> (user code)
```

Failing to properly define the `Content`-headers will yield in the file not being read and decoded correctly, which may result in a garbled mess. There is *no* automatic pipeline detection.

#### File Includes

Instead of embedding a file directly in the container file, it is also possible to let the runtime load the content for the file from an external source.

This is done with the `Content-Link`-header and the `---no-content` indicator, which replaces the normal end-of-header separator (eg: `\n---no-content\n` instead of `\n---\n`).

```keda
Boundary: "-----EMBED-BOUNDARY-----"
---

-----EMBED-BOUNDARY-----
Content-Link: https://example.com/robots.txt
---no-content
```

It is also possible to define *multiple* URI's to serve as fall-backs, for when the file cannot be found trough the first URI. This is done by separating the URI's by semicolons:

```
Content-Link: https://example.com/robots.txt; cwd/robots.txt
```

#### Directory Includes

Of course, writing the headers for what is possibly hundreds, if not thousands, of files is ridiculous and inhumane. As such, it is also possible to include entire directories trough the `Content-Mount` header...

```keda
Boundary: "-----EMBED-BOUNDARY-----"
---

-----EMBED-BOUNDARY-----
Content-Path: /example.com/
Content-Mount: https://example.com/
---no-content
```

> **Note:** The slash at the end of the content-path is REQUIRED, for the file to be seen as directory.
> 
> Also, all headers except for `Content-Path` and the given `Content-Mount` are ignored.

The result of the above example, is that `https://example.com/` becomes a virtual directory in the CVFS:

```
CVFS
┆ --snip--
╰┬╴/example.com/ -> https://example.com/
 ╰ ...
```

When any file is accessed in such a directory, it will be loaded trough the given URI-scheme. Know URI-schemes include:

- `http`: Hyper-Text Transport-Protocol.
- `https`: Secure Hyper-Text Transport-Protocol.
- `cwd`: The current working directory/archive; does not support relative paths.
- `file`: The local filesystem; disabled if the `cask` comes from an external source.

It is also possible to define *multiple* URI's to serve as fall-backs, for when a file cannot be found trough the first URI. This is done by separating the URI's by semicolons:

```
Content-Mount: https://example.com/; cwd
```

#### Caching

> While caching is only intended for included content, embedded content MAY also be cached, but it is not necessary or enforced.

When the `Caching`-header is defined, the file/directory (and all it's contents) in question may be locally cached, either in memory or persistently, depending on the flags.

The flags are all packed into the `VALUE` of the header, separated by commas.

Merely by being defined, any affected content will be cached in memory.

Flags:
- `priority=P`: Low priority content *must* be evicted from the cache first, before higher priorities.
- `min=DURATION`: A integer duration, in seconds, for how long the content should try to stay in cache.
- `max=DURATION`: A integer duration, in seconds, for the maximum time the content may be cached.
- `persist`: If in any way possible, the content *SHOULD* be persisted in non-volatile storage.

### Example

```keda
#! /usr/sbin/caskeda
Boundary: "-----EMBED-BOUNDARY-----"
Version: 0.0.0
---

// Root Script goes here!

-----EMBED-BOUNDARY-----
# Content-Type will be detected as image/gif.
Content-Path: /tiny.gif
Content-Radix: base64
---
R0lGODlhAQABAIABAP///wAAACwAAAAAAQABAAACAkQBADs=

-----EMBED-BOUNDARY-----
# Content-Type will be detected as text/markdown.
Content-Path: /readme.md
---

**Hello**, *world*!

-----EMBED-BOUNDARY-----
Content-Path: /www/
Content-Mount: https://caskeda.com/
---no-content
```

Resulting in the following CVFS:

```
CVFS
┆
├╴/www/ -> https://caskeda.com/
├╴/tiny.gif
├╴/readme.md
├╴/.keda
╰╴/.cask
```

## File Operations

The CVFS must expose the following operations, in a asynchronous manner:

- `headers(path) -> Headers`: Return the headers of the given file/directory.
- `size_of(path) -> Int`: Return the amount of bytes the file occupies, or `null`/`None`.

- `read_raw(path) -> Bytes`: Read the given file and return it entirely, *without* text-decoding.
- `read_str(path) -> Chars`: Read the given file and return it entirely, *with* text-decoding.

- `stream_raw(path) -> Stream<Byte>`: Stream the given file, *without* text-decoding.
- `stream_str(path) -> Stream<Char>`: Stream the given file, *with* text-decoding.

## Packaging

There are two ways to package a `cask`-file: Either by serving it over the internet, or by distributing it named `.cask` within a `zip`-file, with `zcask` as extension (instead of `zip`).

In either case, the `cwd` URI-scheme serves as an abstraction over either for loading files.
