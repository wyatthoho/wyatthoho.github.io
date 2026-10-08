---
layout: post
name: HyperMesh Extension - Embedding an Entity Selector in a Dialog
birth: 2026-10-07
---

HyperMesh has a familiar control called **entity selector** that lets the user select entities of a specific type, such as components, elements, or surfaces. The screenshot below shows it in the native Pressures panel.

![The native Apply Pressures panel in HyperMesh, showing the entity selector](../assets/images/native-hm-selector.png)

Clicking the button turns it into a selector with a few small icons (advanced selection, reset, apply, OK, and cancel). After a selection is made, the number of selected entities appears in parentheses.

Unlike basic widgets such as `hwtk::button` and `hwtk::combobox`, this control has no code examples in the official documentation. So I dug through the Tcl files in the HyperWorks installation directory to figure out how to build it into my own extension. This article describes one way to do it, based on what I found.

---

## Step 1: Create the Widgets

First, create a namespace to hold the selection for future usage

```tcl
namespace eval ::demo_selector {
    variable selected_ids {}
}
```

The control itself is assembled from these widgets:

| Widget | Role |
| :--- | :--- |
| `hmtk::entityselector` | The selector which owns the selection logic. |
| `hwctx::guidebar` | The guide bar that visually hosts the selector. |
| `hwtk::button` | A dummy button that shows the current entity type and count. |
| `hwtk::buttonbar` | Advanced / Reset / Apply / OK / Cancel |

The widget creation, configuration, and layout code below all goes into one procedure, 
`::demo_selector::launch` (see Step 5). 
The helper and callback procedures are defined separately.

```tcl
set dialog         [hwtk::dialog .dialog -title "Demo Selector"]
set recess         [$dialog recess]
set frame_cell     [hwtk::frame $recess.frame_cell]
set guidebar       [hwctx::guidebar $frame_cell.guidebar -fillet 2]
set button         [hwtk::button $frame_cell.button]
set buttonbar      [hwtk::buttonbar [winfo toplevel $recess].buttonbar -showseparator 0]
set entityselector [hmtk::entityselector $frame_cell.entityselector \
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
    -acceptcommand      [list ::demo_selector::on_accept $frame_cell.entityselector $buttonbar $button] \
    -cancelcommand      [list ::demo_selector::on_cancel $frame_cell.entityselector $buttonbar $button]
]
```

The button bar is created directly on the dialog's toplevel window rather than inside `frame_cell`, 
so it can later float over the cell with `place` (see Step 3).

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
grid $frame_cell -row 0 -column 0 -sticky ew -padx 4 -pady 4
grid $guidebar   -row 0 -column 0 -sticky ew

grid rowconfigure    $recess 0 -weight 1
grid columnconfigure $recess 0 -weight 1
grid columnconfigure $frame_cell 0 -weight 1
```

Gridding `frame_cell` into the dialog is ordinary layout. 
The interesting part is what happens inside `frame_cell`:

- `$guidebar` is gridded at row 0, column 0.
- `$button` is gridded into the same cell by `_update_dummy_button` (see Step 3). 
  Because it is gridded later, it covers the guide bar. 
  At rest, the user only sees the dummy button.
- `$buttonbar` is **not** laid out here at all. 
  It is placed over the cell only when the user starts selecting (see Step 3).
- The `-weight 1` setting on `$frame_cell` 
  lets the guide bar and the dummy button stretch to the full width of the cell.

---

## Step 3: Configure the Dummy Button

The dummy button's label shows the entity type and the count. 
The helper below grids the button into the cell and updates the label. 
It is called once at startup (see Step 5), and again after Accept or Cancel.

```tcl
proc ::demo_selector::_update_dummy_button {button entitytype {count 0}} {
    grid $button -row 0 -column 0 -sticky nsew
    $button configure -text "$count $entitytype"
}
```

Clicking the dummy button starts a selection session.

```tcl
$button configure \
    -command [list ::demo_selector::activate_selector $frame_cell $button $buttonbar $entityselector]
```

```tcl
proc ::demo_selector::activate_selector {frame_cell button buttonbar entityselector} {
    variable selected_ids
    set selected_ids [$entityselector ExecSelectionCommand GetSelectionIds]
    set cell_width  [winfo width $frame_cell]
    set cell_height [winfo height $frame_cell]

    grid forget $button
    place $buttonbar -in $frame_cell -anchor ne -x $cell_width -y $cell_height
    raise $buttonbar
    $entityselector SetActive
    $entityselector UpdateButtonWidth
}
```

1. Save the current selection to `selected_ids`, so that Cancel can restore it later.
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
    -image "toolbarMoreOptionsStrip-16.png" \
    -indicator hide \
    -help "Advanced Selection" \
    -command [list $entityselector OpenAdvancedSelection]

$buttonbar add a_reset \
    -image "toolbarActionResetStrip-16.png" \
    -indicator hide \
    -help "Reset" \
    -command [list $entityselector ExecSelectionCommand Clear]

$buttonbar add a_accept \
    -image "toolbarActionOKStrip-16.png" \
    -indicator hide \
    -help "Ok" \
    -command [list ::demo_selector::on_accept $entityselector $buttonbar $button]

$buttonbar add a_cancel \
    -image "toolbarActionCancelStrip-16.png" \
    -indicator hide \
    -help "Cancel" \
    -command [list ::demo_selector::on_cancel $entityselector $buttonbar $button]
```

Advanced Selection and Reset need no extra code. 
Accept and Cancel have to switch the UI back to the dummy button, 
so they call our own procedures.

### Accept

```tcl
proc ::demo_selector::on_accept {entityselector buttonbar button} {
    set ids        [$entityselector ExecSelectionCommand GetSelectionIds]
    set entitytype [$entityselector GetEntityType]
    $entityselector SetInactive
    place forget $buttonbar
    ::demo_selector::_update_dummy_button $button $entitytype [llength $ids]
    raise $button
}
```

Read the selection, deactivate the selector, remove the button bar, 
and bring the dummy button back with the new count as its label.

### Cancel

```tcl
proc ::demo_selector::on_cancel {entityselector buttonbar button} {
    variable selected_ids
    $entityselector ExecSelectionCommand Clear
    if {[llength $selected_ids]} {
        $entityselector ExecSelectionCommand SelectByAdvanced "by id" $selected_ids
    }
    set entitytype [$entityselector GetEntityType]
    $entityselector SetInactive
    place forget $buttonbar
    ::demo_selector::_update_dummy_button $button $entitytype [llength $selected_ids]
    raise $button
}
```

The selector has no built-in "undo" for a selection session. 
So Cancel clears whatever was picked and re-selects the ids saved at the start, 
using `SelectByAdvanced "by id"`. 
The rest is identical to Accept, except that the count comes from `selected_ids`.

---

## Step 5: Show the Dialog

At the end of `::demo_selector::launch`, initialize the dummy button, then post the dialog:

```tcl
::demo_selector::_update_dummy_button $button "Elements"

$dialog post
```

Putting it together, `launch` has this shape:

```tcl
proc ::demo_selector::launch {} {
    catch { destroy .dialog }

    # ---- create ----             (Step 1)
    # ---- configure commands ---- (Steps 3 and 4)
    # ---- layout ----             (Step 2)

    ::demo_selector::_update_dummy_button $button "Elements"

    $dialog post
}
```

Run `::demo_selector::launch`, click the button, pick some elements in the graphics area, 
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
