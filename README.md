# editbox
EditBox is a plugin for the [Volatility Framework](https://github.com/volatilityfoundation/volatility). It extracts the text from Windows Edit controls, that is, textboxes as generated by Windows Common Controls.

## How does it work?
Edit controls are created by a call to [CreateWindowEx](https://msdn.microsoft.com/en-us/library/windows/desktop/ms632680%28v=vs.85%29.aspx). As such, they are technically windows. Their class name looks something like this:
`6.0.7601.17514!Edit`

The part before the exclamation mark (`6.0.7601.17514`) is the version of the [Common Controls](http://msdn.microsoft.com/en-us/library/windows/desktop/bb775493%28v=vs.85%29.aspx) library being used. Owing to Windows Side-by-Side execution (WinSxS), there can be many different versions of the Common Controls library in use at one time.

Edit controls, like Buttons and ListBoxes, are obviously special kinds of windows. As such, they have extra data after the window definition which differentiates them from a generic window. It works like this:

* The window has a property called cbWndExtra. This "[s]pecifies the number of extra bytes to allocate following the window instance."
* Typically, these bytes are a memory address to a structure.
* The structure holds the extra bytes that distinguish a specific window, for example an Edit control, from a generic window.
* The EditBox plugin understands what (some of) the extra bytes mean, so, by parsing the structure, is able to get information about the Edit control.

## What information can it get?
1. **nChars**: The number of characters in the Edit control.
2. **selStart**: Selection Start. If text within the control is selected, this is the 0-based index of the first selected character.
3. **selEnd**: Selection End. If text within the control is selected, this is the 0-based index of the last selected character.
4. **isPwdControl**: Whether the EditBox is a password control, that is, characters are masked by another character.
5. **pwdChar**: If a password control, the masking character. For example: *.
6. And of course, the actual text of the Edit control.

When memory addresses are output, they are always virtual. Following them, in square brackets is the physical address. That is, the offset at which the data can be found when searching the raw dump. For example:

`address-of cbwndExtra: 0xfffff900c065d958 [0x6203958]`
## How do I use it?
### Let Volatility know you're using an additional plugin.
```
$ python vol.py --plugins=/folder/to/editboxplugin --filename=memory.dmp --profile=Win7SP1x64 editbox
```
### A Patch
EditBox calculates the position of the `WndExtra` bytes from the `tagWND` structure. At the time of writing, the defintion of the size of the tagWND structure for Windows XP is wrong so a patch is needed. (A pull request has been made so hopefully this won't be the case for long.)
```
--- xp.py.old	2015-03-21 13:52:20.264852000 +0000
+++ xp.py	2015-03-21 13:55:06.892852000 +0000
@@ -157,7 +157,7 @@
             'right' : [ 0x8, ['long']],
             'bottom' : [ 0xc, ['long']],
             }],
-            'tagWND' : [ 0x90, {
+            'tagWND' : [ 0xA4, {
             'head' : [ 0x0, ['_THRDESKHEAD']],
             'ExStyle' : [ 0x1c, ['unsigned long']],
             'style' : [ 0x20, ['unsigned long']],
```
### Switches
#### --dump-dir/-D
The text of an Edit control can be long. For example, the contents of a Notepad window. Using this switch, EditBox will dump the text to a file in the specified folder. The file will be named as per the MD5 of the text from the Edit control.
#### --pid/-p
EditBox will only work on processes with the specified Process ID.
#### --minimal/-m
Minimal output. Typically, EditBox is quite verbose. With this switch, EditBox will only output the PID, Process Image Name, and the extracted text. (Does not apply to experimental output; see below.)
#### --experimental/-e
Enables EditBox's experimental options. Currently EditBox is also capable of parsing some data from ListBox and ComboBox controls. This switch will dump this data **as well as** the EditBox data.
#### --experimental-only/-E
EditBox will **only** carry out the experimental options. Currently that is ListBoxes and ComboBoxes. Edit controls will be ignored.

## Sample Output
```
Wnd context          : 1\WinSta0\Default
pointer-to tagWND    : 0xfffff900c065d7a0 [0x62037a0]
pid                  : 1748
imageFileName        : explorer.exe
wow64                : No
atom_class           : 6.0.7601.17514!Edit
address-of cbwndExtra: 0xfffff900c065d958 [0x6203958]
value-of cbwndExtra  : 8 (0x8)
address-of WndExtra  : 0xfffff900c065d998 [0x6203998]
value-of WndExtra    : 0x5ad76d0 [0x250c6d0]
pointer-to hBuf      : 0x5b2f910 [0x8bb5910]
hWnd                 : 0xa044a
parenthWnd           : 0x90408
nChars               : 6 (0x6)
selStart             : 6 (0x6)
selEnd               : 6 (0x6)
text_md5             : cc2a31715ee8a734ef20747a2edc2d22
isPwdControl         : Yes
pwdChar              : 0x25cf
monkey
```
## More information about the experimental options.
The experimental option will try and extract useful information from the following controls:

    ListBox
    ComboBox
### ListBox
The following information can be extracted from a ListBox control:

    caretPos: The 0-based offset of the item in the list which has focus.
    rowsVisible: The number of rows the ListBox shows before the user must scroll.
    firstVisibleRow: The 0-based offset of the first row which is visible.
    itemCount: The number of items contained in the list.
    stringsStart: The memory offset at which the strings making up the items in the list can be found.
    stringsLength: The number of bytes which make up the strings. Each character requires 2 bytes, and each string is null-terminated.
    strings: A comma-separated list of the strings from the ListBox.

#### Sample Output
```

```
### ComboBox
The following information can be extracted from a ComboBox control:

    handle-of combolbox: The handle of the ListBox which is a component of the ComboBox.
#### Sample Output
```
*******************************************************
*** Experimental **************************************
*******************************************************
Wnd context          : 1\WinSta0\Default
pointer-to tagWND    : 0xfffff900c061c4d0 [0x18ce44d0]
pid                  : 1748
process              : explorer.exe
wow64                : No
atom_class           : 6.0.7601.17514!Combobox
address-of cbwndExtra: 0xfffff900c061c5b8 [0x18ce45b8]
value-of cbwndExtra  : 8 (0x8)
address-of WndExtra  : 0xfffff900c061c5f8 [0x18ce45f8]
value-of WndExtra    : 0x342c770 [0xc0fd770]
handle-of combolbox  : 0x10162
```
