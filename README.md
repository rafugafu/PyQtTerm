# About  
This is a full, standalone terminal emulator inside a PyQt6 widget. It is a PyQt port of [PyTkTerm](https://github.com/rafugafu/pytkterm). Instead of a `Text` widget it keeps its own grid of styled cells (scrollback plus the live screen) and paints that grid itself.  
# Specifications  
## Constructor  
`Terminal(parent=None, command=None, endmessage=None, nocolor=False, font=monospace, fontsize=12, foreground=None, background=None)`  
- `parent`: the parent Qt widget.  
- `command`: a list of argv to run in place of the shell, for example `['ssh', 'host']`. Pass `None` to launch the default shell (`$SHELL` on Linux, `powershell.exe` on Windows).  
- `endmessage`: text shown once the child process exits. Once it is shown, the widget stops forwarding keypresses to the child (there is none left) and instead treats any keypress as a request to tear down the pty/process and emit `stopped`; the widget itself is not destroyed, only cleaned up, and stays visible showing its final contents unless something connected to `stopped` closes it. Pass `None` to skip the message and run that same teardown (emitting `stopped`) automatically as soon as the child process exits, with no keypress needed.  
- `nocolor`: when `True`, disables ANSI SGR color/style rendering and cursor color escapes, keeping only plain text with cursor movement and editing.  
- `font`: the font family name, as a string. Defaults to `'monospace'` (`'Courier'` on Windows). It is always used explicitly, for the plain text, the bold/italic variants, the cursor and the character cell size.  
- `fontsize`: the font size in points, as a number. Defaults to `12`. Used together with `font` in the same way.  
- `foreground`, `background`: the default text and background colors (anything `QColor` accepts). When omitted, they come from the widget's palette and follow it when it changes.  
## Terminal Capabilities  
- Spawns the child process on a real pty on Linux (via `pty.openpty`) or `winpty` on Windows, and keeps the pty's window size in sync with the widget's size.  
- Supports a VT100/xterm-compatible escape sequence subset: cursor movement (CUU/CUD/CUF/CUB/CUP/HVP/CNL/CPL/CHA/VPA), save/restore cursor (`ESC 7`/`ESC 8`, CSI `s`/`u`), erase in line/display (EL, ED, modes 0-3), insert/delete character and line (`@`, `P`, `L`, `M`), scroll up/down (`S`, `T`), erase character (`X`), repeat character (`b`), scrolling regions (DECSTBM), tab stops (HTS/TBC), device status report and device attributes queries, and full reset (`RIS`).  
- Supports 16-color, 256-color, and 24-bit truecolor SGR rendering (bold, italic, underline, strikethrough, blink, reverse video, concealed text) unless `nocolor` is set.  
- Supports the alternate screen buffer (modes 47/1047/1049), origin mode (DECOM), reverse screen (DECSCNM), and DECCOLM-style screen clearing.  
- Supports bracketed paste mode, focus-in/focus-out reporting, application cursor keys mode, and `modifyOtherKeys` for modified special keys.  
- Supports mouse reporting in X10, normal, button-motion, and any-motion tracking modes, with both legacy and SGR (1006) coordinate encoding, plus scroll-wheel-to-arrow-key translation outside mouse mode via alternate scroll. Holding Shift bypasses mouse reporting for every mouse button.  
- Supports OSC sequences for the clipboard (OSC 52), and for querying/setting the foreground color, background color, and cursor color (OSC 10/11/12/112).  
- Supports DECSCUSR cursor shape/blink selection (block, underline, bar; blinking or steady) and honors DECTCEM cursor visibility. The cursor is painted over the current cell and can take on any of the three shapes, follow the child process's requested color, and blink independently while still showing the character underneath it in block shape. It is hidden while the widget does not have focus.  
- Right-click opens a context menu with Copy, Paste, and Select All. Selections can be made with the mouse (drag, double-click for a word, triple-click for a line) and copied/pasted with the mouse (middle-click pastes) or `Ctrl+Shift+C`/`Ctrl+Shift+V`. `Ctrl+Shift+A` selects everything. Copying joins autowrapped lines back together and strips trailing spaces.  
- Sends text from input methods (composed characters, dead keys) to the child process.  
- Debounces widget resize events, then recomputes the character grid from the new pixel size and pushes the new size to the pty (`TIOCSWINSZ` plus `SIGWINCH`) so the child process is notified. In the primary screen, rows move between the screen and scrollback; in the alternate screen, each row is padded or trimmed to the new column count. Existing line content is never rewrapped to the new width.  
- Keeps at most 10000 lines of scrollback. The scroll bar is hidden; scroll with the mouse wheel. The view follows new output only while it is scrolled to the bottom.  
## Public Methods  
- `setcursortype(to='block')`: sets the cursor shape to `'block'`, `'underline'`, or `'bar'`.  
- `setDefaultColors(foreground=None, background=None)`: sets the default text and background colors. Pass `None` for either to go back to the widget's palette.  
- `restart()`: terminates the running child process, clears the screen, and starts a new one with the same `command`.  
- `_write(data)`: sends raw bytes to the child process's stdin through the pty (decoded to text first on Windows).  
## Signals  
- `processEnded`: emitted once when the child process exits.  
- `stopped`: emitted once the widget has fully torn down the process and its resources (skipped when restarting via `restart()`).  
- `outputProcessed`: emitted after each batch of child output has been processed and rendered.  
# Examples  
Basic:  
```python
import sys
from PyQt6.QtWidgets import QApplication
from pyqtterm import Terminal

app = QApplication(sys.argv)
term = Terminal()
term.show()

sys.exit(app.exec())
```  
With focus and process end handling:  
```python
import sys
from PyQt6.QtWidgets import QApplication
from pyqtterm import Terminal

# setup application
app = QApplication(sys.argv)

# make terminal widget and focus
term = Terminal()
term.setFocus()

# quit when the process has been torn down
term.stopped.connect(app.quit)

# show window
term.show()
sys.exit(app.exec())
```  
With window title and endmessage:  
```python
import sys
from PyQt6.QtWidgets import QApplication
from pyqtterm import Terminal

# setup application
app = QApplication(sys.argv)

# make terminal widget and focus
term = Terminal(endmessage='--- process finished, press any key to continue ---')
term.setWindowTitle('Terminal')
term.setFocus()

# quit when the process has been torn down
term.stopped.connect(app.quit)

# show window
term.show()
sys.exit(app.exec())
```  
# Standalone Use  
Running `python pyqtterm.py` opens a single terminal window running your default shell. Pass a command as arguments to run it instead, for example `python pyqtterm.py htop`. The program is only used if it can be found on `PATH`; otherwise the default shell is started. The application quits when the process ends.  
# Dependencies  
- Python 3 with [`PyQt6`](https://pypi.org/project/PyQt6/).  
- On Windows only: [`pywinpty`](https://pypi.org/project/pywinpty/) (imported as `winpty`), used in place of the standard library's `pty` module, which is Linux-only.  
- No other dependencies; everything else used (`os`, `platform`, `subprocess`, `codecs`, `sys`, `base64`, `re`, `queue`, `threading`, `time`, `traceback`, `unicodedata`, and, on Linux, `pty`/`fcntl`/`termios`/`struct`/`select`/`signal`) is part of the Python standard library.  
# To Use  
1. Install the [dependencies](#dependencies).  
2. Copy the PyQtTerm file into your program's directory. Then import `pyqtterm.Terminal` normally. You can use this anywhere a normal Qt widget can be used.  
