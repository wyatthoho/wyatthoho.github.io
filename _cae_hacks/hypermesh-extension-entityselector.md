---
layout: post
name: HyperMesh Extension - Embedding an Entity Selector in a Dialog
birth: 2026-10-07
---

Native HyperMesh panels have a familiar pattern: 
a button shows something like `12 Elements`. 
Click it, and the button turns into a selector with a few small icons 
(advanced selection, reset, OK, cancel). 
Confirm or cancel, and it collapses back into the button.

![The native Apply Pressures panel in HyperMesh, showing the entity selector](../assets/images/native-hm-selector.png)

The screenshot above is from the native **Apply Pressures** function.

Unlike basic widgets such as `hwtk::button` and `hwtk::combobox`, 
the official documentation provides no code examples for this control.

I wanted the same control inside my own HyperMesh Extension dialog. 
I spent time digging through the Tcl files in the HyperWorks 
installation directory and found that it is not a single widget, 
but a combination of several widgets working together. 
This article describes one way to implement it, based on what I found.

---

## Step 1: Create the Widgets

You can paste the code blocks of this article, in order, into the Tcl terminal of HyperMesh. 
After the last step, the dialog appears.

First, a namespace to hold the selection that is saved while the user is selecting (see Step 3), 
and the dialog with the containers for the control:

```tcl
namespace eval ::entityselector_test {
    variable saved_ids {}
}

catch { destroy .dialog }

set dialog          [hwtk::dialog .dialog -title "Test"]
set parent          [$dialog recess]
set frame_container [hwtk::frame $parent.frame_container]
set label           [hwtk::label $frame_container.label -text "Entities:"]
```

The control itself is assembled from these widgets:

| Widget | Role |
| :--- | :--- |
| `hmtk::entityselector` | The selector itself. Owns the selection logic and the graphics-area interaction. |
| `hwctx::guidebar` | The guide bar that visually hosts the selector. |
| `hwtk::button` | A *dummy button* that shows the current count, e.g. `12 Elements`. |
| `hwtk::buttonbar` | My own OK / Cancel / Reset / Advanced icons, replacing the built-in ones. |

Besides these, a few plain frames are needed to group them. 
Note that `frame_buttonbar` is created inside the **toplevel** (`[winfo toplevel $parent]`), 
not inside `frame_cell`. 
That allows it to be placed over the cell and raised above its siblings later.

```tcl
set frame_cell      [hwtk::frame $frame_container.frame_cell]
set guidebar        [hwctx::guidebar $frame_cell.guidebar -fillet 2]
set frame_button    [hwtk::frame $frame_cell.frame_button]
set button          [hwtk::button $frame_button.button]
set frame_buttonbar [hwtk::frame [winfo toplevel $parent].frame_buttonbar]
set entityselector  [hmtk::entityselector $frame_cell.entityselector \
    -guidebar           $guidebar \
    -types              "Elements Nodes" \
    -defaultentity      "Elements" \
    -selectmode         multiple \
    -isembeddedselector 1 \
    -showcount          1 \
    -showreset          0 \
    -showadvanced       0 \
    -showaccept         0 \
    -showcancel         0 \
    -syncwithbrowser    false \
    -restorelastentity  0 \
    -useeventhandler    true \
    -acceptcommand      [list ::entityselector_test::on_accept $frame_cell.entityselector $frame_buttonbar $button $frame_button] \
    -cancelcommand      [list ::entityselector_test::on_cancel $frame_cell.entityselector $frame_buttonbar $button $frame_button]
]
set buttonbar       [hwtk::buttonbar $frame_buttonbar.buttonbar -showseparator 0]
```

A few options of the selector deserve explanation:

- `-guidebar`: ties the selector to the guide bar.
- `-types` and `-defaultentity`: the entity types the user can switch between, and the one chosen initially.
- `-isembeddedselector 1`: marks the selector as embedded in a dialog rather than a standalone panel.
- `-showreset`, `-showadvanced`, `-showaccept`, `-showcancel` all set to `0`: 
  hides the built-in icons. 
  I draw these myself with the button bar so they can sit at the corner of the cell.
- `-showcount 1`: shows the number of selected entities.
- `-syncwithbrowser false` and `-restorelastentity 0`: 
  keep the selector independent of the model browser and of whatever was selected last time.
- `-acceptcommand` and `-cancelcommand`: the procedures to run on accept and cancel. 
  They are covered in Step 4.

---

## Step 2: Lay Out the Widgets

The key idea is that the dummy button and the selector occupy the **same grid cell**, 
with the dummy button stacked on top. 
Only one of them is visible at a time, 
depending on whether the user is currently selecting.

The `entityselector` has no appearance of its own, 
and it is never gridded. 
It borrows the appearance of the `guidebar` passed in through `-guidebar`, 
so the guide bar is what actually shows up on screen.

Here is the complete layout code:

```tcl
grid $label           -row 0 -column 0 -sticky w    -padx 4 -pady 4
grid $frame_container -row 0 -column 0 -sticky nsew -padx 4 -pady 4
grid $frame_cell      -row 0 -column 1 -sticky ew   -padx 4 -pady 4
grid $guidebar        -row 0 -column 0 -sticky ew
grid $button          -row 0 -column 0 -sticky ew
grid $buttonbar       -row 0 -column 0

grid rowconfigure    $parent 0 -weight 1
grid columnconfigure $parent 0 -weight 1
grid columnconfigure $frame_container 1 -weight 1
grid columnconfigure $frame_cell 0 -weight 1
grid columnconfigure $frame_button 0 -weight 1
```

The first three lines are ordinary dialog layout: 
the label and the cell sit side by side inside `frame_container`. 
The interesting part is what happens inside `frame_cell`:

