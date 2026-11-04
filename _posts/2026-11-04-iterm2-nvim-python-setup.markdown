---
layout: post
title:  "Setting up iTerm2 and Neovim for Python development"
date:   2026-11-04 01:00:00 +0000
categories: [python, nvim, iterm2]
---

Once again I've been setting up a new machine for development *sigh*. I never remember my setups, always having to sleuth for bits and bobs here and there on old machines, so I'm writing a guide to myself for next time. I'm also add basic Neovim and iTerm2 commands in here so a complete beginner can get setup.

I'm using [Neovim](https://neovim.io/) for Python development within [iTerm2](https://iterm2.com/) on macOS. I'm trying to keep a lightweight development environment with code navigation, type checking, linting, and formatting, without adding dependencies to individual Python projects.

Why this setup? Well it's very portable, I can use it across work and home, not needing any licenses, and it works across most OS' with a few snags on Windows.

The setup uses Neovim's built in Language Server Protocol (LSP) support, [Basedpyright](https://docs.basedpyright.com/) for type checking and navigation, and [Ruff](https://docs.astral.sh/ruff/) for linting and formatting.

# Installation

Using Homebrew on the global env:

```bash
brew install neovim basedpyright ruff ripgrep
brew install --cask iterm2
```

Neovim configuration is stored in `~/.config/nvim/init.vim` or `init.lua`. I'm using `init.vim`, with Lua embedded for the LSP configuration. If you are starting fresh probably just begin with lua, but my config has... evolved, or maybe mutated is a better descriptor. I typically use venv and uv for python projects but I'm trying to keep the local editor settings out of that with this setup.

# iTerm2

