---
name: nvim_plugin_structure
description: Standard Neovim Lua plugin layout with init.lua entry point and modular
  components
tags:
- neovim
- lua-plugin
created_at: '2026-07-27T22:05:53.158425+00:00'
---

A standard Neovim Lua plugin follows this structure:

```
plugin-name/
├── lua/plugin-name/init.lua    -- Entry point, setup(), keymaps
└── README.md                   -- Plugin documentation
```

Key patterns used in the user's config (from `init.lua`):
- Uses vim.pack.add with GitHub URLs for plugin installation
- Setup function pattern: require('plugin').setup({ opts })
- Keymap registration via vim.keymap.set or vim.api.nvim_set_keymap
