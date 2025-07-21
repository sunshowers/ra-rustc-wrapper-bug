# rust-analyzer rustc wrapper bug

There is a cache invalidation bug caused by a combination of rust-analyzer's rustc wrapper and Cargo.

Tested against rust-analyzer at `9a1ee18e4dccc29c41d5c642860e58641d5ed0de` and Cargo 1.88.

## Steps to reproduce

1. Clone this repo and cd into it.

2. Run `cargo check -v` -- for me it produces:

  ```
  % cargo check -v
      Checking ra-rustc-wrapper-bug v0.1.0 (/home/rain/dev/ra-rustc-wrapper-bug)
     Running `/opt/rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc --crate-name ra_rustc_wrapper_bug --edition=2024 src/main.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts,future-incompat --diagnostic-width=180 --crate-type bin --emit=dep-info,metadata -C embed-bitcode=no -C debuginfo=2 --check-cfg 'cfg(docsrs,test)' --check-cfg 'cfg(feature, values())' -C metadata=ff2adbd4766bc495 -C extra-filename=-19dfec7f77f210bd --out-dir /home/rain/dev/ra-rustc-wrapper-bug/target/debug/deps -C incremental=/home/rain/dev/ra-rustc-wrapper-bug/target/debug/incremental -L dependency=/home/rain/dev/ra-rustc-wrapper-bug/target/debug/deps`
      Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
  ```

3. Run `cargo check -v` again to ensure that the build is fresh:

  ```
  % cargo check -v
         Fresh ra-rustc-wrapper-bug v0.1.0 (/home/rain/dev/ra-rustc-wrapper-bug)
      Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.00s
  ```

4. Check the `.rmeta` file (the path might be different for you):

  ```
  % stat target/debug/deps/libra_rustc_wrapper_bug-19dfec7f77f210bd.rmeta
    File: target/debug/deps/libra_rustc_wrapper_bug-19dfec7f77f210bd.rmeta
    Size: 0               Blocks: 0          IO Block: 4096   regular empty file
  Device: 259,4   Inode: 4823540     Links: 1
  Access: (0644/-rw-r--r--)  Uid: ( 1000/    rain)   Gid: ( 1000/  ubuntu)
  Access: 2025-07-21 19:38:17.245213032 +0000
  Modify: 2025-07-21 19:38:17.245213032 +0000
  Change: 2025-07-21 19:38:17.245213032 +0000
   Birth: -
  ```

4. Run `touch src/main.rs` -- this should cause Cargo's cache to be invalidated.
5. Run `RA_RUSTC_WRAPPER=1 RUSTC_WRAPPER=~/dev/rust-analyzer/target/debug/rust-analyzer cargo check -v`:

  ```
  % RA_RUSTC_WRAPPER=1 RUSTC_WRAPPER=~/dev/rust-analyzer/target/debug/rust-analyzer cargo check -v
         Dirty ra-rustc-wrapper-bug v0.1.0 (/home/rain/dev/ra-rustc-wrapper-bug): the file `src/main.rs` has changed (1753126805.099594533s, 1m 48s after last build at 1753126697.231212978s)
      Checking ra-rustc-wrapper-bug v0.1.0 (/home/rain/dev/ra-rustc-wrapper-bug)
       Running `/home/rain/dev/rust-analyzer/target/debug/rust-analyzer /opt/rustup/toolchains/stable-x86_64-unknown-linux-gnu/bin/rustc --crate-name ra_rustc_wrapper_bug --edition=2024 src/main.rs --error-format=json --json=diagnostic-rendered-ansi,artifacts,future-incompat --diagnostic-width=180 --crate-type bin --emit=dep-info,metadata -C embed-bitcode=no -C debuginfo=2 --check-cfg 'cfg(docsrs,test)' --check-cfg 'cfg(feature, values())' -C metadata=ff2adbd4766bc495 -C extra-filename=-19dfec7f77f210bd --out-dir /home/rain/dev/ra-rustc-wrapper-bug/target/debug/deps -C incremental=/home/rain/dev/ra-rustc-wrapper-bug/target/debug/incremental -L dependency=/home/rain/dev/ra-rustc-wrapper-bug/target/debug/deps`
      Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.05s
  ```

  Note that this build does not cause the output `.rmeta` to be updated.

6. Run `cargo check -v` again:

  ```
  % cargo check -v
         Fresh ra-rustc-wrapper-bug v0.1.0 (/home/rain/dev/ra-rustc-wrapper-bug)
      Finished `dev` profile [unoptimized + debuginfo] target(s) in 0.02s
  ```

Cargo now believes the build is fresh, but the `.rmeta` is still the same one from before!

  ```
  % stat target/debug/deps/libra_rustc_wrapper_bug-19dfec7f77f210bd.rmeta
    File: target/debug/deps/libra_rustc_wrapper_bug-19dfec7f77f210bd.rmeta
    Size: 0               Blocks: 0          IO Block: 4096   regular empty file
  Device: 259,4   Inode: 4823540     Links: 1
  Access: (0644/-rw-r--r--)  Uid: ( 1000/    rain)   Gid: ( 1000/  ubuntu)
  Access: 2025-07-21 19:38:17.245213032 +0000
  Modify: 2025-07-21 19:38:17.245213032 +0000
  Change: 2025-07-21 19:38:17.245213032 +0000
   Birth: -
  ```

## Why this happens

Cargo uses mtime-based cache invalidation, through the following scheme (paths might be different for you):

* There's a fingerprint file: `target/debug/.fingerprint/ra-rustc-wrapper-bug-19dfec7f77f210bd/dep-bin-ra-rustc-wrapper-bug`. This fingerprint file contains the list of source files (in this case `src/main.rs`).
* Cargo compares the mtimes of each source file with the mtime of this fingerprint file. If any of the source files have an mtime newer than that of the fingerprint file, the build is considered *dirty* and is rebuilt, otherwise *fresh*
* How is the fingerprint file generated? rustc outputs its own fingerprint file ending in `.d`: `target/debug/deps/ra_rustc_wrapper_bug-19dfec7f77f210bd.d`. After a build completes, Cargo reads this file and updates its fingerprint file based on the contents of this `.d` file.

Now, when rust-analyzer's `RUSTC_WRAPPER` is invoked with a check build:

* The wrapper [skips over](https://github.com/rust-lang/rust-analyzer/blame/9a1ee18e4dccc29c41d5c642860e58641d5ed0de/crates/rust-analyzer/src/bin/rustc_wrapper.rs#L41-L43) the build and exits with code 0
* Cargo is unaware of this skip and believes the build is successful
* Cargo looks for the `.d` file output by rustc
* Cargo finds the file produced by the first `cargo check` run -- a file that is now outdated, but Cargo isn't aware of this
* Cargo updates its fingerprint file

As a result, the actual `.rmeta` output isn't up-to-date, but Cargo thinks it's up-to-date

## Possible solutions

* The rust-analyzer rustc wrapper could delete the `.d` file as part of skipping the build. The path to the `.d` file can be constructed through a combination of `--out-dir`, `--crate-name`, `--crate-type`, and `-C metadata`.
* Cargo could somehow be made aware that the build didn't happen.
