---
title: Using SDL2 to mount an archive from the Android APK assets with PhysicsFS
slug: 
date: 2023-04-30T14:12:14-04:00
description: 
draft: false
tags:
    - C++
    - android
    - physfs
    - SDL2
#hideSummary: true
#ShowWordCount: true

---

## Introduction

The C++ game engine that I've been working on stores game data in a custom binary archive that I mount using [PhysicsFS](https://icculus.org/physfs/) in order to access individual files.

On the desktop version of the engine the archive lives in the same folder as the executable, and it can be easily mounted by using `PHYSFS_getBaseDir` to find its path. The Android port of the engine, on the other hand, stores the data archive in the APK assets, which cannot be accessed directly by PhysicsFS.

There are multiple ways to work around this problem:
1. Copy the archive to external storage when the game is launched for the first time and mount it from there. Alternatively, don't ship the archive with the game and instead download it from a server at the first launch.
2. Read the whole archive to memory and then use `PHYSFS_mountMemory` to mount it.
3. Write a custom `PHYSFS_Io` interface to access android assets, and load the archive using `PHYSFS_mountIo`.

I decided to go with the third option since the first approach required coding too much new behaviour, and I didn't like the idea of keeping the archive loaded in memory for the whole game.

## SDL2 to the rescue

I have never done any Android development before porting my engine, so I wanted to avoid using the Android Asset Manager API and having to interact with Java. Luckily, my engine uses [SDL2](https://www.libsdl.org/), which includes an I/O interface (`SDL_RWops`) that can read Android assets, so I just wrapped this structure in a custom `PHYSFS_Io`. 

## The code
### Header
```c++
#include <string>

//forward declaration
struct PHYSFS_Io;

PHYSFS_Io* SDL_RW_Io(const std::string& filename);
```
The only non-static function that we need is `SDL_RW_Io`, which takes as input the filename of the archive (relative to the assets folder), creates the `SDL_RWops` structure using `SDL_RWFromFile`, and wrapts it in a `PHYSFS_Io`.

### Source
```c++
#include "sdl_rw.h" // header from before
#include "physfs.h"
#include "SDL_rwops.h"

//struct to hold the SDL_RWops pointer and other file information
struct sdl_rw_file_info {
    SDL_RWops* RW;
    std::string name;
    Sint64 cursor_pos;
    Sint64 size;
};

//forward declarations
static PHYSFS_sint64 io_read(PHYSFS_Io* io, void* buffer, PHYSFS_uint64 len);
static int io_seek(PHYSFS_Io* io, PHYSFS_uint64 offset);
static PHYSFS_sint64 io_tell(PHYSFS_Io* io);
static PHYSFS_sint64 io_tell(PHYSFS_Io* io);
static PHYSFS_Io* io_duplicate(PHYSFS_Io* io);
static void io_destroy(PHYSFS_Io* io);
```
The forward declarations are for the functions to assign to the fields of the `PHYSFS_Io` structure.

The function to create the `PHYSFS_Io` is pretty straightfoward: we use `SDL_RWFromFile` to create the `SDL_RWops`, populate the `sdl_rw_file_info` (which will be the `opaque` field of the `PHYSFS_Io`), and create the `PHYSFS_Io`.

If anything goes wrong when opening or reading the size of the archive any allocated memory is freed and the function returns a null pointer.
```c++
PHYSFS_Io* SDL_RW_Io(const std::string& filename) {
    
    SDL_RWops* RW = SDL_RWFromFile(filename.c_str(), "rb");
    if (!RW) { return nullptr; }

    auto finfo = new sdl_rw_file_info;
    finfo->RW = RW;
    finfo->size = SDL_RWsize(RW);
    if (finfo->size < 0) {
        SDL_RWclose(RW);
        delete finfo;
        return nullptr;
    }
    finfo->cursor_pos = 0;
    finfo->name = filename;

    auto retval = new PHYSFS_Io;

    retval->version = 0;
    retval->opaque = (void*)finfo;
    retval->read = io_read;
    retval->write = nullptr; // read-only
    retval->seek = io_seek;
    retval->tell = io_tell;
    retval->length = io_length;
    retval->duplicate = io_duplicate;
    retval->flush = nullptr; // read-only
    retval->destroy = io_destroy;

    return retval;
};
```

The `io_read` function uses `SDL_RWread` to read a certain number of bytes from the archive. The only thing to note is that `SDL_RWread` returns `0` whether there's an error or it's the end of the file, so we check `SDL_GetError` to distinguish the two cases and return `-1` in case of error.
```c++
static PHYSFS_sint64 io_read(PHYSFS_Io* io, void* buffer, PHYSFS_uint64 len) {
    auto finfo = (sdl_rw_file_info*)io->opaque;
    const PHYSFS_uint64 bytes_left = (finfo->size - finfo->cursor_pos);
    PHYSFS_sint64 bytes_read;

    if (bytes_left < len) { len = bytes_left; }

    bytes_read = SDL_RWread(finfo->RW, buffer, 1, len);
    
    if (bytes_read > 0) {
        finfo->cursor_pos += bytes_read;
    }
    else {
        SDL_ClearError();
        bytes_read = SDL_GetError()[0] != '\0' ? -1 : 0;
    }

    return bytes_read;
}
```
The next three functions are quite simple. `io_seek` uses `SDL_RWseek`, while the other two use the data stored in the `sdl_rw_file_info`.
```c++
static int io_seek(PHYSFS_Io* io, PHYSFS_uint64 offset) {
    auto finfo = (sdl_rw_file_info*)io->opaque;

    if (offset >= finfo->size) {
        PHYSFS_setErrorCode(PHYSFS_ERR_PAST_EOF);
        return 0;
    }

    finfo->cursor_pos = SDL_RWseek(finfo->RW, offset, RW_SEEK_SET);
    return finfo->cursor_pos >= 0;
}

static PHYSFS_sint64 io_tell(PHYSFS_Io* io) {
    auto finfo = (sdl_rw_file_info*)io->opaque;
    return finfo->cursor_pos;
}

static PHYSFS_sint64 io_length(PHYSFS_Io* io) {
    auto finfo = (sdl_rw_file_info*)io->opaque;
    return finfo->size;
}
```
For `io_duplicate` we have to create a full independent copy of the `PHYSFS_Io`, so we create a new `SDL_RWops` by using `SDL_RW_Io` again with the filename that we stored.
```c++
static PHYSFS_Io* io_duplicate(PHYSFS_Io* io) {
    auto origfinfo = (sdl_rw_file_info*)io->opaque;
    auto retval = SDL_RW_Io(origfinfo->name);

    if (!retval) { return nullptr; }

    auto finfo = (sdl_rw_file_info*)retval->opaque;
    finfo->cursor_pos = origfinfo->cursor_pos;
    // seek the SDL_RWops to the appropriate position
    if (SDL_RWseek(finfo->RW, finfo->cursor_pos, RW_SEEK_SET) < 0) {
        retval->destroy(retval); // free allocated memory
        return nullptr;
    }

    return retval;
}
```
Finally, `io_destroy` frees all allocated memory.
```c++
static void io_destroy(PHYSFS_Io* io) {
    auto finfo = (sdl_rw_file_info*)io->opaque;
    SDL_RWclose(finfo->RW);
    delete finfo;
    delete io;
}
```


### Usage
```c++
auto io = SDL_RW_Io("data.pack");
if (io) {
    if (!PHYSFS_mountIo(io, "data.pack", "/", false)) {
        // PHYSFS_mountIo does not destroy the PHYSFS_Io if it fails
        // so we do it manually
        io->destroy(io);
    }
}
```
Note that the `PHYSFS_Io` needs to be kept until the archive is unmounted. Calling `PHYSFS_unmount` will automatically call `io->destroy(io)` and free the associated memory.
