+++
title = "Understanding Serde"
date = 2020-09-09
+++

I have been enjoying learning Rust lately, slowly learning modules and crates which I feel are "essential" for any Rustacean to be familiar with. And one of the most popular crates is without a doubt [Serde](https://serde.rs/). In its own words, Serde is "a framework for serializing and deserializing Rust data structures efficiently and generically."

Basically it allows you to convert different data formats to Rust types and vice versa. Have a JSON string? Just declare a `struct` with the format and use `serde_json` to deserialize it automatically like so:

```rust
use serde::{Deserialize, Serialize};
use serde_json::{json, Result};

#[derive(Serialize, Deserialize)]
struct Entry {
    id: u32,
    name: String,
    value: String,
}

fn main() -> Result<()> {
    // have some JSON string data?
    let data = r#"{
            "id": 152,
            "name": "pasta",
            "value": "yummy food"
        }"#;

    // deserialize it into Rust type
    let p: Entry = serde_json::from_str(data)?;
    println!("{} is {}", p.name, p.value);
    // and serialize it back to JSON string data
    println!("{}", json!(p).to_string());
    Ok(())
}
```

Simple enough. So what's with the title? Well, while Serde is pretty trivial to use given a data format crate (`serde_json`, `quick_xml`, etc), it is not nearly as trivial to create your own data format. I have been struggling to understand Serde deserialization for an _embarrassingly_ long amount of time. So I just want to cover some of the pitfalls I ran into when learning this beast so hopefully you don't have to.

Just to be clear: I am not trying to bash on Serde for being a pain to use. It is a powerful library and already seems like an essential part of the Rust ecosystem. But what makes Serde fast, safe, and generic are also what makes it complicated to learn. The Serde docs cover each component well, but fails to connect the components together, which is what I hope to do in this post.

## Deserializer

So lets start by looking at the [`Deserializer` documentation](https://serde.rs/impl-deserializer.html). Alright, so it looks like a `Deserializer` is basically just a type that implements a bunch of `deserialize_x` methods which each call a function on a `Visitor`. Simple enough, except what's a `Visitor`? And which function am I supposed to call? And where is the actual parsing happening?

Alright, so looking straight at those docs didn't seem to help that much, so lets tackle these questions one by one.

### What is a `Visitor`?

`Visitor`s are explained in more detail in the [`Deserialize` documentation](https://serde.rs/impl-deserialize.html).

Fortunately, it turns out that understanding `Visitor`s isn't all that important for understanding `Deserializer`s. But the general idea is that a `Visitor` is where you define the data types you _could_ accept when converting to the data type you _want_. Take the `I32Visitor` for example. An `i32` is the type we want, and to get an `i32`, we allow visiting most other integer types as long as they fit in an `i32`.

This is pretty straight forward for primitives, but gets much more complicated for structured types like `struct`, `Vec`, etc.

### What `Visitor` function should I call?

This was also pretty straight forward in hindsight. For the most part, you just call `visit_X` for `deserialize_X`. However, the problem is figuring out which function to call when this general principle _doesn't_ hold. For example, `deserialize_identifier` doesn't have a corresponding `visit_identifier` function. So what should we call?

The short answer is that the default derive for a `struct` allows the identifier to be one of two types:

1. a `&str` - the literal name of one of the fields in the struct

2. an index - the position of one of the fields in the struct

As far as I could tell, this essential information isn't clearly laid out in the Serde documentation anywhere. To figure this out, I personally had to look at the expanded code from a `#[derive(Deserialize)]`'d example `struct`. So lets do that here!

### Understanding `#[derive(Deserialize)]`

Take the following code:

```rust
#[derive(Deserialize)]
struct Duration {
    secs: u64,
    nanos: u32,
}
```

This would end up creating a deserialize implementation similar to:

```rust
impl<'de> Deserialize<'de> for Duration {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
    where
        D: Deserializer<'de>,
    {
        // creates an enum to represent the fields
        enum Field { Secs, Nanos };

        // creates a deserialize for the internal field enum
        impl<'de> Deserialize<'de> for Field {
            fn deserialize<D>(deserializer: D) -> Result<Field, D::Error>
            where
                D: Deserializer<'de>,
            {
                // which has a visitor for the fields using the two
                // visit types we talked about above
                struct FieldVisitor;

                impl<'de> Visitor<'de> for FieldVisitor {
                    type Value = Field;

                    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                        formatter.write_str("`secs` or `nanos`")
                    }

                    // 1. as a &str - matches against the field strings
                    fn visit_str<E>(self, value: &str) -> Result<Field, E>
                    where
                        E: de::Error,
                    {
                        match value {
                            "secs" => Ok(Field::Secs),
                            "nanos" => Ok(Field::Nanos),
                            _ => Err(de::Error::unknown_field(value, FIELDS)),
                        }
                    }

                    // 2. as an index - uses the declared field order
                    // note there should also be ones for u8, u16, etc
                    fn visit_i32<E>(self, value: i32) -> Result<Field, E>
                    where
                        E: de::Error,
                    {
                        match value {
                            0 => Ok(Field::Secs),
                            1 => Ok(Field::Nanos),
                            _ => Err(de::Error::unknown_field(value, FIELDS)),
                        }
                    }
                }

                // this is where `deserialize_identifier` is called
                // more on this later
                deserializer.deserialize_identifier(FieldVisitor)
            }
        }

        struct DurationVisitor;

        // creates a visitor for the actual struct
        impl<'de> Visitor<'de> for DurationVisitor {
            type Value = Duration;

            fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                formatter.write_str("struct Duration")
            }

            // this function is called by our deserializer
            // more on this later
            fn visit_map<V>(self, mut map: V) -> Result<Duration, V::Error>
            where
                V: MapAccess<'de>,
            {
                let mut secs = None;
                let mut nanos = None;
                // `next_key` should be the function which eventually
                // leads to `deserialize_identifier` above being called
                while let Some(key) = map.next_key()? {
                    match key {
                        Field::Secs => {
                            if secs.is_some() {
                                return Err(de::Error::duplicate_field("secs"));
                            }
                            // `next_value` will eventually call `deserialize_X`
                            secs = Some(map.next_value()?);
                        }
                        Field::Nanos => {
                            if nanos.is_some() {
                                return Err(de::Error::duplicate_field("nanos"));
                            }
                            // `next_value` will eventually call `deserialize_X`
                            nanos = Some(map.next_value()?);
                        }
                    }
                }
                // finally, build the resulting struct and return it
                let secs = secs.ok_or_else(|| de::Error::missing_field("secs"))?;
                let nanos = nanos.ok_or_else(|| de::Error::missing_field("nanos"))?;
                Ok(Duration::new(secs, nanos))
            }
        }

        // this is the starting point for the whole thing!
        // i.e. when deserializing into a struct, the first function that
        // gets called on our deserializer is `deserialize_struct`
        const FIELDS: &'static [&'static str] = &["secs", "nanos"];
        deserializer.deserialize_struct("Duration", FIELDS, DurationVisitor)
    }
}
```

That was a lot of code (most of which was copied and then annotated from the [deserialize struct docs](https://serde.rs/deserialize-struct.html)). But seeing this laid out as the starting point really helped me to better understand the `Deserializer` code. So let's go back to our deserialization.

### How do `Deserializer` functions get called?

As a quick reminder, lets look at our basic `Deserializer` functions again. I'm going to skip some types (ex. `unit_struct`) for simplicity. The main ones are:

* `deserialize_X` where `X` is a primitive (`i32`, `String`, etc)
* `deserialize_any`
* `deserialize_ignored_any`
* `deserialize_identifier`
* `deserialize_seq`
* `deserialize_map/struct`

Lets go over these one by one in a little more detail.

#### `deserialize_X`

These are for deserializing primitives. For example, `deserialize_i32` should parse an `i32` from the input and visit it. Really not much more explanation needed.

#### `deserialize_any`

This is not necessarily that useful in most cases I've run into. It can be used in conjunction with `forward_to_deserialize_any` whenever you either can't tell what data type it is. Then you use current parser state or lookahead to tell what it should be. For example, in JSON if you see a `{` you know you are starting a dictionary.

#### `deserialize_ignored_any`

This function is, in my opinion, more important (and less clear) than its non-ignored brethren. It is called whenever the deserializer (a struct probably) finds data it doesn't need in the resulting deserialized type. While the resulting value isn't important, the parser _needs_ to be advanced. More on this later.

#### `deserialize_identifier`

We talked in great detail previously, but this function is called when the deserializer needs a map key or struct identifier.

#### `deserialize_seq`

This function is called when we start parsing a sequence like a `Vec`. It does _not_ result in the above `deserialize_identifier` being called, but should result in the deserialize function for the inner sequence type being called.

#### `deserialize_map/struct`

These two are very similar. They both follow the same pattern of calling `deserialize_identifier` followed by a deserialize function matching the identified field's type.

Now let's look at another example of deserializing our `struct Duration`:

```rust
// this basically just calls `Duration::deserialize()`
// which kicks off deserialization (pseudo-impl above)
let d: Duration = serde_json::from_str(data)?;
```

And some example JSON input and how this data translates to calls in our deserializer:

```json
// since we are deserializing a struct, the first thing
// to get called is `deserialize_struct`
data = {
    // which should result in `deserialize_identifier` getting called
    // which parses and visits "secs"
    // v
    "secs": 12,
    //      ^
    // since secs is a u64, `deserialize_u64` ends up getting called
    // which parses and visits 12

    // another `deserialize_identifier` should get called
    // which parses and visits "extra"
    // v
    "extra": "data",
    //        ^
    // but since this field isn't in the Duration struct,
    // it results in `deserialize_ignored_any` being called
    // indicating the next value needs to be parsed even though
    // we aren't storing it in the final struct

    // and then another `deserialize_identifier` should get called
    // which parses and visits "nanos"
    // v
    "nanos": 34
    //       ^
    // since nanos is a u32, `deserialize_u32` ends up getting called
    // which parses and visits 34

    // at this point the `MapAccess` should know that there are no more
    // keys and should stop parsing the struct
}
// `deserialize_struct` returns with our data!
```

### Where should parsing happen?

And finally let's talk about when parsing actually happens. You can probably guess when you should parse from the calls laid out above, but it seems like parsing usually happens in the `deserialize_X` functions. For example, a call to `deserialize_struct` parses the `{` out of a JSON stream, and the return from `deserialize_struct` should make sure that `}` has been parsed as well.

I personally initially thought that some parsing should happen in `MapAccess` and `SeqAccess`, but that code all got moved into the `deserialize_struct` function and friends. Obviously, this is more of a "do whatever works" thing, but parsing in these trait functions caused me more problems than solutions in my last project.

## Wrapping up

Hopefully Serde deserialization makes a lot more sense after looking through some of the generated code and the generated calls from example input data. These are the kind of connections I wish would have been laid out in the official documentation. Either way, Serde is awesome, and this definitely won't be my last time writing a `Deserializer`.
