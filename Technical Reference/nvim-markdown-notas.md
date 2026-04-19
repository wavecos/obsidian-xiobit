# Neovim como sistema de notas (AstroNvim v5)

Setup probado en **Arch Linux / Debian** con **AstroNvim v5** y **lazy.nvim**.

---

## Requisitos previos

```bash
# Node y npm (para markdown-preview)
node --version && npm --version

# Diccionario español para el corrector
mkdir -p ~/.local/share/nvim/site/spell

curl -o ~/.local/share/nvim/site/spell/es.utf-8.spl \
  https://ftp.nluug.nl/pub/vim/runtime/spell/es.utf-8.spl

curl -o ~/.local/share/nvim/site/spell/es.utf-8.sug \
  https://ftp.nluug.nl/pub/vim/runtime/spell/es.utf-8.sug
```

---

## Estructura de archivos relevante

```
~/.config/nvim/
├── init.lua               ← no tocar
└── lua/
    ├── lazy_setup.lua     ← no tocar
    ├── polish.lua         ← keymaps y autocmds personales
    └── plugins/
        └── markdown.lua   ← plugins de Markdown (crear)
```

Carpeta de notas:

```bash
mkdir -p ~/notas/templates
```

---

## 1. Crear `~/.config/nvim/lua/plugins/markdown.lua`

```lua
return {

  -- ─────────────────────────────────────────────
  -- 1. obsidian.nvim — gestión del vault
  -- ─────────────────────────────────────────────
  {
    "epwalsh/obsidian.nvim",
    version = "*",
    lazy = true,
    ft = "markdown",
    dependencies = {
      "nvim-lua/plenary.nvim",
      "nvim-telescope/telescope.nvim",
      "nvim-treesitter/nvim-treesitter",
    },
    opts = {
      workspaces = {
        {
          name = "personal",
          path = "~/notas",
        },
      },

      note_id_func = function(title)
        if title ~= nil then
          return title:gsub(" ", "-"):gsub("[^A-Za-z0-9-]", ""):lower()
        else
          return os.date("%Y-%m-%d")
        end
      end,

      new_notes_location = "current_dir",

      templates = {
        folder = "templates",
        date_format = "%Y-%m-%d",
        time_format = "%H:%M",
      },

      completion = { nvim_cmp = true, min_chars = 2 },

      ui = {
        enable = true,
        checkboxes = {
          [" "] = { char = "󰄱", hl_group = "ObsidianTodo" },
          ["x"] = { char = "", hl_group = "ObsidianDone" },
        },
      },
    },
  },

  -- ─────────────────────────────────────────────
  -- 2. render-markdown.nvim — render inline
  -- ─────────────────────────────────────────────
  {
    "MeanderingProgrammer/render-markdown.nvim",
    dependencies = {
      "nvim-treesitter/nvim-treesitter",
      "nvim-tree/nvim-web-devicons",
    },
    ft = { "markdown" },
    opts = {
      enabled = false, -- apagado por defecto, toggle con keymap
    },
  },

  -- ─────────────────────────────────────────────
  -- 3. markdown-preview.nvim — preview en browser
  -- ─────────────────────────────────────────────
  {
    "iamcco/markdown-preview.nvim",
    cmd = { "MarkdownPreview", "MarkdownPreviewStop" },
    build = function()
      vim.fn.system "cd ~/.local/share/nvim/lazy/markdown-preview.nvim/app && npm install"
    end,
    ft = { "markdown" },
    config = function()
      vim.g.mkdp_auto_close = 1
      vim.g.mkdp_theme = "dark"
      vim.g.mkdp_browser = ""
    end,
  },

}
```

> **Nota:** El `build` usa `npm` directamente para evitar el warning de `yarn.lock`
> que aparece con el build por defecto al hacer `:Lazy sync`. El plugin funciona
> igual pero sin el warning. Si igual aparece, se puede ignorar sin problema.

---

## 2. Agregar al final de `~/.config/nvim/lua/polish.lua`

