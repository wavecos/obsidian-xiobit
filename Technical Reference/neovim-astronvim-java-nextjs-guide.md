# Neovim + AstroNvim Setup Guide
## Java Spring Boot, Next.js & Angular Development

> **Tested with:** AstroNvim v5 · nvim 0.10+ · macOS & Linux

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Neovim](#2-install-neovim)
3. [Install AstroNvim](#3-install-astronvim)
4. [Configure AstroCommunity Packs](#4-configure-astrocommunity-packs)
5. [Java / Spring Boot Config](#5-java--spring-boot-config)
6. [Running & Debugging Spring Boot from Neovim](#6-running--debugging-spring-boot-from-neovim)
7. [Next.js / TypeScript Config](#7-nextjs--typescript-config)
8. [Angular Config](#8-angular-config)
9. [Prettier Formatting](#9-prettier-formatting)
10. [Final Steps](#10-final-steps)
11. [Key Bindings Reference](#11-key-bindings-reference)
12. [Troubleshooting](#12-troubleshooting)

---

## 1. Prerequisites

Before installing Neovim, make sure the following are available on your system.

### Required tools

```bash
# Git
git --version

# Node.js 18+ (for TypeScript LSP tools)
node --version
npm --version

# Java 17+ (21 recommended for modern Spring Boot)
java -version

# Angular CLI (for Angular projects)
ng version

# ripgrep (for Telescope file search)
rg --version

# A Nerd Font (for icons) — install from https://www.nerdfonts.com
```

### Install missing tools

**macOS (Homebrew):**
```bash
brew install git node ripgrep
brew install --cask font-jetbrains-mono-nerd-font
```

**Ubuntu / Debian:**
```bash
sudo apt install git nodejs npm ripgrep
```

**Java — recommended via SDKMAN (works on macOS & Linux):**
```bash
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install java 21-tem
```

**Angular CLI & Language Service (global):**
```bash
npm install -g @angular/cli @angular/language-service
```

---

## 2. Install Neovim

You need **Neovim 0.10 or newer**.

**macOS:**
```bash
brew install neovim
```

**Ubuntu / Debian:**
```bash
# Official PPA for latest stable
sudo add-apt-repository ppa:neovim-ppa/stable -y
sudo apt update
sudo apt install neovim
```

**Verify:**
```bash
nvim --version
# Should show: NVIM v0.10.x or higher
```

---

## 3. Install AstroNvim

### Step 1: Back up any existing Neovim config

```bash
mv ~/.config/nvim ~/.config/nvim.bak
mv ~/.local/share/nvim ~/.local/share/nvim.bak
mv ~/.local/state/nvim ~/.local/state/nvim.bak
mv ~/.cache/nvim ~/.cache/nvim.bak
```

### Step 2: Clone the AstroNvim template

```bash
git clone https://github.com/AstroNvim/template ~/.config/nvim
rm -rf ~/.config/nvim/.git  # detach from template repo
```

### Step 3: Launch Neovim

```bash
nvim
```

AstroNvim will bootstrap itself automatically on first launch. Wait for all plugins to install, then restart Neovim.

---

## 4. Configure AstroCommunity Packs

AstroCommunity provides pre-built, maintained packs for language support. This is the **correct and recommended** way to add Java, TypeScript, and Angular support.

### Critical: Activate community.lua

Open `~/.config/nvim/lua/community.lua`. By default it has a guard at the top that **disables the entire file**:

```lua
if true then return {} end -- WARN: REMOVE THIS LINE TO ACTIVATE THIS FILE
```

**Delete that line entirely.** Then replace the contents with:

```lua
-- ~/.config/nvim/lua/community.lua

---@type LazySpec
return {
  "AstroNvim/astrocommunity",
  { import = "astrocommunity.pack.lua" },

  -- Java: includes jdtls LSP, DAP debugger, test runner, treesitter
  { import = "astrocommunity.pack.java" },

  -- TypeScript / Next.js: uses vtsls LSP + ts/tsx/js treesitter
  { import = "astrocommunity.pack.typescript" },

  -- Angular: angularls LSP + angular treesitter for templates
  { import = "astrocommunity.pack.angular" },

  -- Tailwind CSS (common in Next.js and Angular projects)
  { import = "astrocommunity.pack.tailwindcss" },

  -- JSON: package.json, tsconfig.json, angular.json, etc.
  { import = "astrocommunity.pack.json" },

  -- HTML + CSS support
  { import = "astrocommunity.pack.html-css" },

  -- Inline debug variable display
  { import = "astrocommunity.debugging.nvim-dap-virtual-text" },
}
```

> **Note:** The `angular` pack provides the Angular Language Server (`angularls`) which gives template completions, diagnostics, and go-to-definition between TypeScript components and their HTML templates. It works alongside the `typescript` pack — both LSPs attach to Angular projects.

---

## 5. Java / Spring Boot Config

Create the file `~/.config/nvim/lua/plugins/java.lua`:

```lua
-- ~/.config/nvim/lua/plugins/java.lua

---@type LazySpec
return {

  -- Configure jdtls LSP settings
  -- NOTE: settings go inside the jdtls() function — NOT as a flat opts table
  {
    "mfussenegger/nvim-jdtls",
    opts = {
      jdtls = function(opts)
        opts.settings = {
          java = {
            configuration = {
              runtimes = {
                -- Adjust this path to match your Java installation:
                --
                -- SDKMAN (macOS/Linux):
                --   path = os.getenv("HOME") .. "/.sdkman/candidates/java/current"
                --
                -- Homebrew (macOS):
                --   path = "/opt/homebrew/opt/openjdk@21"
                --
                -- apt (Ubuntu/Debian):
                --   path = "/usr/lib/jvm/java-21-openjdk-amd64"
                {
                  name = "JavaSE-21",
                  path = os.getenv("HOME") .. "/.sdkman/candidates/java/current",
                  default = true,
                },
              },
            },
            eclipse = { downloadSources = true },
            maven = { downloadSources = true },
            implementationsCodeLens = { enabled = true },
            referencesCodeLens = { enabled = true },
            inlayHints = { parameterNames = { enabled = "all" } },
          },
        }
        return opts -- required!
      end,
      -- DAP config: adds "Attach to Spring Boot" option on port 5005
      dap = function(opts, extra)
        local dap = require("dap")
        if not dap.configurations.java then dap.configurations.java = {} end
        table.insert(dap.configurations.java, {
          type = "java",
          request = "attach",
          name = "Attach to Spring Boot (port 5005)",
          hostName = "localhost",
          port = 5005,
        })
      end,
    },
  },

  -- Spring Boot support
  {
    "JavaHello/spring-boot.nvim",
    ft = "java",
    dependencies = { "mfussenegger/nvim-jdtls" },
    opts = {},
  },
}
```

### Find your Java path

Run one of these to find the correct path for your system:

```bash
# SDKMAN
echo $JAVA_HOME

# macOS Homebrew
/usr/libexec/java_home -v 21

# Ubuntu/Debian (apt)
update-alternatives --list java
# then strip "/bin/java" from the end of the path

# Generic
dirname $(dirname $(readlink -f $(which java)))
```

---

## 6. Running & Debugging Spring Boot from Neovim

### Shell scripts for running with `.env.local`

Spring Boot projects often use a `.env.local` file for environment variables. Create these scripts in your project root:

**`run-dev.sh`** — Run normally:
```bash
#!/bin/zsh
set -a
source <(grep -v '^#' .env.local | sed 's/#.*//' | grep -v '^$')
set +a
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev
```

**`run-debug.sh`** — Run with remote debug enabled on port 5005:
```bash
#!/bin/zsh
set -a
source <(grep -v '^#' .env.local | sed 's/#.*//' | grep -v '^$')
set +a
./mvnw spring-boot:run -Dspring-boot.run.profiles=dev \
  -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
```

Make them executable:
```bash
chmod +x run-dev.sh run-debug.sh
```

> **macOS important:** Use `#!/bin/zsh` NOT `#!/bin/bash`. macOS ships with Bash 3.2 (from 2007) which does not support `source <(...)` (process substitution). zsh handles it correctly. If your env vars are not loading, this is likely the cause.

### Running from Neovim

Open a terminal inside nvim with ToggleTerm (`<F7>` or `<C-\>`) and run:

```bash
./run-dev.sh      # normal run
./run-debug.sh    # run with debugger listening on port 5005
```

### Debugging workflow

1. Start the app with `./run-debug.sh` in a ToggleTerm terminal
2. Set breakpoints in your Java code: `Space` `d` `b`
3. Attach the debugger: `Space` `d` `c` → select **"Attach to Spring Boot (port 5005)"**
4. Trigger a request (curl, browser, Postman) that hits your breakpoint
5. Use debug controls:

| Key | Action |
|-----|--------|
| `Space` `d` `c` | Continue |
| `Space` `d` `o` | Step over |
| `Space` `d` `i` | Step into |
| `Space` `d` `O` | Step out |
| `Space` `d` `u` | Toggle DAP UI (variables, call stack) |
| `Space` `d` `b` | Toggle breakpoint |

The DAP attach configuration is defined in `~/.config/nvim/lua/plugins/java.lua` (see section 5).

---

## 7. Next.js / TypeScript Config

The `astrocommunity.pack.typescript` pack handles the LSP (`vtsls`) and treesitter parsers automatically. No extra LSP config is needed.

If you want to add ESLint auto-fix on save, create `~/.config/nvim/lua/plugins/typescript.lua`:

```lua
-- ~/.config/nvim/lua/plugins/typescript.lua

---@type LazySpec
return {
  {
    "AstroNvim/astrolsp",
    opts = {
      config = {
        eslint = {
          settings = {
            workingDirectories = { mode = "auto" },
          },
          on_attach = function(_, bufnr)
            -- Auto-fix ESLint errors on save
            vim.api.nvim_create_autocmd("BufWritePre", {
              buffer = bufnr,
              command = "EslintFixAll",
            })
          end,
        },
      },
    },
  },
}
```

---

## 8. Angular Config

The `astrocommunity.pack.angular` pack (added in section 4) provides the core Angular Language Server. This section covers additional configuration for a complete Angular development experience.

### How Angular LSP works

When you open an Angular project (one with `angular.json` in the root), **two LSPs attach**:

- **`angularls`** — Angular Language Server: provides template completions, template diagnostics, go-to-definition from template to component, and Angular-specific code intelligence
- **`vtsls`** — TypeScript LSP: provides standard TypeScript features (autocompletions, type checking, refactoring) in `.ts` files

Both work together seamlessly. You get full IntelliSense in both `.ts` and `.html` template files.

### AstroLSP config for Angular

Configure `angularls` root detection in `~/.config/nvim/lua/plugins/astrolsp.lua`:

```lua
-- ~/.config/nvim/lua/plugins/astrolsp.lua

---@type LazySpec
return {
  "AstroNvim/astrolsp",
  ---@type AstroLSPOpts
  opts = {
    features = {
      codelens = true,
      inlay_hints = false,
      semantic_tokens = true,
    },
    formatting = {
      format_on_save = {
        enabled = true,
        ignore_filetypes = {
          "htmlangular", -- let prettier handle Angular templates
        },
      },
      disabled = {
        "angularls", -- use prettier instead for formatting
      },
      timeout_ms = 2000,
    },
    config = {
      angularls = {
        -- angularls needs the project root with angular.json
        root_dir = function(fname)
          local util = require "lspconfig.util"
          return util.root_pattern("angular.json", "project.json")(fname)
        end,
      },
    },
  },
}
```

### Mason tools for Angular

Activate `~/.config/nvim/lua/plugins/mason.lua` (remove the guard line) and include Angular tools:

```lua
-- ~/.config/nvim/lua/plugins/mason.lua

---@type LazySpec
return {
  {
    "WhoIsSethDaniel/mason-tool-installer.nvim",
    opts = {
      ensure_installed = {
        -- Language servers
        "lua-language-server",
        "angular-language-server",
        "typescript-language-server",
        "css-lsp",
        "html-lsp",
        "tailwindcss-language-server",
        "json-lsp",

        -- Formatters
        "stylua",
        "prettier",

        -- Linters
        "eslint_d",

        -- Debuggers
        "js-debug-adapter",  -- for Angular/TS/Next.js debugging

        -- Tools
        "tree-sitter-cli",
      },
    },
  },
}
```

### Treesitter parsers for Angular

Activate `~/.config/nvim/lua/plugins/treesitter.lua` (remove the guard line):

```lua
-- ~/.config/nvim/lua/plugins/treesitter.lua

---@type LazySpec
return {
  "nvim-treesitter/nvim-treesitter",
  opts = {
    ensure_installed = {
      "lua",
      "vim",
      -- Angular / Web
      "angular",      -- Angular template syntax highlighting
      "typescript",
      "javascript",
      "html",
      "css",
      "scss",
      "json",
      "yaml",
      -- Java
      "java",
    },
  },
}
```

### Running Angular from Neovim (custom keybindings)

Create `~/.config/nvim/lua/plugins/angular.lua` for Angular-specific terminal commands and Chrome debugging:

```lua
-- ~/.config/nvim/lua/plugins/angular.lua

---@type LazySpec
return {
  -- Terminal commands for Angular CLI
  {
    "akinsho/toggleterm.nvim",
    opts = {
      size = 15,
      open_mapping = [[<C-\>]],
      direction = "horizontal",
    },
    keys = {
      {
        "<Leader>rs",
        function()
          local Terminal = require("toggleterm.terminal").Terminal
          local ng_serve = Terminal:new {
            cmd = "npm start",
            direction = "horizontal",
            close_on_exit = false,
            auto_scroll = true,
          }
          ng_serve:toggle()
        end,
        desc = "Angular: ng serve (npm start)",
      },
      {
        "<Leader>rt",
        function()
          local Terminal = require("toggleterm.terminal").Terminal
          local ng_test = Terminal:new {
            cmd = "npm test",
            direction = "horizontal",
            close_on_exit = false,
          }
          ng_test:toggle()
        end,
        desc = "Angular: ng test",
      },
      {
        "<Leader>rb",
        function()
          local Terminal = require("toggleterm.terminal").Terminal
          local ng_build = Terminal:new {
            cmd = "npm run build",
            direction = "horizontal",
            close_on_exit = false,
          }
          ng_build:toggle()
        end,
        desc = "Angular: ng build",
      },
      {
        "<Leader>rg",
        function()
          vim.ui.input({ prompt = "ng generate: " }, function(input)
            if input then
              local Terminal = require("toggleterm.terminal").Terminal
              local ng_gen = Terminal:new {
                cmd = "ng generate " .. input,
                direction = "horizontal",
                close_on_exit = false,
              }
              ng_gen:toggle()
            end
          end)
        end,
        desc = "Angular: ng generate ...",
      },
    },
  },

  -- DAP configuration for Angular/TypeScript debugging via Chrome
  {
    "mfussenegger/nvim-dap",
    config = function()
      local dap = require "dap"

      -- Chrome debugging adapter (via js-debug-adapter from Mason)
      if not dap.adapters["pwa-chrome"] then
        dap.adapters["pwa-chrome"] = {
          type = "server",
          host = "localhost",
          port = "${port}",
          executable = {
            command = "node",
            args = {
              vim.fn.stdpath "data"
                .. "/mason/packages/js-debug-adapter/js-debug/src/dapDebugServer.js",
              "${port}",
            },
          },
        }
      end

      -- Angular/TypeScript debug configurations
      for _, lang in ipairs { "typescript", "javascript" } do
        if not dap.configurations[lang] then dap.configurations[lang] = {} end
        table.insert(dap.configurations[lang], {
          type = "pwa-chrome",
          request = "launch",
          name = "Launch Chrome (Angular localhost:4200)",
          url = "http://localhost:4200",
          webRoot = "${workspaceFolder}",
          sourceMaps = true,
        })
        table.insert(dap.configurations[lang], {
          type = "pwa-chrome",
          request = "attach",
          name = "Attach to Chrome",
          port = 9222,
          webRoot = "${workspaceFolder}",
          sourceMaps = true,
        })
      end
    end,
  },

  -- ESLint integration via none-ls
  {
    "nvimtools/none-ls.nvim",
    opts = function(_, opts)
      local null_ls = require "null-ls"
      opts.sources = opts.sources or {}
      table.insert(opts.sources, null_ls.builtins.diagnostics.eslint_d)
      table.insert(opts.sources, null_ls.builtins.code_actions.eslint_d)
    end,
  },
}
```

### Angular debugging workflow

1. Start `npm start` (or `ng serve`) using `<Leader>rs` — the app runs on `http://localhost:4200`
2. Set breakpoints in your TypeScript code: `Space` `d` `b`
3. Start the debugger: `Space` `d` `c` → select **"Launch Chrome (Angular localhost:4200)"**
4. Chrome opens connected to the debugger. Interact with the app to hit breakpoints
5. Use the same debug controls as Java (see section 6)

### Angular keybindings summary

| Key | Action |
|-----|--------|
| `<Leader>rs` | Run `npm start` (ng serve) |
| `<Leader>rt` | Run `npm test` |
| `<Leader>rb` | Run `npm run build` |
| `<Leader>rg` | Run `ng generate ...` (prompts for input) |
| `Ctrl + \` | Toggle terminal |

---

## 9. Prettier Formatting

AstroNvim v4+ uses `conform.nvim` for formatting. Create `~/.config/nvim/lua/plugins/formatting.lua`:

```lua
-- ~/.config/nvim/lua/plugins/formatting.lua

---@type LazySpec
return {
  {
    "stevearc/conform.nvim",
    opts = {
      formatters_by_ft = {
        javascript      = { "prettier" },
        typescript      = { "prettier" },
        javascriptreact = { "prettier" },
        typescriptreact = { "prettier" },
        css             = { "prettier" },
        scss            = { "prettier" },
        html            = { "prettier" },
        htmlangular     = { "prettier" },  -- Angular templates
        json            = { "prettier" },
        yaml            = { "prettier" },
        markdown        = { "prettier" },
      },
      format_on_save = {
        timeout_ms = 2000,
        lsp_fallback = true,
      },
    },
  },
}
```

> **Angular note:** The `htmlangular` filetype is how Neovim identifies Angular template files (`.html` files inside Angular components). Adding it to conform ensures they are formatted by prettier on save.

Then install prettier via Mason inside Neovim:

```
:MasonInstall prettier prettierd eslint-lsp
```

---

## 10. Final Steps

### Sync all plugins

Open Neovim and run:

```
:Lazy sync
```

Wait for everything to install. On first open of a `.java` file, `jdtls` will index your project — this takes 1-2 minutes. A progress indicator appears in the bottom right.

### Verify LSPs are working

```
# Open a Java file, then:
:LspInfo
# Should show: jdtls attached

# Open a .tsx file (Next.js), then:
:LspInfo
# Should show: vtsls, eslint attached

# Open a .ts file inside an Angular project, then:
:LspInfo
# Should show: angularls, vtsls attached

# Open an Angular template .html file, then:
:LspInfo
# Should show: angularls attached
```

### Your final file structure

```
~/.config/nvim/
├── init.lua
└── lua/
    ├── community.lua        ← AstroCommunity packs (no guard line!)
    ├── polish.lua
    └── plugins/
        ├── astrolsp.lua     ← LSP config (angularls root_dir, formatting)
        ├── mason.lua        ← Auto-install LSPs, formatters, debuggers
        ├── treesitter.lua   ← Treesitter parsers (angular, typescript, etc.)
        ├── java.lua         ← jdtls settings + Spring Boot DAP
        ├── angular.lua      ← Angular terminal commands + Chrome DAP
        ├── typescript.lua   ← ESLint on-save (optional)
        └── formatting.lua   ← Prettier via conform.nvim
```

---

## 11. Key Bindings Reference

AstroNvim uses `Space` as the leader key.

### LSP (works in any language)

| Key | Action |
|-----|--------|
| `Space` `l` `a` | Code actions (imports, fixes, refactors) |
| `Space` `l` `f` | Format file |
| `Space` `l` `r` | Rename symbol |
| `Space` `l` `d` | Show diagnostics list |
| `Space` `l` `i` | LSP info |
| `g` `d` | Go to definition |
| `g` `D` | Go to declaration |
| `g` `r` | Find all references |
| `g` `I` | Go to implementation |
| `K` | Hover documentation |
| `]` `d` | Next diagnostic |
| `[` `d` | Previous diagnostic |

### File Navigation

| Key | Action |
|-----|--------|
| `Space` `f` `f` | Find files (Telescope) |
| `Space` `f` `g` | Live grep (search text) |
| `Space` `e` | Toggle file explorer |
| `Space` `b` `d` | Close buffer |

### Debugging (Java / Angular / DAP)

| Key | Action |
|-----|--------|
| `Space` `d` `b` | Toggle breakpoint |
| `Space` `d` `c` | Continue / Start debugger |
| `Space` `d` `i` | Step into |
| `Space` `d` `o` | Step over |
| `Space` `d` `O` | Step out |
| `Space` `d` `u` | Toggle DAP UI |

### Angular Commands

| Key | Action |
|-----|--------|
| `<Leader>rs` | Run `npm start` (ng serve) |
| `<Leader>rt` | Run `npm test` |
| `<Leader>rb` | Run `npm run build` |
| `<Leader>rg` | Run `ng generate ...` |
| `Ctrl + \` | Toggle terminal |

### Git

| Key | Action |
|-----|--------|
| `Space` `g` `g` | Open Lazygit |
| `Space` `g` `s` | Git status |

---

## 12. Troubleshooting

### "attempt to call field 'setup' (a table value)" on startup

**Cause:** The guard line `if true then return {} end` is still at the top of `community.lua`, so the Java pack never loads and `nvim-jdtls` starts without its proper config wrapper.

**Fix:** Delete line 1 of `~/.config/nvim/lua/community.lua`.

---

### jdtls not attaching to Java files

1. Confirm Mason installed jdtls: `:Mason` → search for `jdtls` → should show installed
2. Check your Java path is correct in `java.lua`
3. Make sure you opened Neovim from the **project root** (where `pom.xml` or `build.gradle` lives)
4. Run `:LspLog` to see detailed error output

---

### angularls not attaching to Angular files

1. Confirm Mason installed angular-language-server: `:Mason` → search `angular`
2. Make sure `@angular/language-service` is installed globally: `npm list -g @angular/language-service`
3. Open Neovim from the **project root** (where `angular.json` lives)
4. Check `:LspInfo` when a `.ts` or `.html` file inside the Angular project is open
5. Run `:LspLog` to see detailed error output

**Common issue:** `angularls` requires `angular.json` in the project root to activate. If your Angular project is inside a monorepo or subdirectory, ensure you open nvim from the directory that contains `angular.json`.

---

### TypeScript LSP not working in Next.js

1. Open Neovim from the **project root** (where `package.json` lives)
2. Confirm vtsls is installed: `:Mason` → search `vtsls`
3. Check `:LspInfo` when a `.ts` or `.tsx` file is open

---

### Angular template completions not working

1. Verify both `angularls` and `vtsls` are attached: `:LspInfo`
2. Check that the `angular` treesitter parser is installed: `:TSInstallInfo`
3. Make sure the Angular project has been compiled at least once (`ng serve` or `ng build`) — `angularls` relies on the TypeScript compiler for some features
4. Restart the LSP: `:LspRestart`

---

### Prettier not formatting on save

1. Confirm prettier is installed: `:Mason` → search `prettier`
2. Check conform is configured: `:ConformInfo`
3. Make sure a `prettier.config.js` or `.prettierrc` exists in your project root
4. For Angular templates, verify `htmlangular` is in the `formatters_by_ft` table

---

### Environment variables not loading from `.env.local` (macOS)

**Cause:** macOS ships with Bash 3.2 which does not properly support `source <(...)` (process substitution). Shell scripts with `#!/bin/bash` will silently fail to load environment variables.

**Fix:** Change the shebang from `#!/bin/bash` to `#!/bin/zsh` in `run-dev.sh` and `run-debug.sh`. Verify with:

```bash
bash --version   # 3.2.x on macOS — broken
zsh --version    # 5.9+ on macOS — works
```

---

### Clean slate (reset everything)

If things are badly broken, wipe all Neovim data and start fresh:

```bash
rm -rf ~/.local/share/nvim
rm -rf ~/.local/state/nvim
rm -rf ~/.cache/nvim
nvim  # re-bootstraps automatically
```

---

*Guide based on AstroNvim v5, updated March 2026.*
