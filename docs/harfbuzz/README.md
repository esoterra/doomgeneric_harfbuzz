# harfbuzz doom

## TODOs

- [ ] Implement `doomgeneric_cliargs` test tool
  - Parse the input from arg or file and run the game
  - Output an image of the final frame
  - Bonus: try to add arg to output a video of all frames
- [ ] Experiment with building fonts
  - Experiment with `Bad-Apple-Font`
  - Experiment with `harfbuzz-wasm-examples`
- [ ] Get doomgeneric building as a font
- [ ] Draw the frame output as glyphs

## Premise

It's possible to write WebAssembly (Wasm) code that is used to translate a string of text into the combination of Glyphs that will be rendered in a font using harfbuzz with the wasm shaper turned on.

So, what if we made it so that a string of text was interpreted as inputs to the game doom and "ticks" (representing time passing) and the shaped output was the resulting frame of doom?

This would allow you to theoretically* play doom in a text field as long as you have a way of appending the "tick" characters consistently.

*practically the wasm shaper is not enabled in any broadly available applications so you have to modify an application to turn it on for it to work.

## Interfaces

The simplest way to implement this appears to be to bridge two interfaces: the harfbuzz wasm-shaper API and the doomgeneric interface.

### harfbuzz wasm-shaper

[docs](https://github.com/harfbuzz/harfbuzz/blob/main/docs/wasm-shaper.md)

Harfbuzz supports "wasm shapers" which are wasm modules that export a `shape` function (see signature below) that takes in a buffer that represents the string being shaped and at the end of the execution contains the glyphs to be rendered.

```c++
uint32 shape(uint32 shape_plan_token, uint32 font_token, uint32 buffer_token, uint32 features_token, uint32 num_features);
```

The functions `buffer_copy_contents` and `buffer_set_contents` are used to read and write to the buffer respectively.

The glyph ids placed in the buffer are retrieved using the `font_get_glyph` function.

```c++
uint32 font_get_glyph(
    uint32 font_token,
    uint32 codepoint,
    uint32 variation_selector
)
```

**Open Questions:**

1. Is a new module instance used for each `shape` call or is it reused?
2. What does the `bool` returned by `buffer_copy_contents` and `buffer_set_contents` indicate? If failure, what kind of failure?
3. Is `buffer_contents` initially allocated using `buffer_contents_realloc`?

### doomgeneric

[docs](../../README.md)

doomgeneric is a source port of doom classic designed to be as easy to port as possible. We just have to implement these functions, call `doomgeneric_Create()` once, and then call `doomgeneric_Tick()` repeatedly for the game to run.

|Functions            |Description|
|---------------------|-----------|
|DG_Init              |Initialize your platform (create window, framebuffer, etc...).
|DG_DrawFrame         |Frame is ready in DG_ScreenBuffer. Copy it to your platform's screen.
|DG_SleepMs           |Sleep in milliseconds.
|DG_GetTicksMs        |The ticks passed since launch in milliseconds.
|DG_GetKey            |Provide keyboard events.
|DG_SetWindowTitle    |Not required. This is for setting the window title as Doom sets this from WAD file.

**Open Questions:**
1. Would there be any adverse affects to disabling draws to `DG_ScreenBuffer` while we replay the input log until we reach the last tick to improve performance?

## Design

### No-ops

Some of the expected functions should be able to be implemented as no-ops.
* `DG_Init` - available if we need it for our own initialization but not required for doom to run
* `DG_DrawFrame` - we don't want/need to render every frame, just the last one. so we won't be doing any work here.
* `DG_SetWindowTitle` - we don't need to set a title. maybe at some point we'll emit glyphs for this for fun but it's not needed.

### Time

The `DG_SleepMs` and `DG_GetTicksMs` functions will need to be mocked since we don't have the ability to sleep or interest in doing so. Both calling `DG_SleepMs` and advancing to the next frame should cause an internally tracked counter to advance.

**Open Questions:**
1. does `doomgeneric_Tick()` always advance to the next frame, waiting if needed, or will it return without advancing a frame if not enough time has passed?

### Input Handling

Iterate over the string of characters treating it like a sequence of frames.

Each `f` in the input represents the start of a new frame. Any other character indicates that the [corresponding key](#key-mapping-table) was pressed during that frame.

In practical terms, this means we'll need to
1. create a structure to hold the keys currently held down
2. set a key to pressed whenever we see its corresponding codepoint
3. implement `DG_GetKey` to read from the structure
4. when we encounter a `f`, advance the game to the next frame, then reset the keypress structure
5. repeat 2-4 until we reach the final frame

### Drawing

When we reach the end of the input, we need to turn the `DG_ScreenBuffer` into a sequence of glyphs. If we make some assumptions about line wrapping, we can essentially treat this as similar to [doom-ascii](https://github.com/wojciech-graj/doom-ascii), which is itself a doomgeneric port, and split the output into a grid of cells which we select a glyph for based on the color and intensity. Ideally, these would be colored glyphs, but it may be necessary to use dithering if it is too complicated to use color.

# Appendix

## Key Mapping Table

One challenge of making doom a font is that only keys which represent drawable characters can be used as inputs. So a custom codepoint to key mapping must be used.

| codepoint | key |
|-|-|
| w | `KEY_UPARROW`
| a | `KEY_LEFTARROW`
| s | `KEY_DOWNARROW`
| d | `KEY_RIGHTARROW`
| q | `KEY_STRAFE_L`
| e | `KEY_STRAFE_R`
| 0 | `KEYP_0`
| 1 | `KEYP_1`
| 2 | `KEYP_2`
| 3 | `KEYP_3`
| 4 | `KEYP_4`
| 5 | `KEYP_5`
| 6 | `KEYP_6`
| 7 | `KEYP_7`
| 8 | `KEYP_8`
| 9 | `KEYP_9`