+++
title = "no more bit fiddling - introducing bilge"
description = "Use a better alternative to bit fiddling in low-level Rust: bilge. It builds upon the idea of bitfields to declare easy-to-use memory-mapped registers."
updated = 2023-05-16
+++

## TLDR

_`bilge`_ is a new bitfield crate following in the footsteps of `modular-bitfield`, with improved ergonomics and type safety, while still being as performant as handmade bit fiddling.

```rust
#[bitsize(14)]
#[derive(FromBits)]
struct Register {
    header: u4,
    body: u7,
    footer: Footer,
}
```

## How bit fiddling works, and why it sucks

When working on low-level code, you often need to set or get single bits.
Let's use a simple example to show how this works: Mouse Packets. These allow you to use your PC mouse in the first place.

Specifically, this is how PS/2 mouse packets look like:

| bits   | Description           |
|--------|-----------------------|
| 0      | left button           |
| 1      | right button          |
| 2      | middle button         |
| 3..8   | other important bits  |
| 8..16  | x-axis movement       |
| 16..24 | y-axis movement       |

But in rust, we can (currently) only access 8 bits = 1 byte at once, using `u8`.
Getting at smaller values can be done by so-called "bit fiddling", using binary operators to shift by some offset and then masking some bits.

### basic bit fiddling

For example, let's see if the mouse has been right-clicked.
Imagine the mouse sent us this value:

```rust
let mouse_packet = 0b00011000_00000011_00001_0_1_0;
```

For reading purposes, I delimit the different fields with `_`. As you can see, bit 0 is at the end of the value. This is called little-endian (bit-) ordering.

Our right mouse button is therby in the second-to-last bit (in this case, set to 1).
So, if we want to have a value containing only the information that a right-click happened, we need to remove all other bits, by using a mask:

```rust
let mouse_packet = 0b00011000_00000011_00001_0_1_0;
let mask =         0b00000000_00000000_00000_0_1_0;
let right_clicked = mouse_packet & mask;
// right_clicked is now 0b00000000_00000000_00000_0_1_0, so, 0b10
```

And then we also have to shift the value into place:

```rust
// bit offset of right_clicked in mouse_packet, counting from the left
let bit_offset = 1;
// right_clicked is 0b10
let right_clicked = right_clicked >> bit_offset;
// right_clicked is now 1
```

If we imagine some user-code reacting to the right-click, it would be nice to convert this to a `bool`:

```rust
let right_clicked = right_clicked == 1;
if right_clicked {
    show_context_menu();
}
```

Now, we don't want this code floating around in `fn main()`, so let's put it in a struct.

### encapsulated bit fiddling

We start by adding a wrapper struct:

```rust
struct MousePacket(u32);
```

And then put all of our access code inside:

```rust
impl MousePacket {
    // let's add this for good measure
    const fn left_clicked(&self) -> bool {
        self.0 & 1 == 1
    }
    const fn right_clicked(&self) -> bool {
        (self.0 >> 1) & 1 == 1
    }
    /* add the other bit-fields here */
}
```

Which means our high-level code now looks like this:

```rust
// again, imagine the mouse sent us this
let mouse = MousePacket(0b00011000_00000011_00001_0_1_0);
if mouse.right_clicked() {
    show_context_menu();
}
```

Now imagine you had a mouse with LEDs inside its buttons, which can be lit up by sending back a mouse packet with `right_clicked` set to true.

I know, grandiose example, but I just wanted to show how to set bits as well:

```rust
impl MousePacket {
    const fn set_right_clicked(&mut self, value: bool) -> bool {
        // move right_clicked where it belongs
        let right_clicked = (value as u32) << 1;
        // produce a mask which only ignores right_clicked
        // will look like this: 0b11111111_11111111_11111_1_0_1
        let others_mask = !(0b1 << 1);
        // keep all fields besides right_clicked
        let other_values = self.0 & others_mask;
        // merge old fields and right_clicked
        self.0 = other_values | right_clicked;
    }
}
```