```lua
-- ─────────────────────────────────────────────
-- Markdown: opciones y keymaps
-- ─────────────────────────────────────────────
vim.api.nvim_create_autocmd("FileType", {
  pattern = "markdown",
  callback = function()
    -- Opciones locales
    vim.opt_local.wrap = true
    vim.opt_local.linebreak = true
    vim.opt_local.spell = true
    vim.opt_local.spelllang = "es,en"
    vim.opt_local.conceallevel = 2

    -- Keymaps (solo activos en buffers markdown)
    local map = vim.keymap.set
    local opts = { buffer = true, silent = true }

    -- render-markdown
    map("n", "<leader>mr", "<cmd>RenderMarkdown toggle<cr>",
      vim.tbl_extend("force", opts, { desc = "Toggle render Markdown" }))

    -- markdown-preview
    map("n", "<leader>mp", "<cmd>MarkdownPreview<cr>",
      vim.tbl_extend("force", opts, { desc = "Preview en browser" }))
    map("n", "<leader>ms", "<cmd>MarkdownPreviewStop<cr>",
      vim.tbl_extend("force", opts, { desc = "Cerrar preview" }))

    -- obsidian (solo funciona dentro de ~/notas)
    map("n", "<leader>on", "<cmd>ObsidianNew<cr>",
      vim.tbl_extend("force", opts, { desc = "Nueva nota" }))
    map("n", "<leader>ot", "<cmd>ObsidianToday<cr>",
      vim.tbl_extend("force", opts, { desc = "Nota de hoy" }))
    map("n", "<leader>oy", "<cmd>ObsidianYesterday<cr>",
      vim.tbl_extend("force", opts, { desc = "Nota de ayer" }))
    map("n", "<leader>os", "<cmd>ObsidianSearch<cr>",
      vim.tbl_extend("force", opts, { desc = "Buscar en vault" }))
    map("n", "<leader>oq", "<cmd>ObsidianQuickSwitch<cr>",
      vim.tbl_extend("force", opts, { desc = "Cambiar nota" }))
    map("n", "<leader>ob", "<cmd>ObsidianBacklinks<cr>",
      vim.tbl_extend("force", opts, { desc = "Backlinks" }))
    map("n", "<leader>ol", "<cmd>ObsidianFollowLink<cr>",
      vim.tbl_extend("force", opts, { desc = "Seguir link" }))
    map("n", "<leader>og", "<cmd>ObsidianTags<cr>",
      vim.tbl_extend("force", opts, { desc = "Ver tags" }))
    map("n", "<leader>oe", "<cmd>ObsidianTemplate<cr>",
      vim.tbl_extend("force", opts, { desc = "Insertar plantilla" }))
  end,
})
```

---

## 3. Instalar plugins y parsers

Dentro de nvim:

```vim
:Lazy sync
:TSInstall markdown markdown_inline
```

---

## Keymaps de referencia

| Keymap | Acción | Disponible en |
|---|---|---|
| `<leader>mr` | Toggle render inline | cualquier `.md` |
| `<leader>mp` | Abrir preview en browser | cualquier `.md` |
| `<leader>ms` | Cerrar preview browser | cualquier `.md` |
| `<leader>on` | Nueva nota | vault `~/notas` |
| `<leader>ot` | Nota de hoy | vault `~/notas` |
| `<leader>oy` | Nota de ayer | vault `~/notas` |
| `<leader>os` | Buscar en vault | vault `~/notas` |
| `<leader>oq` | Cambiar nota rápido | vault `~/notas` |
| `<leader>ob` | Backlinks | vault `~/notas` |
| `<leader>ol` | Seguir `[[link]]` | vault `~/notas` |
| `<leader>og` | Ver tags | vault `~/notas` |
| `<leader>oe` | Insertar plantilla | vault `~/notas` |

---

## Comportamiento por contexto

| Contexto | Comportamiento |
|---|---|
| Archivo `.md` dentro de `~/notas` | Todo activo: obsidian + render + preview |
| Archivo `.md` fuera del vault (ej. `README.md`) | Solo render y preview, obsidian inactivo |
| Archivo de código (`.lua`, `.java`, etc.) | Ningún plugin de markdown activo |

---

## Problemas conocidos

### `yarn.lock` warning en `:Lazy sync`
`markdown-preview.nvim` modifica su `yarn.lock` durante el build, lo que Lazy
detecta como cambios locales. No afecta el funcionamiento. Se puede ignorar,
o reinstalar el plugin con `x` + `I` dentro de la UI de Lazy.

### Warning de diccionario español
```
Warning: Cannot find word list "es.utf-8.spl"
```
Solución: instalar el diccionario manualmente con los `curl` del paso de
requisitos previos.

### Campo `detect_cwd` deprecado
Versiones recientes de `obsidian.nvim` eliminaron `detect_cwd`. No incluir
esa opción en la config. El plugin detecta el workspace automáticamente.
