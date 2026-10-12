---
name: neovim_plugin_development
description: Guide for writing Neovim plugins using Lua, including structure, loading mechanism, and best practices
tags:
- neovim
- lua
- plugin
- development
created_at: '2026-07-12T20:04:43.612670+00:00'
---

# Writing a Neovim Plugin

## Directory Structure
```
my-plugin/
├── lua/
│   └── my_plugin/
│       ├── init.lua      # Main entry point
│       └── utils.lua     # Helper functions (optional)
├── plugin/
│   └── my_plugin.vim     # Loads the Lua code
├── README.md
└── LICENSE
```

## Loading Mechanism
1. Neovim loads `.vim` files from `plugin/` directory on startup
2. These vim files use `require()` to load Lua modules
3. The `lua/` directory is searched via runtimepath

## Example Plugin (Hello World)

**plugin/my_plugin.vim:**
```vim
" Load the Lua module
require('my_plugin').init()
```

**lua/my_plugin/init.lua:**
```lua
local M = {}

function M.init()
    -- Create a command
    vim.api.nvim_create_user_command('HelloPlugin', function()
        print("Hello from my plugin!")
    end, {
        desc = 'Prints a message!'
    })
    
    -- Or set up keymaps
    vim.keymap.set('n', '<leader>hp', function()
        print("Hello from my plugin!")
    end, { desc = 'Hello Plugin', noremap = true, silent = true })
end

return M
```

## Key APIs to Know

### Creating Commands
```lua
vim.api.nvim_create_user_command('CommandName', function(args)
    -- args.line1, args.line2 for visual range
    -- args.args for arguments
end, {
    nargs = 0,           -- or '*', '+', etc.
    range = true,        -- enable :range
    complete = 'file',   -- tab completion type
    desc = 'Description'
})
```

### Key Mappings (Neovim 0.7+)
```lua
vim.keymap.set('n', '<leader>key', function()
    -- action
end, { 
    desc = 'Description',
    noremap = true,
    silent = true 
})
```

### Autocommands
```lua
vim.api.nvim_create_autocmd('BufReadPost', {
    callback = function(args)
        local buf = args.buf
        -- do something with buf
    end
})
```

## Best Practices

1. **Use Lua** - First-class support, faster than Vimscript
2. **Modular structure** - Split into multiple files if needed
3. **Provide documentation** - Add vimdoc in `doc/` directory
4. **Handle cleanup** - Use `vim.api.nvim_create_autocmd` with proper patterns
5. **Use vim.keymap.set** instead of `nnoremap` (Neovim 0.7+)
6. **Check for dependencies** - Verify API availability before using new features

## Single-Session Reuse Pattern (buffer + job plugins)

For plugins that spawn a buffer/window/background job (e.g. terminal-like chat UIs),
users often expect the command to *reuse* an existing instance instead of opening
a new one every time. Pattern:

```lua
M.state = nil  -- { buf, win, job_id, append, ... }

local function open(...)
    local state = M.state
    if state and vim.api.nvim_buf_is_valid(state.buf) then
        -- reuse: focus existing window (or re-open one for the buffer),
        -- send new prompt to the running job; restart job if it exited
        return
    end
    -- else create buffer/window/job from scratch, store in M.state
end
```

Key points:
- Keep all per-session data (job_id, closures like append/send) in one `state`
  table stored on the module, not as scattered local upvalues — otherwise the
  reuse path can't reach them.
- `bufhidden = 'wipe'` + a `BufWipeout` autocmd clears `M.state` on close, so a
  closed session never lingers. Guard with `if M.state == state then M.state = nil end`.
- On reuse, check `job_id > 0` to decide between `chansend` (continue conversation)
  and `jobstart` (restart in same buffer).
- Buffer-local keymaps (`{ buffer = buf }`) die with the buffer, so no manual cleanup.

## Documentation (vimdoc)

Create `doc/my_plugin.txt`:
```
*my_plugin.txt*        Plugin documentation

CONTENTS                                            *my_plugin-contents*
    Introduction..................................................|my_plugin-intro|
    Commands......................................................|my_plugin-commands|

INTRODUCTION                                      *my_plugin-intro*

This plugin does something useful.

COMMANDS                                        *my_plugin-commands*

:HelloPlugin                                    *:HelloPlugin*
    Prints a message.
```

Then run `:helptags doc/` to generate tags.

## Testing Your Plugin

1. Place in `~/.local/share/nvim/site/lua/` or use packer/nui/lazy.nvim
2. Restart Neovim
3. Test commands with `:HelloPlugin`
4. Check for errors with `:messages`