Since this follows some basic rules, it's a good idea to separate concerns and put all of this bit fiddling into an abstraction - bitfields.

## How bitfields work, and why they "suck"

Besides the horrors of C bitfields (which I have only heard about), bitfields don't suck. I only dislike that they're not provided by rust itself [^1]. I'll try to work on that.

Again, let's take our example and rewrite it into a bitfield, with some imaginary syntax:

```rust
struct MousePacket {
    left_clicked: bool,
    right_clicked: bool,
    middle_clicked: bool,
    other_important_bits: u5,
    x_movement: u8,
    y_movement: u8,
}
```

Since rust doesn't support bitfields, the only way to have this work is by using macros, so we would at least need to add something like `#[bitfield]` to our struct. Behind the scenes, this can generate very similar code to the one we implemented by hand.

With that in place, we don't need to worry about bits, we would at most have to think about the size and order of our fields when interacting with hardware.

This avoids bit fiddling errors (well, if the bitfield implementation doesn't contain bugs itself) and means you don't have to reimplement different kinds of _fallible nested-struct enum-tuple array field access_, which might not be so fun.

It's also far easier to use for beginners, although I would argue even seniors, since the real question is: "Why do we need to learn and debug bit fiddling if we can get most of it done by using structs?"

So, when I started working on [Theseus' PS/2 driver code](https://github.com/theseus-os/Theseus/commit/a76ca0f09e3d4fcf4715aa225ef1213deebd3a53#diff-c51fc9762ba849c357b487bcc25b5eeadb0f3f1e358162f81076833391f996b5), I found the most prominent bitfield crate to be `modular-bitfield`.

## How modular-bitfield works, and what we want to solve

With [`modular-bitfield`](https://github.com/robbepop/modular-bitfield), we would define our example code like this:

```rust
#[bitfield(bits = 24)]
struct MousePacket {
    left_clicked: bool,
    right_clicked: bool,
    middle_clicked: bool,
    other_important_bits: B5,
    x_movement: B8,
    y_movement: B8,
}
```

The _"frontend"_ looks very readable, since we just use normal struct syntax and an attribute. Bit-sized types are specified using `B1`-`B128` and we can just use nested structs and enums.

But there are some problems with the crate:
- it is unmaintained and has a quirky structure
- constructors use the builder pattern
- `from_bytes` can easily take invalid arguments
- it is a big god-macro

### how to structure procedural macros

With "quirky structure" I specifically mean the proc macro code itself. Yes, macro code is always a bit hard to read, but naming, documentation and file structure can guide you.

A nice read on this is [Structuring, testing and debugging procedural macro crates](https://ferrous-systems.com/blog/testing-proc-macros/#the-pipeline), specifically the part about "the pipeline".
Just using basic naming, like `analyze` for getting basic information from an annotated item and `generate` for anything generating a `TokenStream`, works wonders.

As an example, I hope [this flow](https://github.com/hecatia-elegua/bilge/blob/2008c9c00a0a7c7d796eec3c1a7d5b72fa7bec90/bilge-impl/src/from_bits.rs#L7) is readable enough. I'm only a bit unsatisfied about the low amount of docs.

The main proc macro `lib.rs` just contains whatever documentation, `TokenStream`-conversion and so on and immediately calls into a different module.
For easy search inside the filesystem, the module (e.g. `from_bits`) has the same name as the proc macro (e.g. `FromBits`) and contains a function (e.g. `from_bits`) of the same name as well. This top-level function then contains the very general routing of parse -> analyze -> expand.

<aside>

Another piece of advice I want to give: Just use functions. <br>
Maybe it's just me, but using struct methods for all flow in `modular-bitfield` made it more confusing.

</aside>

### why the builder pattern should be used carefully

Let's take a look at why the builder pattern isn't that great. To stay with our example of PS/2 code, imagine we have:

```rust
#[bitfield(bits = 3)]
pub struct LEDState {
    pub scroll_lock: bool,
    pub num_lock: bool,
    pub caps_lock: bool,
}
```

This needs to be sent to the keyboard hardware, meaning we need to instantiate it with the builder pattern:

```rust
LEDState::new()
    .with_scroll_lock(is_scroll_lock)
    .with_num_lock(is_num_lock)
    .with_caps_lock(is_caps_lock),
```

It looks acceptable in this example, but imagine we had 12 fields. Also, if we accidentally leave out one of these `with`-calls, it would leave these fields zeroed out. In rust, we can't declare variables without initializing them, so why should this be possible, either?

### type-centric design - parse, don't validate

To talk a bit more about `from_bytes`, here we have an unfilled bitfield:

```rust
#[bitfield(bits = 10)]
pub struct PackedData {
    body: B8,
    #[bits = 2]
    status: Status,
}
#[derive(BitfieldSpecifier)]
#[bits = 2]
pub enum Status {
    Red = 0b00, Green = 0b01, Yellow = 0b10,
}
```

First of all, by now you can probably guess how an enum in a bitfield works. Similar to our `bool` example, the enum value gets parsed. For enums, this is based on the declared value of the variant.

In this case however, not every bit-combination is specified inside the enum - `0b11` is missing.
There are some ways to handle this, but I will argue `modular-bitfield`'s way is wrong:

```rust
let mut data = PackedData::from_bytes([0b0000_0000, 0b1100_0000]);
//           The 2 status field bits are invalid -----^^
assert_eq!(data.status_or_err(), Err(
    InvalidBitPattern { invalid_bytes: 0b11 }
));
data.set_status(Status::Green);
assert_eq!(data.status_or_err(), Ok(Status::Green));
```

We can also call `status()`, which does this:

```rust
self.status_or_err().expect("value contains invalid bit pattern for field PackedData.status")
```

So, `from_bytes` can easily take invalid arguments, which turns the type "inside-out":

`u16` -> `PackedData::from_bytes` -> `PackedData::status_or_err()?`

This needs to check for `Err` on every single access, adds duplicate getters and setters with postfix `_or_err` and reinvents `From<u16>`/`TryFrom<u16>` as a kind of hybrid.

The usual, type-centric flow is:

`u16` -> `PackedData::try_from(u16)?` -> `PackedData::status()`

This checks once on initialization of the type and needs to check nothing on access.
Some more general info on this can be found in [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).

### how to split up big procedural macros

Another interesting problem is the tendency for crates like this to evolve into a kind of "god-macro". They tend to handle all kinds of special usecases and even implement derives (e.g. `impl Debug`) inside the one macro (e.g. `#[bitfield]`).
That's powerful, but not scalable.

It would be much nicer if others could access the bit-information in their own attribute or derive macros.
This can be solved by providing a kind of scope for any bit-based derive macros, as we will see later.

### implementation peculiarities

If we talk about the bitfield implementation itself, it's interesting that their underlying type is a byte array. It doesn't matter all that much [on any benchmark](https://github.com/hecatia-elegua/bilge#benchmarks-performance-asm-line-count), since the computations are done at compile time. The build-time itself also doesn't suffer.
It only matters for ergonomics in some cases.

I assume it's an array because `modular-bitfield` was based on [this dtolnay workshop project](https://github.com/dtolnay/proc-macro-workshop/blob/a2a05d0aafcdf9fe13f1ca0bbe5f74418401b19f/README.md?plain=1#L223). This could be useful for bitfields larger than u128, but if our bitfields get larger than u128, we can split them into multiple bitfields of a primitive size (like u64) and put those in a parent struct which is not a bitfield.

Still, `modular-bitfield` is pretty good and worked nicely for getting PS/2 up and running.

## How bilge came to be

Since I started prototyping on USB, PCI and ARM driver implementations, I quickly realized there's some need to modify what `modular-bitfield` does. Due to the above mentioned problems and because of some more register-related requirements, I started looking for alternatives.

So I went through most bitfield crates on crates.io:
- Most bitfield libs just initialize with raw values, or something similar to `from_bytes` (see above)
- Many bitfield libs have a terrible frontend (i.e. try to look like C bitfields and so on)
- bitflags fit better into a struct with bools and enums as well, in my opinion

Then I found [danlehmann's `bitbybit`](https://crates.io/crates/bitbybit), which has a similar frontend to `modular-bitfield` and is actually maintained and simple.
They also use [`arbitrary bitwidth integers`](https://github.com/danlehmann/bitfield/issues/4), which I deeply wish to be integrated into rust.

After hacking around in `bitbybit` for a while, I noticed parts of it would need to work differently, so I began removing some features and later concluded it would be best to write my own while learning some more about proc macros.

I set out to build `bilge`, a crate to handle bit-sized types at least equal to or better than `modular-bitfield` and `bitbybit`.
Tell me where I can do better, I will try.

I wanted a design fitting rust:

- **safe**
    - types model as much of the functionality as possible
    - don't allow false/unsafe usage (unless specified)
- **fast**
    - like handwritten bit fiddling code
- **simple to complex**
    - obvious and readable basic frontend, like normal structs
    - only minimally and gradually introduce advanced concepts

## Building bilge

First, I noticed the byte-oriented `size_of!`, `offset_of!`, `repr` and [`num_enum::TryFromPrimitive`](https://github.com/illicitonion/num_enum#attempting-to-turn-a-primitive-into-an-enum-with-try_from) would kinda need to have a bit-oriented version.
Then I had the basic idea to just provide a bitsize on items, which could later be used by other proc macros.
So I scratched the usual "`#[bitfield]`" idea in favor of `#[bitsize(num)]` and actual derive macros.

### attribute macro as a scope

This means `#[bitsize]` basically acts as a scope for macros underneath it, providing whatever is necessary for them.
That way, `#[bitsize]` would take on all the basic validity checks and special bitfield handling, while `#[derive(FromBits)]` can do the actual parsing of raw values (e.g. registers).
It also opens the way for others to define their own derive macros based on this.

```rust
#[bitsize(6)]
#[derive(FromBits)]
struct ParentStruct {
    field1: ChildStruct,
    field2: ChildStruct,
    field3: ChildEnum,
}
```

Now the main question was how to pass the bitsize to the other macros, as macros are evaluated outside-in and then deleted.
Along the way I learned that you can't generate "inert" attributes, only [derive macro helper attributes](https://doc.rust-lang.org/reference/procedural-macros.html#derive-macro-helper-attributes).

So at that point, the way to make this work was by letting `#[bitsize(num)]` add `#[bitsize_internal(num)]` to the item. It stays on the item even after macro expansion:

```rust
#[bitsize_internal(6u8)]
struct ParentStruct {
    value: u6,
}
```

You may be confused why I didn't just parse the field value.
Of course not because I didn't notice, but because it will be useful to have for enums 🙂

### deriving `TryFrom` and `From` for bit-sized enums

I then started the `#[derive(TryFromBits)]` implementation for enums, to allow enums which don't specify all of their bits. This meant I had to parse the `#[bitsize_internal]` helper attribute and generate this:

```rust
impl ::core::convert::TryFrom<u2> for IncompleteEnum {
    type Error = u2;
    fn try_from(number: u2) -> ::core::result::Result<Self, Self::Error> {
        match number.value() {
            0 => Ok(Self::A),
            1 => Ok(Self::B),
            2 => Ok(Self::C),
            i => Err(u2::new(i)),
        }
    }
}
```

The next, easy step was duplicating the code for `#[derive(FromBits)]` and adjusting it to fit its requirements. That means it needs to validate that all bit-patterns are used (one enum variant is specified for every bit-pattern).

### deriving `From` for bit-sized structs

Then structs needed to be implemented, so I started to get some use out of my experiments with Daniel's `bitbybit` crate.
A getter or setter acts on one field and needs to know the field's bit-offset and mask. This is accomplished by specifying `BITS` and `MAX` for every bit-sized type.

Later on, this gets handled differently, but just to visualize it:

```rust
impl ParentStruct {
    fn field1(&self) -> ChildStruct {
        ChildStruct::new((self.value >> 0) & ChildStruct::MAX)
    }
    fn field2(&self) -> ChildStruct {
        ChildStruct::new((self.value >> (0 + ChildStruct::BITS)) & ChildStruct::MAX)
    }
}

impl ChildStruct {
    const BITS: usize = 2;
    const MAX: u2 = u2::MAX;
}
```

Next up, structs should accept nested enums deriving `TryFromBits`. The struct itself would then have to be `TryFromBits`, obviously, since not all of its bit-combinations are declared.
So, let's validate that `FromBits` can't be used if any sub-structs/enums use `TryFromBits`.

Since we're in macro land, this can't be checked by somehow querying the type system. Instead, after stumbling around for a bit, I decided to add a `const FILLED`, which is just a flag for telling whether or not the item is `From`-compatible.

For enums, `FILLED` just checks if all bit combinations are filled, and for structs it iterates all fields and then adds `::FILLED` to those and joins them using `&&`.
It leaves out all fields which are not a `uN`, since they're always filled (by abusing the fact that `uN` types start with a lowercase `u` 🙂).

Which ends up looking like this:

```rust
impl ParentStruct {
    const FILLED: bool = ChildStruct::FILLED && ChildEnum::FILLED;
}
```

And then, in `#[derive(FromBits)]`, we generate this:

```rust
const _: () = assert!(#type_name::FILLED, "implementing FromBits on bitfields with unfilled bits is forbidden");
```

Wait, where is `TryFromBits` on structs!?

### deriving `TryFrom` for bit-sized structs

Well, that's the most interesting part of this.
Until now, I basically did this for structs:

```rust
let is_ok = true;

if is_ok {
    Ok(Self { value })
} else {
    Err(value)
}
```

It was time to finally compute `is_ok`.
To do so, at least field offsets and types need to be sent across the macro boundary.

The choice of using an _inert attribute_ had me going astray for a bit:

```rust
#[bitsize_internal(6u8, 0, ChildStruct, 0+ChildStruct::BITS, ChildStruct, 0+ChildStruct::BITS+ChildStruct::BITS, ChildEnum)]
struct ParentStruct {
    value: u6,
}
```

This isn't necessary and I removed this later, but I still want to share this silly piece of code. Just skip the below indented part (`<aside>`) if uninterested.

<aside>

I just kinda threw field offsets and types into the internal attribute with some commas.
Maybe I shouldn't have done that. This is the parsing code:

```rust
#[derive(Debug)]
struct Attributes {
    size: LitInt,
    others: TokenStream,
}
impl Parse for Attributes {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        Ok(Attributes {
            size: input.parse()?,
            others: input.parse()?,
        })
    }
}
#[derive(Debug)]
struct Others {
    _comma_token: Token![,],
    field_offset_and_type: Punctuated<(Expr, Token![,], Ident), Token![,]>,
}
impl Parse for Others {
    fn parse(input: ParseStream) -> syn::Result<Self> {
        Ok(Others {
            _comma_token: input.parse()?,
            field_offset_and_type: input.parse_terminated(parse_expr_ident)?,
        })
    }
}
fn parse_expr_ident(input: ParseStream) -> syn::Result<(Expr, Token![,], Ident)> {
    Ok((input.parse()?, input.parse()?, input.parse()?))
}
let attributes = syn::parse2::<Attributes>(internal_bitsize_braced.stream()).expect("wrong");
let size = attributes.size;
let fields_info: Vec<(Expr, Ident)> = if !attributes.others.is_empty() {
    let others = syn::parse2::<Others>(attributes.others).expect("wrong");
    others.field_offset_and_type.iter().map(|field| (field.0.clone(), field.2.clone())).collect()
} else {
    vec![]
};
```

YES, that's some succinct parsing code! (Again, this is old code, don't worry.)

I couldn't have known that syn parsing is so intricate. If I had chosen another way to encode this, it would have been easier.
Forgive me for I have synned (Thanks [Nathan](https://github.com/NathanRoyer), I will keep that joke).

</aside>

And now that all the necessary field information is available, we just generate our bit-shifting and masking for each field:

```rust
let field_value = quote! {
    #field_type::new(
        (struct_value >> (#field_offset)) & #field_type::MAX
    )
};
let is_ok = quote! {
    // has try_from impl
    if !#field_type::FILLED { 
        #field_type::try_from(#field_value).is_ok()
    } else {
        true
    }
}
```

And put it into the `impl TryFrom`:

```rust
quote! {
    let is_ok = {#is_ok};

    if is_ok {
        Ok(Self { value })
    } else {
        Err(value)
    }
}
```

And that's it, right?
Let's run this.

<div style="text-decoration: underline 0.1em wavy #c7ab0e; text-decoration-skip-ink: none">

```
ParentStruct::try_from(u6::new(0b110000));
```

</div>

Huh?

```sh
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: 3'
```

Let's run [`cargo expand`](https://github.com/dtolnay/cargo-expand#cargo-expand):

```rust
/* ... */
let is_ok = if !ChildEnum::FILLED {
    ChildEnum::try_from(ChildEnum::new(
        (value >> (0)) & ChildEnum::MAX
    )).is_ok()
} else {
    true
}
/* ... */
```

Ah yes, `ChildEnum::try_from(ChildEnum)`.

What I actually want to put there is not `#field_type`, but `#field_arbitrary_int_type`, to produce:

```rust
ChildEnum::try_from(u2::new(value)).is_ok()
```

It would be pretty nice if one had access to the field's `arbitrary-int` type in general, so let's add a trait to all bit-sized items!

Let's add it like this for now, just to check if it works:

```rust
pub trait Bitsized {
    type ArbitraryInt;
}
```

And in macro code:

```rust
impl Bitsized for #name {
    type ArbitraryInt = #arb_int;
}
```

And in the `impl TryFrom`:

```rust
let field_arbitrary_int_type = quote!(<#field_type as Bitsized>::ArbitraryInt);
let field_value = quote! {
    #field_arbitrary_int_type::new(
        (struct_value >> (#field_offset)) & #field_type::MAX
    )
};
```

Now `ParentStruct::try_from` works!

### deriving `Debug` for bit-sized structs, while fixing the mess I made

Since we want nice debug output, we need to `#[derive(Debug)]`.
Say we have:

```rust
#[bitsize(2)]
#[derive(FromBits, Debug)]
struct Thing {
    a: bool,
    b: bool,
}

fn main() {
    let thing = Thing::from(u2::new(0b11));
    println!("{thing:?}");
}
```

In this case, we'd just see `Thing { value: 3 }` printed. What we actually want to see are the bitfields, meaning we need to have a `DebugBits` macro reading the field information. Since it's not really special, let's skip talking about it.

Until this point, I had this hacky solution of passing field information across the macro boundary by using an internal, inert attribute.
Then I finally realized I could adjust the macro organization a bit and save a lot of sanity:

```rust
// Check a few invariants in bitsize and add
// bitsize_internal as a *last*, non-inert
// attribute macro, to execute it *last*.
#[bitsize(2)]
#[derive(DebugBits)]
struct Thing {
    a: bool,
    b: bool,
}

/* after the bitsize expansion */
// Derive Debug, which is easy since it can read all the field info
#[derive(DebugBits)]
#[bitsize_internal(2)]
struct Thing {
    a: bool,
    b: bool,
}

/* after the derive(DebugBits) expansion */
// Compress the fields, destroying field info *last*.
#[bitsize_internal(2)]
struct Thing {
    a: bool,
    b: bool,
}

/* after the bitsize_internal expansion */
// Done
struct Thing {
    value: u2,
}
```

At this point, the most important parts of `bilge` are done.

Later on I added constructors and all these _fallible nested-struct enum-tuple array field accesses_ I mentioned before (which is a bit more involved).

## How bilge works

Our mouse packet example now looks like this:

```rust
#[bitsize(24)]
#[derive(FromBits)]
struct MousePacket {
    left_clicked: bool,
    right_clicked: bool,
    middle_clicked: bool,
    other_important_bits: u5,
    x_movement: u8,
    y_movement: u8,
}
```

The frontend is pretty similar to `modular-bitfield`, though `FromBits` and others are their own macro and the arbitrary width integers can be used for calculations, like primitives.

We use `#[derive(FromBits)]` to also allow parsing the packet from raw bytes, similar to what `modular-bitfield` enables by default:

```rust
MousePacket::from(0b00011000_00000011_00001_0_1_0);
```

Without it, we can still use the generated constructor:

```rust
MousePacket::new(0, 1, 0, 0b00001, 0b00000011, 0b00011000);
```

Of course we also have getters and setters.

### a register-mapping showcase

Just to show you another "real" example, here's the _bilge_ based definition of an ARM register, without documentation:

```rust
#[bitsize(64)]
#[derive(DebugBits, TryFromBits)]
#[doc(alias("GICR_TYPER"))]
struct RedistributorType {
    affinity_value: [u8; 4],
    max_ppi_num: MaxPpiIntId,
    virtual_sgi_supported: bool,
    #[doc(alias("CommonLPIAff"))]
    lpi_config_shared_by: LpiConfigSharers,
    processor_number: u16,
    resident_vpe_recorded_as_id: bool,
    mpam_supported: bool,
    control_dpgs_supported: bool,
    is_last: bool,
    direct_lpi_supported: bool,
    dirty: bool,
    virtual_lpi_supported: bool,
    physical_lpi_supported: bool,
}

#[bitsize(5)]
#[derive(Debug, TryFromBits)]
enum MaxPpiIntId {
    _31,
    _1087,
    _1119
}

#[bitsize(2)]
#[derive(Debug, FromBits)]
enum LpiConfigSharers {
    All,
    SameAffinity3,
    SameAffinity3_2,
    SameAffinity3_2_1,
}

fn main() {
    let value_from_hardware = 0x4_0000_0000;
    let rt = RedistributorType::try_from(value_from_hardware).unwrap();
    // panicked, since we got a wrong value for `MaxPpiIntId`
}
```

These doc aliases [are pretty handy as well](https://github.com/rust-lang/rust-analyzer/pull/14775).

### easy usage explanations

If you want to start using `bilge`, the `README` contains some relevant information and usage:

[crates.io/crates/bilge](https://crates.io/crates/bilge)

[lib.rs/crates/bilge](https://lib.rs/crates/bilge)

[github.com/hecatia-elegua/bilge](https://github.com/hecatia-elegua/bilge)

Feel free to talk about more usecases on github!

## Conclusion

By using a few ideas like "attribute macros as a scope" and "type-centric design", we made our bitfield macro code a lot simpler while providing an easy frontend and without sacrificing performance, only increasing build times by a small amount compared to handmade code.

I specifically want to reduce bit fiddling to a minimum, which means there are some open usecases to address (there might be more bitflags functionality and I want to add strides).

The frontend and API should be as forward-compatible as possible to an integration of bitfields into rust, but _I think_ I can't prophesy.

Having said and built all this, I want to thank you for reading this far!

This was my first longform post and took quite a while to write. I hope this finds some people who maybe know more about rust's internals and could guide me or help implementing arbitrary sized integers and bitfields into rust.

If any bitfield crate maintainers read this: Could we somehow open up a working group / chat to merge at least some of our efforts and get bitfields into rust?

[Discuss this article on /r/rust](https://www.reddit.com/r/rust/comments/13ic0mf/no_more_bit_fiddling_and_introducing_bilge/)

<br><br><br>

[^1]: If you're wondering what I think about the proliferation of bitfield crates, to quote another maintainer: "I'm looking forward to the day where they can all die in a big fire as Rust includes it in the language."
