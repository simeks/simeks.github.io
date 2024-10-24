---
title: "Zig comptime and C"
date: 2024-10-24
excerpt_separator: <!--more-->
---

I've been using [Zig](https://ziglang.org) a lot for the last 6 months and I have to say I really enjoy it. Two of Zigs strengths are integration with C and comptime. Thought I would share a small showcase of this.

<!--more-->

I have these two components of a system that I'm working on in parallel. The core is written in Zig but I have this C library that I want to talk to. Thanks to how well Zig deals with C it has been a blast. I can even configure the whole Zig build system to build the C project for me and you barely need any wrapping to call the C functions from Zig:

```zig
pub const Module = anyopaque;

// Our C functions
extern fn moduleInit() *Module;
extern fn moduleDeinit(*Module);
extern fn moduleDoStuff(*Module) void;

test "talk to C" {
    const module = moduleInit();
    defer moduleDeinit(module);

    moduleDoStuff(module);
}
```
and if you like methods you could even do:
```zig
pub const Module = opaque {
    const init = moduleInit;
    extern fn moduleInit() *Module;

    const deinit = moduleDeinit;
    extern fn moduleDeinit(*Module);

    const doStuff = moduleDoStuff;
    extern fn moduleDoStuff(*Module) void;
};

// Our C functions

test "talk to C" {
    const module = Module.init();
    defer module.deinit(module);

    module.doStuff();
}
```
If you already have a header file for your C library you wouldn't even need to declare all the externs yourself, you can just:
```zig
const c = @cImport({
    @cInclude("module.h");
});

pub const Module = opaque {
    const init = c.moduleInit;
    const deinit = c.moduleDeinit;
    const doStuff = c.moduleDoStuff;
};
```
Note that `@cImport` will be replaced in later versions of Zig to be replaced with a build step instead: https://github.com/ziglang/zig/issues/20630.

There is a catch though. After you have been getting used to the quality of life features of Zig it's a bit constraining to go back to the C ABI. For instance there is no way to pass Zig specific constructs over this boundary, such as errors or slices, which makes total sense. Zig provides a way for you to declare a struct that is compatible with the C ABI (`extern struct`). These structs can be used over the ABI and Zig ensures you do not use anything that is not allowed within them.

So my aim was to provide some better ergonomics to them. Consider the following regular struct in Zig:
```zig
const Desc = struct {
    // Array of Textures, struct holds the memory for 4 Textures
    // but the used size of the array might be smaller.
    textures: std.BoundedArray(Texture, 4),

    // Slice of data, which is a fat pointer (ptr + len)
    data: []const u8,
};

test "zig struct init" {
    const data = try readData();
    const desc: Desc = .{
        .textures = try .fromSlice(&.{
            .{ .width = 256, .height = 256 },
            .{ .width = 256, .height = 256 },
        }),
        .data = data,
    };
}

```
anyone that have been working with a low-level graphics API will surely recognize this type of struct. This same struct would look something like this in C:
```c
struct {
    Texture textures[4];
    size_t textures_len;

    const char* data;
    size_t data_len;
};
```
or in Zigs `extern struct`:
```zig
const ExternDesc = extern struct {
    textures: [4]Texture,
    textures_len: usize,

    data: [*]const u8,
    data_len: usize,
};

test "zig extern struct init" {
    const data = try readData();
    const desc: ExternDesc = .{
        .textures = .{
            .{ .width = 256, .height = 256 },
            .{ .width = 256, .height = 256 },
            // All 4 must be initialized
            undefined,
            undefined,
        },
        .textures_len = 2,

        .data = data.ptr,
        .data_len = data.len,
    };
}
```
And I make the mistake of forgetting to update `textures_len` all the time and maybe I put in the wrong value for `data_len` at times as well, it has definitely happen.

So to my actual showcase! I managed to solve these two cases fairly easily by using comptime. Just like `std.BoundedArray` is just a comptime function that creates a new type, I wrote my own two new comptime functions: `ExternArray` and `ExternPointer` which provides a helper init function but also ensures the resulting type can pass over the C ABI.

```zig
/// Extern struct type for array[capacity] + len
/// Maps to this in C:
/// struct {
///     T extern_array[capacity];
///     size_t num_extern_array;
/// }
pub fn ExternArray(T: type, comptime capacity: usize) type {
    const alignment = @alignOf(T);
    return extern struct {
        pub const Element = T;
        pub const Capacity = capacity;

        const Self = @This();

        data: [capacity]T align(alignment) = undefined,
        len: usize = 0,

        pub fn init(m: []const T) error{Overflow}!Self {
            if (m.len > capacity) return error.Overflow;
            var self: Self = .{
                .len = m.len,
            };
            @memcpy(self.data[0..m.len], m);
            return self;
        }
    };
}

test "ExternArray" {
    const Desc = extern struct {
        textures: ExternArray(Texture, 4),
    };
    const desc: Desc = .{
        // Decl literals are amazing!
        .textures = try .init(&{
            .{ .width = 256, .height = 256 },
            .{ .width = 256, .height = 256 },
        }),
    };
}
```
Above is the type for wrapping fixed size arrays. It doesn't actually do more than just adding an initializer function for moving my slice into the array (with included bounds checking), but it's so easy to add to my structs.

