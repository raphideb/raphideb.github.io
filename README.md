Check the built website here: [https://crashdump.info](https://crashdump.info)

## Changes in version 1.1.0

- Multi-image selection: Shift + arrow keys, Shift/Ctrl + Home/End, Ctrl + A and click
  modifiers build a selection; Esc clears it.
- Marking and moving act on the whole selection, with per-file results, so one locked file no
  longer stops the rest.
- The automatic blink runs over the selection when two or more images are selected, wrapping
  within it. Starting the blink no longer clears the selection.
- Rotate: R turns the selected frames 180°, for subs taken after a meridian flip. Only the
  display is turned - the file is never written, and a debayered frame keeps its colours.
- Help: the keyboard reference moved to the top and became a table of keys and actions, shared
  with the hints in the control panel.
- Translations updated in all 15 languages for the new strings.
- New download page at https://crashdump.info/astronomy/blinkfits/, with a screenshot and
  instructions for checking the download.
- Tests added for selection and rotation.