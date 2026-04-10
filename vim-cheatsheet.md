# Vim Cheatsheet

## 🧭 Modal Philosophy
Vim operates in modes. Remember which mode you're in!

| Mode | How to Enter | Cursor | Purpose |
| :--- | :--- | :--- | :--- |
| **Normal** | `Esc` | Block | Navigating, deleting, copying (Commands) |
| **Insert** | `i`, `a`, `o` | `\|` | Typing text |
| **Visual** | `v`, `V`, `Ctrl+v` | Highlight | Selecting blocks of text |
| **Command** | `:` | Bottom left | Saving, quitting, searching/replacing |

---

## 🏃‍♂️ Movement (Normal Mode) - *The "Grammar" of Vim*

### Basic Arrows (Learn these first, ditch the arrow keys!)
| Key | Action | Mnemonic |
| :--- | :--- | :--- |
| `h` | **Left** | (Leftmost key) |
| `j` | **Down** | *J*umps down |
| `k` | **Up** | *K*icks up |
| `l` | **Right** | **L**eft... no, just *Right* |

### Word Jumping
| Key | Action |
| :--- | :--- |
| `w` | Jump to start of **w**ord |
| `b` | Jump **b**ack to start of word |
| `e` | Jump to **e**nd of word |

### Screen Navigation
| Key | Action |
| :--- | :--- |
| `gg` | Go to **top** of file |
| `G` | Go to **bottom** of file |
| `42G` | Go to **line 42** |
| `Ctrl+u` | Page **Up** |
| `Ctrl+d` | Page **Down** |
| `zz` | Center screen on cursor |

### In-Line Navigation
| Key | Action |
| :--- | :--- |
| `0` | Go to **first** column |
| `^` | Go to **first non-blank** character |
| `$` | Go to **end** of line |

---

## ✍️ Switching to Insert Mode

| Key | Action |
| :--- | :--- |
| `i` | Insert **before** cursor |
| `a` | Insert **after** cursor |
| `I` | Insert at **beginning** of line |
| `A` | Insert at **end** of line |
| `o` | Open new line **below** |
| `O` | Open new line **above** |

---

## ✂️ Editing (The Operators)

In Vim, an action is `[Operator][Motion]`. *Example: `d2w` = Delete 2 Words.*

### Operators
| Key | Action | Mnemonic |
| :--- | :--- | :--- |
| `d` | **D**elete (Cut) | - |
| `y` | **Y**ank (Copy) | - |
| `c` | **C**hange (Delete + Insert Mode) | - |
| `> / <` | Indent / Outdent | - |
| `p` | **P**aste after cursor | - |
| `P` | **P**aste before cursor | - |

### Common Combos
| Combo | Action |
| :--- | :--- |
| `dd` | Delete whole line |
| `yy` | Yank whole line |
| `dw` | Delete word |
| `ci"` | **C**hange **I**nside `" "` (Powerful!) |
| `di(` | **D**elete **I**nside `( )` |
| `yG` | Yank from here to bottom of file |
| `d0` | Delete from cursor to start of line |
| `D` | Delete from cursor to end of line (same as `d$`) |
| `C` | Change from cursor to end of line (same as `c$`) |

---

## 🔁 Undo & Redo

| Key | Action |
| :--- | :--- |
| `u` | **U**ndo |
| `Ctrl+r` | **R**edo |

---

## 🎯 Visual Mode (Selecting Text)

| Key | Action |
| :--- | :--- |
| `v` | Character-wise selection |
| `V` | Line-wise selection |
| `Ctrl+v` | **Block** selection (Vertical column) |

*Tip: Once selected, press `d` to delete, `y` to yank, or `I` to insert on multiple lines at once!*

---

## 📂 File Management (Command Mode `:`)

| Command | Action |
| :--- | :--- |
| `:w` | **W**rite (Save) |
| `:q` | **Q**uit |
| `:wq` or `:x` | Save and Quit |
| `:q!` | Quit without saving (**Force**) |
| `:e filename` | **E**dit another file |
| `:w filename` | Save **as** new file |

---

## 🔍 Search & Replace

### Search
| Command | Action |
| :--- | :--- |
| `/pattern` | Search **forward** |
| `?pattern` | Search **backward** |
| `n` | Go to **n**ext match |
| `N` | Go to **p**revious match |
| `*` | Search for word **under cursor** forward |

### Find & Replace (The Syntax)
`:s/old/new/g` - Replace on current line  
`:%s/old/new/g` - Replace in **whole file**  
`:%s/old/new/gc` - Replace with **c**onfirmation prompt  

---

## ⚙️ Advanced Tricks

### Marks (Bookmarks)
| Command | Action |
| :--- | :--- |
| `ma` | Set mark **a** |
| `'a` | Jump to line of mark **a** |
| `` `a `` | Jump to exact position of mark **a** |

### Registers (Multiple Clipboards)
| Command | Action |
| :--- | :--- |
| `"ayy` | Yank line into register **a** |
| `"ap` | Paste from register **a** |
| `:reg` | View all registers |

### Splits & Tabs
| Command | Action |
| :--- | :--- |
| `:split` / `:sp` | Horizontal split |
| `:vsplit` / `:vsp` | Vertical split |
| `Ctrl+w` `w` | Switch between splits |
| `Ctrl+w` `q` | Close split |
| `:tabnew` | Open new tab |
| `gt` | Next tab |

---

## 🛠️ Cheat Sheet of the Cheat Sheet (Top 5 to Memorize First)

1. **`:wq`** - Save and quit. **`:q!`** - Abandon changes.
2. **`h j k l`** - Move. (Turn off arrow keys to force learning).
3. **`u`** - Undo. `Ctrl+r` - Redo.
4. **`dd`** - Delete line. `p` - Paste it back.
5. **`/search`** - Find text. `n` for next.