[RU](#русский) | [EN](#english)

# English
# ΣMACs v0.α.0626 — Full Specification

**AEGIS Versioning: 0.α.0626 — announce**

---

## 1. Overview

**ΣMACs** (pronounced "Sigma-macs") is a text editor for engineers, mathematicians, and hackers.

- **Σ** — the summation sign in mathematics, the sign of totality in philosophy. An editor that sums everything.
- **MACs** — MACroS. The classical Emacs paradigm: the editor as a set of macros.

Unlike GNU Emacs, ΣMACs contains no built-in Elisp. Instead, it provides an **APL REPL in the minibuffer** and a subset of APL functions. Everything else is implemented by the user.

**Philosophy:** Happy Hacking.

---

## 2. Sheer Power

***You open the editor.*** Black screen. Blinking cursor. No menus, no toolbars.

***You press `M-x` (Alt+X).*** The minibuffer command line appears at the bottom. You type:

```apl
'TODO'⌕⍵
```

And instantly you get a list of every occurrence of "TODO" across all open buffers. Because ⌕ is a built-in APL operator running at hardware speed.

***You press C-x C-f.*** The editor loads a file into a buffer. A 100 MB file? That's one read operation into memory and one ⍴ to get the dimensions. It takes exactly as long as the disk needs to read the data.

***You press C-/ (undo).*** The editor restores the previous buffer snapshot. 10,000 steps back? That's history[¯10⊢⍴history]. Instant.

**Speed.** Assembler core + APL array processing = latency between keypress and character appearing on screen: 0 microseconds. Physically impossible to go faster.

**Power.** APL allows any text operation to be expressed in a single line. These aren't Elisp macros — this is pure array mathematics.

**Barrier to entry.** You must know APL. But once you do — you are the architect of reality inside ΣMACs.

## 3. Architecture

### 3.1 Layers
|   Layer   |   Language  |                                   Functions                                   |
|:----------|:------------|:------------------------------------------------------------------------------|
| Core      | Assembler   | Buffer management, undo/redo, kill-ring, keyboard events, allocator           |
| APL       | APL         | Buffer representation as APL array, minibuffer command execution              |
| Rendering | C + asm     | Three modes: ANSI terminal (primary), Xlib window, framebuffer               |

### 3.2 Rendering Modes

| Priority       |     Mode      |          Technology          |                   Where it works                    |
|:---------------|:--------------|:-----------------------------|:----------------------------------------------------|
| 1 (primary)    | ANSI terminal | ANSI codes written to stdout | Local terminal, SSH, tmux, virtual console          |
| 2 (secondary)  | GUI           | Xlib + MIT-SHM               | Local X server                                      |
| 3 (tertiary)   | Framebuffer   | /dev/fb0 + mmap              | Virtual console without X11                         |

The primary mode is the ANSI terminal. This is the equivalent of `emacs -nw`. It works everywhere: local terminal, SSH, tmux, screen, virtual console. No dependencies beyond stdout and stdin. This is exactly how Emacs works with `-nw`, and this will be the primary mode of ΣMACs.

### 3.3 Assembler Core
The ΣMACs core is written in assembler. The APL interpreter is called as a subsystem for array operations. Everything else — buffer management, keyboard handling, undo/redo, kill-ring, allocator — is pure assembler.

Core components (all in assembler):
| Module               |                            Function                          |                      Task                       |
|:---------------------|:-------------------------------------------------------------|:------------------------------------------------|
| Allocator            | alloc(size) → ptr                                            | Ring buffer. Head advance. O(1).                |
| Keyboard dispatcher  | kbd_dispatch(scancode) → command                             | Interrupt table: scancode → handler             |
| Buffer manager       | buf_create(), buf_switch(id), buf_kill(id)                   | Doubly-linked list of buffers in ring allocator |
| Undo/Redo            | undo_push(snapshot), undo_pop() → snapshot                   | Snapshot stack. Each snapshot is a buffer copy  |
| Kill Ring            | kill_ring_push(region), kill_ring_yank() → region            | Ring buffer of 60 entries                       |
| ANSI renderer        | ansi_putchar(ch, attr), ansi_clear_line(n), ansi_scroll_up() | Escape code output to stdout                    |
| X11 renderer         | x11_putchar(ch, attr, x, y)                                  | Write to MIT-SHM buffer                         |
| FB renderer          | fb_putchar(ch, attr, offset)                                 | Write to mmap framebuffer                       |

Main loop pseudocode (assembler):
```asm
; ΣMACs main loop
main_loop:
    call    kbd_get_event           ; get event (scancode or signal)
    cmp     eax, EV_KEYPRESS        ; key pressed?
    je      handle_keypress
    cmp     eax, EV_TIMER           ; timer (autosave)?
    je      handle_timer
    cmp     eax, EV_SIGNAL          ; OS signal?
    je      handle_signal
    jmp     main_loop

handle_keypress:
    call    kbd_dispatch            ; scancode → command
    call    execute_command         ; execute command on buffer
    call    render_dirty            ; redraw changed lines
    jmp     main_loop
```

### 3.4 Ring Allocator (Assembler)

Pseudocode:
```asm
; alloc(size) → ptr
; Allocates size bytes from the ring buffer. O(1).
alloc:
    mov     eax, [head]             ; current head
    add     eax, ecx                ; eax = head + size
    cmp     eax, [tail]             ; caught up with tail?
    jae     alloc_fail              ; yes — out of space
    mov     [head], eax             ; advance head
    sub     eax, ecx                ; return pointer to block start
    ret
alloc_fail:
    xor     eax, eax                ; return 0
    ret

; free(ptr, size)
; Frees a block. O(1), only if block is at tail.
free:
    cmp     eax, [tail]             ; is this the tail?
    jne     free_skip               ; no — ignore
    add     eax, edx                ; eax = ptr + size
    mov     [tail], eax             ; advance tail
free\_skip:
    ret
```

### 3.5 Undo/Redo (Assembler + APL)
Undo/Redo is implemented as a stack of buffer snapshots. A snapshot is a full copy of the APL array ⍵. Saving and restoring are done in assembler via calls to APL functions.

Pseudocode:

```asm
; undo_push(buffer)
; Saves buffer snapshot to undo stack
undo_push:
    push    rdi                     ; save buffer pointer
    call    apl_snapshot            ; APL: snapshot ← ⊂⍵
    mov     rdi, rax                ; pointer to snapshot
    call    stack_push              ; assembler: push to stack
    pop     rdi
    ret

; undo_pop(buffer)
; Restores buffer from undo stack
undo_pop:
    call    stack_pop               ; assembler: pop from stack
    test    rax, rax
    jz      undo_empty              ; stack empty
    mov     rdi, rax                ; pointer to snapshot
    call    apl_restore             ; APL: ⍵ ← ⊃snapshot
    ret
undo_empty:
    ret
```

Stack depth: 10,000 operations.

### 3.6 Minibuffer and APL REPL

The minibuffer is a single-line window at the bottom of the screen. Pressing M-x activates the minibuffer, and the user enters an APL expression. Pressing Enter executes the expression against the current buffer ⍵.

Special minibuffer variables:

| Variable  |             Meaning                |
|:----------|:-----------------------------------|
| ⍵         | Current buffer (character matrix) |
| curr      | Cursor position (row, column)      |
| mark      | Mark position                      |
| kill_ring | Ring of killed strings             |
| history   | Undo stack                         |
| path      | Current file path                  |

The Kill Ring is the classic Emacs mechanism for storing deleted text. Not just a clipboard — a ring buffer of the 60 most recent deletions.

When you delete text with C-w (kill-region), C-k (kill-line), M-d (kill-word) — it enters the kill-ring. It doesn't overwrite the previous entry; it appends.

When you paste with C-y (yank) — it inserts the last killed chunk. Pressing M-y (yank-pop) cycles through the previous ones.

### 3.7 Zero-Cost FFI (for GUI mode)
GUI (Xlib) and the core communicate via shared memory. The APL array serves as the text buffer.

```text
┌─────────────────────────────────────────────────────────┐
│                   Shared memory                         │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │ apl_array* buf  │    │ struct {                    │ │
│  │ (current buffer)│    │   int dirty_from;           │ │
│  │                 │    │   int dirty_to;             │ │
│  │ char** data     │    │ } buffer_state;             │ │
│  │ int rows, cols  │    └─────────────────────────────┘ │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

## 4. Rendering Modes

### 4.1 Mode 1: ANSI Terminal (Primary)

The renderer sends ANSI escape codes to stdout. No libraries. Pure byte stream.

- Cursor positioning: ```ESC [ row ; col H```
- Clear line: ```ESC [ 2K```
- Text colour: ```ESC [ 3? m``` (30-37 for dark, 90-97 for bright)
- Background colour: ```ESC \[ 4? m```
- Scrolling: ```ESC [ r``` (set scroll region) + ```ESC [ 1S``` (scroll up)

### 4.2 Mode 2: GUI (Secondary)
Xlib window with MIT-SHM for fast rendering. Frames — geometry on the C side. Minibuffer — bottom line of the window.

### 4.3 Mode 3: Framebuffer (Tertiary)
Direct access to /dev/fb0 via mmap. Virtual console only.

## 5. Buffer as APL Array
```apl
⍵ ← (rows × cols) ⍴ characters
⍴⍵          ⍝ dimensions
⍵[3;]       ⍝ third row
⍵[;10]      ⍝ tenth column
↓⍵          ⍝ split into rows
⊃↓⍵         ⍝ reassemble
```

## 6. Happy Hacking Kit (HHK)

### 6.1 APL Subset
The distribution includes a subset of APL sufficient for text editing. The exact list of glyphs is finalised at the *β (brainware)* stage.

### 6.2 Classic Emacs Commands

Movement:
|           Command        | Hotkey |          Action         |
|:-------------------------|:-------|:------------------------|
| forward-char             | C-f    | Next character          |
| backward-char            | C-b    | Previous character      |
| next-line                | C-n    | Next line               |
| previous-line            | C-p    | Previous line           |
| forward-word             | M-f    | Next word               |
| backward-word            | M-b    | Previous word           |
| move-beginning-of-line   | C-a    | Beginning of line       |
| move-end-of-line         | C-e    | End of line             |
| forward-sentence         | M-e    | Next sentence           |
| backward-sentence        | M-a    | Previous sentence       |
| forward-paragraph        | M-}    | Next paragraph          |
| backward-paragraph       | M-{    | Previous paragraph      |
| beginning-of-buffer      | M-<    | Beginning of buffer     |
| end-of-buffer            | M->    | End of buffer           |
| goto-line                | M-g g  | Go to line              |
| goto-char                | M-g c  | Go to position          |

Editing:
|        Command         |   Hotkey    |    Action      |
|:-----------------------|:------------|:---------------|
| delete-char            | C-d         | Delete forward |
| backward-delete-char   | Backspace   | Delete backward|
| kill-line              | C-k         | Kill line      |
| kill-word              | M-d         | Kill word      |
| backward-kill-word     | M-Backspace | Kill word back |
| kill-region            | C-w         | Kill region    |
| copy-region-as-kill    | M-w         | Copy           |
| yank                   | C-y         | Paste          |
| yank-pop               | M-y         | Cycle yank     |
| undo                   | C-/         | Undo           |
| redo                   | C-M-/       | Redo           |
| newline                | Return      | New line       |
| open-line              | C-o         | Open line      |
| transpose-chars        | C-t         | Swap chars     |
| transpose-words        | M-t         | Swap words     |
| capitalize-word        | M-c         | Capitalise     |
| upcase-word            | M-u         | UPPERCASE      |
| downcase-word          | M-l         | lowercase      |

Search:
|      Command     |  Hotkey |     Action         |
|:-----------------|:--------|:-------------------|
| isearch-forward  | C-s     | Search forward     |
| isearch-backward | C-r     | Search backward    |
| query-replace    | M-%     | Replace with query |
| replace-string   | M-x     | Replace string     |

Files:
|          Command        | Hotkey  |   Action     |
|:------------------------|:--------|:-------------|
| find-file               | C-x C-f | Open         |
| save-buffer             | C-x C-s | Save         |
| write-file              | C-x C-w | Save as      |
| save-buffers-kill-emacs | C-x C-c | Exit         |

Windows:
|        Command       | Hotkey |           Action            |
|:---------------------|:-------|:----------------------------|
| split-window-below   | C-x 2  | Split horizontally          |
| split-window-right   | C-x 3  | Split vertically            |
| delete-window        | C-x 0  | Close window                |
| delete-other-windows | C-x 1  | Close all but current       |
| other-window         | C-x o  | Other window                |

Buffers:
|      Command      | Hotkey  |    Action    |
|:------------------|:--------|:-------------|
| switch-to-buffer  | C-x b   | Switch       |
| list-buffers      | C-x C-b | List         |
| kill-buffer       | C-x k   | Close        |

## 6.3 Minibuffer (M-x)

APL REPL. Examples:
```apl
'TODO'⌕⍵                         ⍝ find all TODO
'bar'@('foo'⍷,⍵)⊢,⍵             ⍝ replace foo → bar
⌽¨↓⍵                            ⍝ reverse lines
```

### 6.4 Extension API (sigma.h)
```c
// sigma.h — Zero-Cost FFI for extensions

typedef struct {
    char** data;
    int rows, cols;
    int dirty_from, dirty_to;
} apl_array;

extern apl_array* current_buffer;
int apl_eval(const char* expr, apl_array* buf);
extern void (*on_buffer_updated)(void);
```
## 7. Technical Highlights
### 7.1 Maximum File Size
Limited by RAM. With 64 GB — files up to ~50 GB without lag.

### 7.2 Instant Start
Time from Enter to first character — under 10 ms.

### 7.3 Binary Editing
The buffer is an APL array. It holds any data: text, binary files, pixel matrices.

```apl
⍵ ← ⎕MAP path                   ⍝ binary file into memory
⍵[⍳64;]                          ⍝ ELF header
'new'@(256+⍳3)⊢⍵                 ⍝ byte replacement
```

### 7.4 Multi-Buffers as Rank-3 Array
```apl
⍵ ← (buffers × rows × cols) ⍴ characters
⍵[1;;]  ⍝ first buffer
⍵[2;;]  ⍝ second buffer
```

### 7.5 Session Persistence
```apl
session ← ⊂(buffers)(history)(redo_stack)(kill_ring)(cursor_positions)
⎕NPUT '~/.sigmacs.dump' session
```
No parsing. Just an APL array dump.

## 8. Versions (AEGIS)
|   Version   |   Stage    |                              Description                               |
|:------------|:-----------|:-----------------------------------------------------------------------|
| 0.α.0626    | announce   | Specification published                                                |
| 0.β.XXXX    | brainware  | Core algorithms, FFI protocol, memory layout, APL glyph list           |
| 0.π.XXXX    | proto      | ANSI terminal version (primary mode)                                 |
| 0.δ.XXXX    | dev        | Optimisation, debugging, GUI mode                                      |
| 0.σ.XXXX    | sigma      | First stable release (ANSI + GUI)                                    |
| 0.φ.XXXX    | final      | Framebuffer mode, binary editing                                      |

## 9. HHK Distribution

```text
sigmacs/
├── src/
│   ├── kernel.s            # Core (assembler)
│   ├── alloc.s             # Ring allocator
│   ├── kbd.s               # Keyboard dispatcher
│   ├── render_ansi.s       # ANSI renderer
│   ├── render_fb.c         # Framebuffer renderer
│   ├── gui.c               # GUI (Xlib)
│   ├── sigma.h             # Zero-Cost FFI
│   └── Makefile
├── apl/
│   └── kernel.apl          # APL core
├── doc/
│   └── SPEC.md
└── examples/
    ├── highlight.apl
    ├── autocomplete.apl
    └── git.apl
```

Happy Hacking.

# Русский
# ΣMACs v0.α.0626 — Полная спецификация

**AEGIS Versioning: 0.α.0626 — announce**

---

## 1. Обзор

**ΣMACs** (произн. «Сигмакс») — текстовый редактор для инженеров, математиков и хакеров.

- **Σ** — знак суммы в математике, знак тотальности в философии. Редактор, суммирующий всё
- **MACs** — MACroS. Классическая парадигма Emacs: редактор как набор макросов

В отличие от GNU Emacs, ΣMACs не содержит встроенного языка Elisp. Вместо этого он предоставляет **APL-REPL в минибуфере** и подмножество APL-функций. Всё остальное пользователь реализует самостоятельно.

**Философия:** Happy Hacking.

---

## 2. Чистая мощь

***Ты открываешь редактор.*** Чёрный экран. Мигающий курсор. Никаких меню, никаких тулбаров.

***Ты нажимаешь `M-x` (Alt+X).*** Внизу появляется командная строка минибуфера. Ты вводишь:

```apl
'TODO'⌕⍵
```

И мгновенно получаешь список всех позиций, где встречается «TODO», во всех открытых буферах. Потому что ⌕ — это встроенный оператор APL, работающий на скорости железа.

***Ты нажимаешь C-x C-f.*** Редактор загружает файл в буфер. Файл размером 100 МБ? Это одна операция чтения в память и одна операция ⍴ для получения размерности. Занимает столько же времени, сколько диск читает данные.

***Ты нажимаешь C-/ (undo).*** Редактор восстанавливает предыдущий снимок буфера. 10 000 шагов назад? Это history[¯10⊢⍴history]. Мгновенно.

**Скорость.** Ассемблерное ядро + APL-обработка массивов = задержка между нажатием клавиши и появлением символа на экране — 0 микросекунд. Физически быстрее нельзя.

**Мощь.** APL позволяет выразить любую операцию над текстом в одной строке. Это не макросы Elisp, это чистая математика массивов.

Порог входа. Ты должен знать APL. Но когда ты это знаешь — ты архитектор реальности в мире ΣMACs.

## 3. Архитектура

### 3.1 Слои
|    Слой   |     Язык    |	                                 Функции                                  |
|:----------|:------------|:--------------------------------------------------------------------------|
| Ядро	    | Ассемблер   | Управление буферами, undo/redo, kill-ring, события клавиатуры, аллокатор  |
| APL	    | APL	      | Представление буфера как APL-массива, выполнение команд минибуфера        |
| Рендеринг | C + asm     | вставки	Три режима: ANSI-терминал (основной), Xlib-окно, фреймбуфер       |

### 3.2 Режимы рендеринга

| Приоритет	    |     Режим     |       	Технология	      |                   Где работает                    |
|:--------------|:--------------|:----------------------------|:--------------------------------------------------|
| 1 (основной)	| ANSI-терминал | Запись ANSI-кодов в stdout  | Локальный терминал, SSH, tmux, виртуальная консоль|
| 2 (вторичный)	| GUI	        | Xlib + MIT-SHM	          | Локальный X-сервер                                |
| 3 (третичный)	| Фреймбуфер	| /dev/fb0 + mmap	          | Виртуальная консоль без X11                       |

Основной режим — ANSI-терминал. Это аналог emacs -nw. Работает везде: локальный терминал, SSH, tmux, screen, виртуальная консоль. Никаких зависимостей кроме stdout и stdin. Именно так работает Emacs с -nw, и именно это будет основным режимом ΣMACs.

### 3.3 Ядро на ассемблере
Ядро ΣMACs написано на ассемблере. APL-интерпретатор вызывается как подсистема для операций над массивами. Всё остальное — управление буферами, обработка клавиатуры, undo/redo, kill-ring, аллокатор — чистый ассемблер.

Компоненты ядра (все на ассемблере):
| Модуль	           |                          Функция                             |                    Задача                        |
|:---------------------|:-------------------------------------------------------------|:-------------------------------------------------|
| Аллокатор            | alloc(size) → ptr	                                          | Кольцевой буфер. Сдвиг head. O(1).               |
| Диспетчер клавиатуры | kbd_dispatch(scancode) → command	                          | Таблица прерываний: scancode → обработчик        |
| Управление буферами  | buf_create(), buf_switch(id), buf_kill(id)	                  | Двусвязный список буферов в кольцевом аллокаторе |
| Undo/Redo	           | undo_push(snapshot), undo_pop() → snapshot	                  | Стек снимков. Каждый снимок — копия буфера       |
| Kill Ring	           | kill_ring_push(region), kill_ring_yank() → region	          | Кольцевой буфер на 60 записей                    |
| Рендерер ANSI	       | ansi_putchar(ch, attr), ansi_clear_line(n), ansi_scroll_up() |	Запись escape-кодов в stdout                     |
| Рендерер X11	       | x11_putchar(ch, attr, x, y)	                              | Запись в MIT-SHM-буфер                           |
| Рендерер FB	       | fb_putchar(ch, attr, offset)	                              | Запись в mmap-фреймбуфер                         |

Псевдокод основного цикла (ассемблер):
```asm
; Основной цикл ΣMACs
main_loop:
    call    kbd_get_event           ; получить событие (scancode или сигнал)
    cmp     eax, EV_KEYPRESS        ; нажата клавиша?
    je      handle_keypress
    cmp     eax, EV_TIMER           ; таймер (автосохранение)?
    je      handle_timer
    cmp     eax, EV_SIGNAL          ; сигнал ОС?
    je      handle_signal
    jmp     main_loop

handle_keypress:
    call    kbd_dispatch            ; scancode → команда
    call    execute_command         ; выполнить команду над буфером
    call    render_dirty            ; перерисовать изменённые строки
    jmp     main_loop
```

### 3.4 Кольцевой аллокатор (ассемблер)

Псевдокод:
```asm
; alloc(size) → ptr
; Выделяет size байт из кольцевого буфера. O(1).
alloc:
    mov     eax, [head]             ; текущая голова
    add     eax, ecx                ; eax = head + size
    cmp     eax, [tail]             ; догнали хвост?
    jae     alloc_fail              ; да — нет места
    mov     [head], eax             ; сдвинуть голову
    sub     eax, ecx                ; вернуть указатель на начало блока
    ret
alloc_fail:
    xor     eax, eax                ; вернуть 0
    ret

; free(ptr, size)
; Освобождает блок. O(1), только если блок в хвосте.
free:
    cmp     eax, [tail]             ; это хвост?
    jne     free_skip               ; нет — игнорируем
    add     eax, edx                ; eax = ptr + size
    mov     [tail], eax             ; сдвинуть хвост
free_skip:
    ret
```

### 3.5 Undo/Redo (ассемблер + APL)
Undo/Redo реализован как стек снимков буфера. Снимок — это полная копия APL-массива ⍵. Сохранение и восстановление делаются на ассемблере через вызов APL-функций.

Псевдокод:

```asm
; undo_push(buffer)
; Сохраняет снимок буфера в стек undo
undo_push:
    push    rdi                     ; сохранить указатель на буфер
    call    apl_snapshot            ; APL: snapshot ← ⊂⍵
    mov     rdi, rax                ; указатель на снимок
    call    stack_push              ; ассемблер: положить в стек
    pop     rdi
    ret

; undo_pop(buffer)
; Восстанавливает буфер из стека undo
undo_pop:
    call    stack_pop               ; ассемблер: взять из стека
    test    rax, rax
    jz      undo_empty              ; стек пуст
    mov     rdi, rax                ; указатель на снимок
    call    apl_restore             ; APL: ⍵ ← ⊃snapshot
    ret
undo_empty:
    ret
```

Глубина стека: 10 000 операций.

### 3.6 Минибуфер и APL-REPL

Минибуфер — это однострочное окно внизу экрана. При нажатии M-x минибуфер активируется, и пользователь вводит APL-выражение. При нажатии Enter выражение выполняется над текущим буфером ⍵.

Специальные переменные минибуфера:

| Переменная |            Значение              |
|:-----------|:---------------------------------|
| ⍵	        | Текущий буфер (матрица символов) |
| curr	    | Позиция курсора (строка, колонка) |
| mark	    | Позиция маркера                   |
| kill_ring | Кольцо убитых строк               |
| history	| Стек undo                         |
| path	    | Путь к текущему файлу             |

Kill Ring — это классический механизм Emacs для хранения удалённого текста. Не просто буфер обмена, а кольцевой буфер на 60 последних удалений.

Когда ты удаляешь текст через C-w (kill-region), C-k (kill-line), M-d (kill-word) — он попадает в kill-ring. Не затирает предыдущий, а добавляется.

Когда вставляешь через C-y (yank) — вставляет последний убитый кусок. Если нажать M-y (yank-pop) — циклически перебирает предыдущие.

### 3.7 Zero-Cost FFI (для GUI-режима)
GUI (Xlib) и ядро общаются через разделяемую память. APL-массив является текстовым буфером.

```text
┌─────────────────────────────────────────────────────────┐
│                   Разделяемая память                    │
│                                                         │
│  ┌─────────────────┐    ┌─────────────────────────────┐ │
│  │ apl_array* buf  │    │ struct {                    │ │
│  │ (текущий буфер) │    │   int dirty_from;           │ │
│  │                 │    │   int dirty_to;             │ │
│  │ char** data     │    │ } buffer_state;             │ │
│  │ int rows, cols  │    └─────────────────────────────┘ │
│  └─────────────────┘                                    │
└─────────────────────────────────────────────────────────┘
```

## 4. Режимы рендеринга

### 4.1 Режим 1: ANSI-терминал (основной)

Рендерер отправляет ANSI-escape-коды в stdout. Никаких библиотек. Чистый поток байтов.

- Позиционирование курсора: ```ESC [ row ; col H```
- Очистка строки: ```ESC [ 2K```
- Цвет текста: ```ESC [ 3? m``` (30-37 для тёмных, 90-97 для ярких)
- Цвет фона: ```ESC [ 4? m```
- Скроллинг: ```ESC [ r``` (установить регион скроллинга) + ```ESC [ 1S``` (скролл вверх)

### 4.2 Режим 2: GUI (вторичный)
Xlib-окно с MIT-SHM для быстрой отрисовки. Фреймы — геометрия на стороне C. Минибуфер — нижняя строка окна.

### 4.3 Режим 3: Фреймбуфер (третичный)
Прямой доступ к /dev/fb0 через mmap. Только в виртуальной консоли.

## 5. Буфер как APL-массив
```apl
⍵ ← (строки × колонки) ⍴ символы
⍴⍵          ⍝ размерность
⍵[3;]       ⍝ третья строка
⍵[;10]      ⍝ десятая колонка
↓⍵          ⍝ разбить на строки
⊃↓⍵         ⍝ собрать обратно
```

## 6. Happy Hacking Kit (HHK)

### 6.1 Подмножество APL
В поставке — подмножество APL, достаточное для редактирования текста. Точный список глифов фиксируется в *стадии β (brainware)*.

### 6.2 Классические команды Emacs

Перемещение:
|           Команда      | Хоткей |	       Действие       |
|:-----------------------|:-------|:----------------------|
| forward-char	         | C-f	  | Следующий символ      |
| backward-char	         | C-b	  | Предыдущий символ     |
| next-line	             | C-n	  | Следующая строка      |
| previous-line	         | C-p	  | Предыдущая строка     |
| forward-word	         | M-f	  | Следующее слово       |
| backward-word	         | M-b	  | Предыдущее слово      |
| move-beginning-of-line | C-a	  | Начало строки         |
| move-end-of-line	     | C-e	  | Конец строки          |
| forward-sentence	     | M-e	  | Следующее предложение |
| backward-sentence	     | M-a	  | Предыдущее предложение|
| forward-paragraph	     | M-}	  | Следующий параграф    |
| backward-paragraph	 | M-{	  | Предыдущий параграф   |
| beginning-of-buffer	 | M-<	  | Начало буфера         |
| end-of-buffer	         | M->	  | Конец буфера          |
| goto-line	             | M-g g  | Перейти на строку     |
| goto-char	             | M-g c  | Перейти на позицию    |

Редактирование:
|        Команда       |  Хоткей	 |  Действие
|:---------------------|:------------|:------------------|
| delete-char	       | C-d	     | Удалить вперёд    |
| backward-delete-char | Backspace	 | Удалить назад     |
| kill-line	           | C-k	     | Убить строку      |
| kill-word	           | M-d	     | Убить слово       |
| backward-kill-word   | M-Backspace | Убить слово назад |
| kill-region	       | C-w  	     | Убить регион      |
| copy-region-as-kill  | M-w	     | Копировать        |
| yank	               | C-y	     | Вставить          |
| yank-pop	           | M-y	     | Циклический yank  |
| undo	               | C-/	     | Отменить          |
| redo	               | C-M-/   	 | Повторить         |
| newline	           | Return	     | Новая строка      |
| open-line	           | C-o	     | Открыть строку    |
| transpose-chars	   | C-t	     | Поменять символы  |
| transpose-words	   | M-t	     | Поменять слова    |
| capitalize-word	   | M-c	     | Заглавная         |
| upcase-word	       | M-u	     | ВЕРХНИЙ           |
| downcase-word	       | M-l	     | нижний            |

Поиск:
|      Команда     |  Хоткей |	Действие          |
|:-----------------|:--------|:-------------------|
| isearch-forward  | C-s	 | Поиск вперёд       |
| isearch-backward | C-r	 | Поиск назад        |
| query-replace	   | M-%	 | Замена с запросом  |
| replace-string   | M-x     | Замена строки      |

Файлы:
|         Команда    	 | Хоткей  | Действие     |
|:-----------------------|:--------|:-------------|
|find-file	             | C-x C-f | Открыть      |
|save-buffer             | C-x C-s | Сохранить    |
|write-file	             | C-x C-w | Сохранить как|
|save-buffers-kill-emacs | C-x C-c | Выход        |

Окна:
|        Команда	  | Хоткей |	     Действие          |
|:--------------------|:-------|:--------------------------|
|split-window-below   | C-x 2  | Разделить горизонтально   |
|split-window-right	  | C-x 3  | Разделить вертикально     |
|delete-window	      | C-x 0  | Закрыть окно              |
|delete-other-windows |	C-x 1  | Закрыть все кроме текущего|
|other-window	      | C-x o  | Другое окно               |

Буферы:
|      Команда	  | Хоткей	|   Действие  |  
|:----------------|:--------|:------------|
|switch-to-buffer |	C-x b	| Переключить |
|list-buffers	  | C-x C-b	| Список      |
|kill-buffer	  | C-x k	| Закрыть     |

## 6.3 Минибуфер (M-x)

APL REPL. Примеры:
```apl
'TODO'⌕⍵                         ⍝ найти все TODO
'bar'@('foo'⍷,⍵)⊢,⍵             ⍝ замена foo → bar
⌽¨↓⍵                            ⍝ реверс строк
```

### 6.4 API для расширений (sigma.h)
```c
// sigma.h — Zero-Cost FFI для расширений

typedef struct {
    char** data;
    int rows, cols;
    int dirty_from, dirty_to;
} apl_array;

extern apl_array* current_buffer;
int apl_eval(const char* expr, apl_array* buf);
extern void (*on_buffer_updated)(void);
```
## 7. Технические фишки
### 7.1 Максимальный размер файла
Ограничен RAM. При 64 ГБ — файлы до ~50 ГБ без задержек.

### 7.2 Мгновенный старт
Время от Enter до первого символа — менее 10 мс.

### 7.3 Двоичное редактирование
Буфер — APL-массив. Содержит любые данные: текст, бинарные файлы, матрицы пикселей.

```apl
⍵ ← ⎕MAP path                   ⍝ бинарный файл в память
⍵[⍳64;]                          ⍝ заголовок ELF
'new'@(256+⍳3)⊢⍵                 ⍝ замена байт
```
### 7.4 Мульти-буферы как массив 3-го ранга
```apl
⍵ ← (буферы × строки × колонки) ⍴ символы
⍵[1;;]  ⍝ первый буфер
⍵[2;;]  ⍝ второй буфер
```

### 7.5 Сохранение сессии
```apl
session ← ⊂(buffers)(history)(redo_stack)(kill_ring)(cursor_positions)
⎕NPUT '~/.sigmacs.dump' session
```
Никакого парсинга. Только дамп APL-массива.

## 8. Версии (AEGIS)
|   Версия	|   Стадия	|                            Описание                           |
|:----------|:----------|:--------------------------------------------------------------|
| 0.α.0626	| announce	| Спецификация опубликована                                     |
| 0.β.XXXX	| brainware	| Алгоритмы ядра, протокол FFI, схема памяти, список APL-глифов |
| 0.π.XXXX	| proto	    | ANSI-терминальная версия (основной режим)                     |
| 0.δ.XXXX	| dev	    | Оптимизация, отладка, GUI-режим                               |
| 0.σ.XXXX	| sigma	    | Первый стабильный релиз (ANSI + GUI)                          |
| 0.φ.XXXX	| final	    | Фреймбуферный режим, двоичное редактирование                  |

## 9. Поставка HHK

```text
sigmacs/
├── src/
│   ├── kernel.s            # Ядро (ассемблер)
│   ├── alloc.s             # Кольцевой аллокатор
│   ├── kbd.s               # Диспетчер клавиатуры
│   ├── render_ansi.s       # ANSI-рендерер
│   ├── render_fb.c         # Фреймбуферный рендерер
│   ├── gui.c               # GUI (Xlib)
│   ├── sigma.h             # Zero-Cost FFI
│   └── Makefile
├── apl/
│   └── kernel.apl          # APL-ядро
├── doc/
│   └── SPEC.md
└── examples/
    ├── highlight.apl
    ├── autocomplete.apl
    └── git.apl
```

Happy Hacking.