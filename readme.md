[![crates.io](https://img.shields.io/crates/v/windows.svg)](https://crates.io/crates/windows)
[![docs.rs](https://docs.rs/windows/badge.svg)](https://docs.rs/windows)
[![Build and Test](https://github.com/microsoft/windows-rs/workflows/Build%20and%20Test/badge.svg?event=push)](https://github.com/microsoft/windows-rs/actions)

## Rust for Windows

The `windows` crate lets you call any Windows API past, present, and future using code generated on the fly directly from the metadata describing the API and right into your Rust package where you can call them as if they were just another Rust module.

The Rust language projection follows in the tradition established by [C++/WinRT](https://github.com/microsoft/cppwinrt) of building language projections for Windows using standard languages and compilers, providing a natural and idiomatic way for Rust developers to call Windows APIs. 

## Getting started

Start by adding the following to your Cargo.toml file:

```toml
[dependencies]
windows = "0.3.1"

[build-dependencies]
windows = "0.3.1"
```

This will allow Cargo to download, build, and cache Windows support as a package. Next, specify which types you need inside of a `build.rs` build script and the `windows` crate will generate the necessary bindings:

```rust
fn main() {
    windows::build!(
        windows::data::xml::dom::*
        windows::win32::system_services::{CreateEventW, SetEvent, WaitForSingleObject}
        windows::win32::windows_programming::CloseHandle
    );
}
```

Finally, make use of any Windows APIs as needed.

```rust
mod bindings {
    ::windows::include_bindings!();
}

use bindings::{
    windows::data::xml::dom::*,
    windows::win32::system_services::{CreateEventW, SetEvent, WaitForSingleObject},
    windows::win32::windows_programming::CloseHandle,
};

fn main() -> windows::Result<()> {
    let doc = XmlDocument::new()?;
    doc.load_xml("<html>hello world</html>")?;

    let root = doc.document_element()?;
    assert!(root.node_name()? == "html");
    assert!(root.inner_text()? == "hello world");

    unsafe {
        let event = CreateEventW(
            std::ptr::null_mut(),
            true.into(),
            false.into(),
            std::ptr::null(),
        );

        SetEvent(event).ok()?;
        WaitForSingleObject(event, 0);
        CloseHandle(event).ok()?;
    }

    Ok(())
}
```

To reduce build time, use a `bindings` crate rather simply a module. This will allow Cargo to cache the results and build your project far more quickly.

There is an experimental [documentation generator](https://github.com/microsoft/windows-docs-rs) for the Windows API. The documentation [is published here](https://microsoft.github.io/windows-docs-rs/). This can be useful to figure out how the various Windows APIs map to Rust modules and which `use` paths you need to use from within the `build` macro.

More examples [can be found here](examples). Robert Mikhayelyan's [Minesweeper](https://github.com/robmikh/minesweeper-rs) is also a great example.

## Safety

We aim to be fully compliant with Rust's safety guarantees. Unfortunately, some Windows APIs are not entirely compatible with Rust's safety semantics and thus need to be marked with `unsafe`. This does not mean these APIs are necessarily unsafe to use. It is possible that some APIs that are safe are still marked with `unsafe` due to the absence of a mechanism in Windows metadata for marking APIs as being compatible with Rust's safety semantics. A best attempt is made to map common idioms to safe Rust, but that is not always possible with older pointer-based APIs.

The `windows` crate makes two blanket policies around the safety of bindings:
* Win32 and COM bindings are marked with `unsafe`. It is up to the developer to read the documentation on how to call these APIs in a safe way.
* WinRT bindings are *not* marked with `unsafe` as the WinRT contract maps nicely to Rust's safety properties.

It is important to note that WinRT APIs are often implemented in non-memory safe languages like C++. Users can be sure that (modulo bugs), the WinRT bindings generated by this crate are 100% safe to use. Users should, however, make sure that the code that implements the APIs being called through the bindings are either written in safe Rust or have been audited for memory safety and correctly adhere to the WinRT contract. WinRT APIs written in safe Rust and consumed from Rust using the `windows` crate should therefore be 100% memory safe.

We take these safety guarantees very seriously. Please let us know if you run into issues where you see Rust's memory safety guarantees being violated using this crate.