- `$guidebar` is gridded at row 0, column 0.
- `$frame_button`, which holds the dummy button, is gridded into the same cell 
  by `_update_dummy_button` (see Step 3). 
  Because it is gridded later, it covers the guide bar. 
  At rest, the user only sees the dummy button.
- `$buttonbar` is gridded only inside its own frame, `frame_buttonbar`. 
  That frame is **not** laid out in the dialog yet. 
  It is placed over the cell only when the user starts selecting (see Step 3).
- The `-weight 1` settings on `$frame_cell` and `$frame_button` 
  let the guide bar and the dummy button stretch to the full width of the cell.

---

## Step 3: Configure the Dummy Button

The dummy button's label shows the entity type and the count. 
The helper below grids the button frame into the cell and updates the label. 
It is called once at startup (see Step 5), and again after Accept or Cancel.

```tcl
proc ::entityselector_test::_update_dummy_button {frame_button button entitytype {count 0}} {
    grid $frame_button -row 0 -column 0 -sticky nsew
    $button configure -text "$count $entitytype"
}
```

Clicking the dummy button starts a selection session.

```tcl
$button configure \
    -command [list ::entityselector_test::activate_selector $frame_cell $frame_button $frame_buttonbar $entityselector]
```

```tcl
proc ::entityselector_test::activate_selector {frame_cell frame_button frame_buttonbar entityselector} {
    variable saved_ids
    set saved_ids [$entityselector ExecSelectionCommand GetSelectionIds]

    grid forget $frame_button
    place $frame_buttonbar -in $frame_cell -anchor ne \
        -x [winfo width $frame_cell] -y [winfo height $frame_cell]
    raise $frame_buttonbar
    $entityselector SetActive
    $entityselector UpdateButtonWidth
}
```

1. Save the current selection to `saved_ids`, so that Cancel can restore it later.
2. Hide the dummy button with `grid forget`, revealing the guide bar underneath.
3. Float the button bar at the corner of the cell and raise it.
4. Activate the selector with `SetActive`.

---

## Step 4: Configure the Button Bar

Each small button in the button bar forwards to a method of the selector. 
The icon names are the ones HyperWorks itself uses, 
so the result looks identical to the native panels.

```tcl
$buttonbar add a_advance \
    -image "toolbarMoreOptionsStrip-16.png" -indicator hide \
    -help "Advanced Selection" \
    -command [list $entityselector OpenAdvancedSelection]
$buttonbar add a_reset \
    -image "toolbarActionResetStrip-16.png" -indicator hide \
    -help "Reset" \
    -command [list $entityselector ExecSelectionCommand Clear]
$buttonbar add a_accept \
    -image "toolbarActionOKStrip-16.png" -indicator hide \
    -help "Ok" \
    -command [list ::entityselector_test::on_accept $entityselector $frame_buttonbar $button $frame_button]
$buttonbar add a_cancel \
    -image "toolbarActionCancelStrip-16.png" -indicator hide \
    -help "Cancel" \
    -command [list ::entityselector_test::on_cancel $entityselector $frame_buttonbar $button $frame_button]
```

Advanced Selection and Reset need no extra code. 
Accept and Cancel have to switch the UI back to the dummy button, 
so they call our own procedures.

### Accept

```tcl
proc ::entityselector_test::on_accept {entityselector frame_buttonbar button frame_button} {
    set ids        [$entityselector ExecSelectionCommand GetSelectionIds]
    set entitytype [$entityselector GetEntityType]
    $entityselector SetInactive
    place forget $frame_buttonbar
    ::entityselector_test::_update_dummy_button $frame_button $button $entitytype [llength $ids]
    raise $frame_button
}
```

Read the selection, deactivate the selector, remove the button bar, 
and bring the dummy button back with the new count as its label.

### Cancel

```tcl
proc ::entityselector_test::on_cancel {entityselector frame_buttonbar button frame_button} {
    variable saved_ids
    $entityselector ExecSelectionCommand Clear
    if {[llength $saved_ids]} {
        $entityselector ExecSelectionCommand SelectByAdvanced "by id" $saved_ids
    }
    set entitytype [$entityselector GetEntityType]
    $entityselector SetInactive
    place forget $frame_buttonbar
    ::entityselector_test::_update_dummy_button $frame_button $button $entitytype [llength $saved_ids]
    raise $frame_button
}
```

The selector has no built-in "undo" for a selection session. 
So Cancel clears whatever was picked and re-selects the ids saved at the start, 
using `SelectByAdvanced "by id"`. 
The rest is identical to Accept, except that the count comes from `saved_ids`.

---

## Step 5: Show the Dialog

Initialize the dummy button, then post the dialog:

```tcl
::entityselector_test::_update_dummy_button $frame_button $button "Elements"

$dialog post
```

The dialog now shows the label `Entities:` and a button `0 Elements`. 
Click the button, pick some elements in the graphics area, 
and press OK or Cancel.

![The test dialog showing the Entities label and the dummy selector button](../assets/images/entityselector-test.png)

---

## Useful Selector Methods

Everything above relies on a handful of methods:

| Method | Purpose |
| :--- | :--- |
| `SetActive` / `SetInactive` | Start / stop the selection session. |
| `ExecSelectionCommand GetSelectionIds` | Get the ids currently selected. |
| `ExecSelectionCommand Clear` | Clear the selection. |
| `ExecSelectionCommand SelectByAdvanced "by id" $ids` | Select entities by id. |
| `GetEntityType` | Current entity type, e.g. `Elements` or `Nodes`. |
| `OpenAdvancedSelection` | Open the advanced selection dialog. |
| `UpdateButtonWidth` | Refresh the selector layout after activation. |
