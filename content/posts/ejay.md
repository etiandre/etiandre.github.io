+++
title = 'Extracting audio samples from eJay software'
date = 2024-10-08T22:51:56+02:00
draft = true
+++

## Findings

from wikipedia:

> eJay is a series of musical arrangement software, and video games, primarily for Microsoft Windows. The first edition, Dance eJay, was released in 1997. It supports eight tracks of audio and, as with its successors, permits the arrangement of sound bites by a drag-and-drop interface. Since the original Dance eJay, there have been many releases catering to different music genres and users, including techno and hip-hop, as well as a PlayStation 2 edition called eJay Clubworld.

The windows version seem all to be Visual Basic applications, with a few dedicated routines in DLLs. Each version seem to bolt new features on the ancient codebase.

The last updates to come out were _Dance 6 reloaded_ and _Hip Hop 5 reloaded_.

The later versions of the software are still [available to buy online](https://www.ejayshop.com/) from the swiss-based company _Yelsi AG_.

The developer has had many names throughout history:

- first, PXD Musicsoft Inc.

## Resources

### Decompression routines

<https://github.com/paator/eJayDecompressor> calls the ejay dlls to decompress .PXD files. The audio decompression routine is called `ADecompress` and its symbol is exported in `pxd32d5_d4.dll`.

The actual decompression routine looks like some LZ variant (?).

The PXD files can either be:

- uncompressed (just the extension is changed)
- a single compressed file
- a flat collection of files
