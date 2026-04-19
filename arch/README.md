Platform-specific implementations of the same interface, selected at build time.
Examples: arch/linux/aarch64, arch/darwin/x86-x64, arch/windows/ for OS-level differences.

The choice is exclusive: one arch is picked per build, and the others are not
supported in that build's output. Switching archs means rebuilding. This is
what separates arch/ from adapters/ — adapters coexist in the same build and
are selected per caller or by config, while archs are mutually exclusive at
compile time.