[iTerm2](https://iterm2.com/) is a replacement for the macOS Terminal. I mainly use it for splitting the terminal into panes, keeping Neovim in one column and Git, tests and other commands in another. I can also save custom pane layouts so I have a default I like.

| Shortcut | Action |
|---|---|
| `Cmd+D` | Split vertically |
| `Cmd+Shift+D` | Split horizontally |
| `Cmd+Option+Arrow` | Switch between panes |
| `Cmd+[` / `Cmd+]` | Previous / next pane |
| `Cmd+T` | New tab |
| `Cmd+W` | Close pane or tab |
| `Cmd+Shift+Enter` | Maximise / restore current pane |

To make new panes open in the same directory, go to ```Settings → Profiles → General```, select ```Advanced Configuration``` and set new split panes to reuse the previous session's directory.

I normally open two columns, the left column is split horizontally 2-4 times containing my panes for git, testing, linting checks, maybe snooping around some dirs, then my right column is vertically split 2-3 times with Neovim open in each one that I want to see some code in.

# Neovim 

## init.vim

My complete ```~/.config/nvim/init.vim``` configuration is below. It uses Vimscript for the basic editor settings and Lua for the language servers. No Neovim plugins are required this way.

```vim
set number
set tabstop=4
set shiftwidth=4
set expandtab
set autoindent
set smartindent
set clipboard=unnamedplus

syntax enable
filetype plugin indent on

autocmd BufRead,BufNewFile *.c,*.h set filetype=c

set signcolumn=yes
set hidden
set updatetime=300
set colorcolumn=80 "char length guide

set termguicolors "pretty
set ignorecase "makes searches case insensitive
set smartcase "switches to case sensitive if caps is used in search
set nowrap "stops long code lines from wrapping
autocmd FileType markdown,text setlocal wrap linebreak "allow some files to wrap




" =================================
" Python dev stuff (Neovim 0.11+)
" =================================

lua << EOF

-- Find project root, falling back to file directory
local root_markers = {
    "pyrightconfig.json",
    "pyproject.toml",
    "ruff.toml",
    ".ruff.toml",
    ".git",
}

local function project_root(bufnr, on_dir)
    local file = vim.api.nvim_buf_get_name(bufnr)
    local root = vim.fs.root(file, root_markers)

    if not root then
        root = vim.fn.fnamemodify(file, ":h")
    end

    on_dir(root)
end

-- Basedpyright: type checking, navigation, completion
vim.lsp.config("basedpyright", {
    cmd = { "basedpyright-langserver", "--stdio" },
    filetypes = { "python" },
    root_dir = project_root,
})

-- Ruff: linting and formatting
vim.lsp.config("ruff", {
    cmd = { "ruff", "server" },
    filetypes = { "python" },
    root_dir = project_root,
})

-- Enable Python language servers
vim.lsp.enable({ "basedpyright", "ruff" })

-- Editor shortcuts
vim.g.mapleader = " "

vim.keymap.set("n", "gd", vim.lsp.buf.definition,
    { desc = "Go to definition" })

vim.keymap.set("n", "gr", vim.lsp.buf.references,
    { desc = "Find references" })

vim.keymap.set("n", "K", vim.lsp.buf.hover,
    { desc = "Show documentation" })

vim.keymap.set("n", "<leader>rn", vim.lsp.buf.rename,
    { desc = "Rename symbol" })

vim.keymap.set("n", "<leader>ca", vim.lsp.buf.code_action,
    { desc = "Code actions" })

-- Format manually with Ruff (no formatting on save)
vim.keymap.set("n", "<leader>f", function()
    vim.lsp.buf.format({
        async = false,
        timeout_ms = 2000,
        filter = function(client)
            return client.name == "ruff"
        end,
    })
end, { desc = "Format with Ruff" })

-- Show diagnostics
vim.diagnostic.config({
    virtual_text = false,
    virtual_lines = false,
    signs = true,
    underline = true,
    update_in_insert = false,
    severity_sort = true,
})

-- Diagnostic navigation
vim.keymap.set("n", "[d", function()
    vim.diagnostic.jump({ count = -1 })
end, { desc = "Previous diagnostic" })

vim.keymap.set("n", "]d", function()
    vim.diagnostic.jump({ count = 1 })
end, { desc = "Next diagnostic" })

vim.keymap.set("n", "<leader>e", function()
    vim.diagnostic.open_float({
        scope = "line",
        border = "rounded",
        focusable = true,
        max_width = 85,
        wrap = true,
    })
end)

vim.keymap.set("n", "<leader>q", function()
    vim.diagnostic.setloclist({ open = true })
end)

-- Built-in autocomplete (Neovim 0.11+)
vim.opt.completeopt = { "menuone", "noselect", "popup" }

vim.api.nvim_create_autocmd("LspAttach", {
    callback = function(args)
        local client = vim.lsp.get_client_by_id(args.data.client_id)

        if client and client:supports_method(
            "textDocument/completion"
        ) then
            vim.lsp.completion.enable(
                true,
                client.id,
                args.buf,
                { autotrigger = true }
            )
        end
    end,
})

EOF

```

## Basic navigation

Neovim has several modes. `Esc` returns to Normal mode, `i` enters Insert mode, and `v` enters Visual mode. Remember: to quit mash Esc then :q, or :q! (Don't save changes), or :wq (save changes).

| Shortcut | Action |
|---|---|
| `h` / `j` / `k` / `l` | Left / down / up / right |
| `w` / `b` | Next / previous word |
| `0` / `$` | Start / end of line |
| `gg` / `G` | Start / end of file |
| `25G` | Go to line 25 |
| `Ctrl+D` / `Ctrl+U` | Half-page down / up |
| `%` | Matching bracket |
| `zz` | Centre current line |
| `Ctrl+O` | Previous jump location |
| `Ctrl+I` | Next jump location |

`Ctrl+O` is great after jumping to a function definition in another file.

## Editing

| Shortcut | Action |
|---|---|
| `i` / `a` | Insert before / after cursor |
| `o` / `O` | New line below / above |
| `dd` | Delete line |
| `yy` | Copy line |
| `p` | Paste |
| `u` / `Ctrl+R` | Undo / redo |
| `ciw` | Change current word |
| `di(` | Delete contents of parentheses |
| `ci"` | Change contents of quotes |
| `>>` / `<<` | Indent / unindent |
| `V` | Select entire lines |

Commands can be combined with counts, e.g. `5dd` deletes five lines.

## Files, buffers and windows

Neovim separates files, buffers and windows. A buffer contains an open file, while a window displays a buffer.

| Command | Action |
|---|---|
| `:e file.py` | Open file |
| `:Ex` | File explorer |
| `:ls` | List buffers |
| `:bnext` / `:bprevious` | Switch buffers |
| `:b#` | Alternate buffer |
| `:vsplit file.py` | Vertical split |
| `:split file.py` | Horizontal split |
| `Ctrl+W h/j/k/l` | Move between windows |
| `Ctrl+W q` | Close current window |
| `Ctrl+W o` | Close other windows |
| `:w` | Save |
| `:q` | Quit current window |
| `:wq` | Save and quit |
| `:qa` | Quit all windows |

For example, `:vsplit features.py` opens another Python file alongside the current one. But I don't split within neovim, I use iterm2 panes for that.


## Python language server

[Basedpyright](https://docs.basedpyright.com/) is a static type checker and language server based on Pyright. It provides code navigation, type checking, completion and symbol references without executing the code.

[Ruff](https://docs.astral.sh/ruff/) handles linting and formatting.

Both run independently of the Python project's dependencies. Existing project configurations are used when present.

The following shortcuts use Neovim's LSP API, with custom mappings for some commands.

| Shortcut | Action |
|---|---|
| `gd` | Go to definition |
| `gr` | Find references |
| `K` | Function documentation |
| `Space rn` | Rename symbol |
| `Space ca` | Code actions |
| `Space f` | Format with Ruff |
| `Ctrl+O` | Return after navigation |

`gd` is particularly useful for following function calls between Python modules. `Ctrl+O` returns to the previous location.

Renaming through LSP can update references across multiple files, unlike ordinary text replacement.

## Diagnostics

Basedpyright reports type errors, while Ruff reports linting issues.

For example:

```python
def calculate_mean(values: list[float]) -> float:
    return sum(values) / len(values)

result = calculate_mean("hello")
```

Basedpyright identifies the incorrect argument type without executing the function.

Inline diagnostics can extend beyond the terminal window. I prefer disabling inline messages and displaying the full diagnostic in a floating window when needed.

```lua
vim.diagnostic.config({
    virtual_text = false,
    virtual_lines = false,
    signs = true,
    underline = true,
    update_in_insert = false,
    severity_sort = true,
})
```

A shortcut to display the complete message:

```lua
vim.keymap.set("n", "<leader>e", function()
    vim.diagnostic.open_float({
        scope = "line",
        border = "rounded",
        focusable = true,
        max_width = 85,
        wrap = true,
    })
end)
```

Additional diagnostic shortcuts:

| Shortcut | Action |
|---|---|
| `]d` | Next diagnostic |
| `[d` | Previous diagnostic |
| `Space e` | Full diagnostic message |
| `Space q` | Diagnostic list |
| `:lclose` | Close location list |

The diagnostic list mapping is:

```lua
vim.keymap.set("n", "<leader>q", function()
    vim.diagnostic.setloclist({ open = true })
end)
```

Here, `<leader>` is configured as the space bar.

## Searching

| Command | Action |
|---|---|
| `/text` | Search current file |
| `n` / `N` | Next / previous result |
| `*` | Search word under cursor |
| `:noh` | Clear highlighting |
| `:%s/old/new/g` | Replace throughout file |
| `:%s/old/new/gc` | Replace with confirmation |

For searching across a project, [ripgrep](https://github.com/BurntSushi/ripgrep) can be used through Neovim's quickfix list.

Add to `init.vim`:

```vim
set grepprg=rg\ --vimgrep\ --smart-case
set grepformat=%f:%l:%c:%m
```

Then:

```vim
:grep! calculate_rms
:copen
```

Navigate using `:cnext` and `:cprev`, and close the list with `:cclose`.

# Git and testing

I keep Git and testing in separate iTerm2 panes rather than adding Neovim plugins.

Depending on the level of the project I will often use a Makefile to run the project's checks and apply fixes, rather than running each tool individually by creating a make and fix target:

```bash
make check
make fix
```

Git commands are run directly in a separate iTerm2 pane.

# References

- [Neovim documentation](https://neovim.io/doc/user/)
- [Neovim LSP documentation](https://neovim.io/doc/user/lsp.html)
- [Neovim diagnostic API](https://neovim.io/doc/user/diagnostic.html)
- [Basedpyright documentation](https://docs.basedpyright.com/)
- [Ruff documentation](https://docs.astral.sh/ruff/)
- [ripgrep](https://github.com/BurntSushi/ripgrep)
- [iTerm2 documentation](https://iterm2.com/documentation.html)

<script src="https://utteranc.es/client.js"
        repo="skoopsy/skoopsy.github.io"
        issue-term="pathname"
        label="nvim-python"
        theme="preferred-color-scheme"
        crossorigin="anonymous"
        async>
</script>

Copyright © 2026 David O'Connor
