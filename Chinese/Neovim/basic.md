

`~/.config/nvim/init.lua`

```lua
local options = {
    -- Visuals
    termguicolors = true,            -- Enable 24-bit color support
    number = true,                   -- Show absolute line numbers
    relativenumber = true,           -- Show relative line numbers
    cursorline = true,               -- Highlight the current line
    signcolumn = "yes",              -- Always show the sign column

    -- Editing
    smartindent = true,              -- Automatically indent new lines
    tabstop = 4,                     -- Number of spaces per tab
    shiftwidth = 4,                  -- Number of spaces for indentation
    expandtab = true,                -- Convert tabs to spaces
    wrap = false,                    -- Disable line wrapping
    mouse = "",                      -- Disable mouse support

    -- Searching
    ignorecase = true,               -- Ignore case in searches
    smartcase = true,                -- Case-sensitive if uppercase is used
    incsearch = true,                -- Incrementally show search results
    hlsearch = true,                 -- Highlight search results

    -- Performance
    updatetime = 300,                -- Faster updates (for CursorHold, etc.)
    timeoutlen = 500,                -- Timeout for mapped sequences

    -- Backup and Undo
    undofile = true,                 -- Enable persistent undo
    swapfile = false,                -- Disable swap files
    backup = false,                  -- Disable backups

    -- Splits
    splitbelow = true,               -- Open horizontal splits below
    splitright = true,               -- Open vertical splits to the right

    -- Clipboard
    clipboard = "unnamedplus",       -- Use system clipboard

    -- Visual alignment
    scrolloff = 4,                   -- Keep 8 lines visible above/below cursor
    sidescrolloff = 4,               -- Keep 8 columns visible on the sides
}

for name, value in pairs(global_local) do
    vim.o[name] = value
end
```


