# Creating pads with Extract

The supported way to create a pad is to let the app's own Extract command write
the WAV and assign the pad. The SDK arms a rendered buffer and calls Extract; the
app keeps ownership of the file, the pad and its undo state.

## The request

```rust
use mpc_sample_sdk::platform::extract::{self, Request};

fn create_pad() {
let screen = mpc_sample_sdk::platform::page::screen();
let Some(selected) = mpc_sample_sdk::platform::selection::current(screen) else {
    return;
};

let audio = vec![0.0_f32; selected.frames as usize * selected.channels as usize];
let request = Request {
    screen,
    state: selected.state,
    name: c"Demo",
    buffer: Some(&audio),
    source_frames: selected.frames,
    channels: u32::from(selected.channels),
    start: 0,
    end: selected.frames,
};

match extract::run(&request) {
    Ok(extracted) => {
        mpc_sample_sdk::platform::status::set_status(&format!(
            "created pad {}",
            extracted.pad + 1
        ));
    }
    Err(error) => {
        mpc_sample_sdk::platform::status::set_status(&format!("extract: {error}"));
    }
}
}
```

`source_frames` is the source sample length, and `start..end` is the range the
app should copy. The buffer may be longer than the range for an effect tail; the
SDK's writer substitutes it block by block.

## Check room first

```rust
use mpc_sample_sdk::platform::{pads, selection};

fn has_room() {
let screen = mpc_sample_sdk::platform::page::screen();
let Some(selected) = selection::current(screen) else {
    return;
};
let program = selection::program(selected.state);

if !pads::has_room(program, 1) {
    mpc_sample_sdk::platform::status::set_status("need 1 empty pad");
    return;
}
}
```

`has_room` returns `false` for an unknown program. That is deliberate: the mod
cannot prove there is room.

## Effects and synth takes

- **Effect:** pass the source frame range and a buffer that contains the rendered
  source plus the effect tail. The app extends the last block.
- **Synth:** the take may be longer or shorter than the source. Use the take's
  length for `end` when it must be the new sample length, or clamp to the source
  if you want only the source-sized part.
- **HPSS:** run Extract twice, once for each part; check `pads::has_room` for
  two pads before rendering.

`extract::run` may block for up to about three seconds while the app assigns the
pad. Do not call it from a paint callback for a large job; use the task pattern
and keep the UI showing progress.