The other example handles the pointer + length case:
```zig
/// Extern fat pointer for passing ptr + len through C ABI
/// Maps to this in C:
/// typedef struct {
///     T* extern_array;
///     size_t num_extern_array;
/// }
pub fn ExternPointer(T: type) type {
    return extern struct {
        pub const Element = T;

        const Self = @This();

        ptr: ?[*]const T = null,
        len: usize = 0,

        pub fn init(s: []const T) Self {
            return .{
                .ptr = s.ptr,
                .len = s.len,
            };
        }
    };
}

test "ExternPointer" {
    const Desc = extern struct {
        data: ExternPointer(u8, 4),
    };

    const data = try readData();
    const desc: Desc = .{
        .textures = .init(data),
    };
}
```
Same here, it's actually a fairly simple thing, but will save me from a lot of mistakes in the future. There are some downside. Given that these are basically just fancy initializers, actually using them after initializing will require you do keep track of both the data and the len. However, I wrote these with the intent of being mostly static structs so I don't mind, and on the C side I will have to deal with that anyways.

Similar to templates in C++ (or any tool for that matter) it's easy to overdo things as well. I got this idea to do something similar with [tagged unions](https://ziglang.org/documentation/master/#Tagged-union) which I also use a lot. Consider this (non-external) example:
```zig 
const Resource = union(enum) {
    texture: Texture,
    buffer: Buffer,
};

const Desc = struct {
    resource: Resource
};
```
Translating this to C gets me something like
```c
struct {
    ResourceType resource_type;
    union {
        Texture texture;
        Buffer buffer;
    };
};
```
so of course I decided to go the same route here, but it got slightly more complicated:
```zig
/// Constructs a C ABI friendly type based on the given tagged union.
///
/// Mapped as
/// struct {
///     Enum tag;
///     union {
///         Value0 value0;
///         Value1 value1;
///         Value2 value2;
///     };
/// }
pub fn ExternTaggedUnion(TaggedUnion: type) type {
    // First we do some checks that TaggedUnion actually is what we expect
    const type_info = @typeInfo(TaggedUnion);
    if (type_info != .@"union" or type_info.@"union".tag_type == null) {
        @compileError("Expected tagged union");
    }
    // Tag type should always be enum (if it exists)
    const tag_type_info = @typeInfo(type_info.@"union".tag_type.?).@"enum";

    // Need u32 for extern

    // Now comes the comptime magic!
    // First we create the two subtypes, a tagged union is essentially a combined
    // union and an enum. The extern struct won't accept the tagged union, but it
    // accepts both unions and enums, so we separate them into two types.
    const ExternTag = @Type(.{
        .@"enum" = .{
            // For this to be extern compatibile we need a supported tag_type,
            // the original enum in TaggedUnion is probably far less number of bits.
            .tag_type = u32,
            // we can just re-use the fields of the TaggedUnion
            .fields = tag_type_info.fields,
            .decls = tag_type_info.decls,
            .is_exhaustive = tag_type_info.is_exhaustive,
        },
    });

    // Union here is just a union type marked as extern and without tag_type
    // otherwise we take all the fields of the original union
    const ExternUnion = @Type(.{
        .@"union" = .{
            .layout = .@"extern",
            .tag_type = null,
            .fields = type_info.@"union".fields,
            .decls = type_info.@"union".decls,
        },
    });

    // The resulting struct is then layed out as in the C example above.
    return extern struct {
        pub const SourceType = TaggedUnion;

        tag: ExternTag,
        value: ExternUnion,

        pub fn init(value: TaggedUnion) @This() {
            // We provide an initializer that allows us to pass in a tagged union, which
            // we then translate into our own representation (with enum and union separated)
            const active_tag = std.meta.activeTag(value);
            // We can't directly map between the two so this searches for the currently
            // active tag by value, and based on that it constructs the value.
            inline for (std.meta.fields(ExternTag)) |field| {
                if (field.value == @intFromEnum(active_tag)) {
                    return .{
                        .tag = @enumFromInt(field.value),
                        .value = @unionInit(ExternUnion, field.name, @field(value, field.name)),
                    };
                }
            }
            unreachable;
        }
    };
}

test "ExternTaggedUnion" {
    const Resource = union(enum) {
        texture: Texture,
        buffer: Buffer,
    };

    const Desc = extern struct {
        resource: ExternTaggedUnion(Resource)
    };

    const resource: Resource = .{
        .buffer = .{ .size = 256 },
    };

    const desc: Desc = .{
        .resource = .init(resource),
    };
}
```
The above works, but I'm not sure the need for it is big enough that I might actually deploy it. One of the drawback of all of these is that it hides details that might be important. For instance, under certain circumstances you might want to layout the struct manually do avoid wasting bits.

Tying this to the start of the post, using these extern structs together with our extern C functions is as simple as:
```zig
const Desc = extern struct {
    textures: ExternArray(Texture, 4),
};

pub const Module = opaque {
    const init = moduleInit;
    extern fn moduleInit() *Module;

    const deinit = moduleDeinit;
    extern fn moduleDeinit(*Module);

    const doStuff = moduleDoStuff;
    extern fn moduleDoStuff(*Module, *Desc) void;
};

test {
    const module = Module.init();
    defer module.deinit();

    const desc: Desc = .{
        // Decl literals are amazing!
        .textures = try .init(&{
            .{ .width = 256, .height = 256 },
            .{ .width = 256, .height = 256 },
        }),
    };

    module.doStuff(&desc);
}
```


On top of this I wrote a generator in Zig that is able to parse all these structs, unions, and enums to generate a C header file that is compatible with all these constructs so that I could actually use them. This was surprisingly easy, again thanks to comptime, but this will have to be the subject for another post.

Necessary disclaimer: All code above is written for Zig 0.14.0-dev.1951+857383689 and might not work on older (or newer) versions. For instance decl literals were introduced recently and is only available in the dev builds (>=0.14).


