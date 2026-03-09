# AstroNvim — Navigation & Refactoring Guide
## Practical Daily Reference for Java Spring Boot, Next.js & Angular

> **AstroNvim v5+ · nvim-jdtls · vtsls · angularls**
> Leader key = `Space`

---

## Table of Contents

1. [File Navigation](#1-file-navigation)
2. [Code Navigation (LSP)](#2-code-navigation-lsp)
3. [Symbols & Project Structure](#3-symbols--project-structure)
4. [Diagnostics & Errors](#4-diagnostics--errors)
5. [Refactoring](#5-refactoring)
6. [Code Editing Tricks](#6-code-editing-tricks)
7. [Splits & Windows](#7-splits--windows)
8. [Running Spring Boot](#8-running-spring-boot)
9. [Running Angular](#9-running-angular)
10. [Daily Workflows](#10-daily-workflows)

---

## 1. File Navigation

### Telescope (Fuzzy Finder)

| Key | Action |
|---|---|
| `Space + f + f` | Find file by name in project |
| `Space + f + g` | Live grep — search text across all files |
| `Space + f + r` | Recent files |
| `Space + f + b` | Open buffers (currently open files) |
| `Space + f + s` | Find symbol by name |
| `Space + f + c` | Find AstroNvim config files |

Inside any Telescope window:

| Key | Action |
|---|---|
| `Ctrl + j / k` | Move up / down the results |
| `Ctrl + d / u` | Scroll preview down / up |
| `Ctrl + v` | Open in vertical split |
| `Ctrl + x` | Open in horizontal split |
| `Ctrl + t` | Open in new tab |
| `Esc` | Close |

### File Explorer (Neo-tree)

| Key | Action |
|---|---|
| `Space + e` | Toggle file explorer |
| `Space + o` | Focus file explorer |

Inside the file explorer:

| Key | Action |
|---|---|
| `Enter` | Open file / expand folder |
| `a` | Create new file or directory |
| `d` | Delete file |
| `r` | Rename file |
| `y` | Copy file |
| `x` | Cut file |
| `p` | Paste file |
| `?` | Show all available keybindings |
| `q` | Close explorer |

---

## 2. Code Navigation (LSP)

These work in **Java, TypeScript/Next.js, and Angular** as long as the LSP is attached.

### Jump Commands

| Key | Action | Notes |
|---|---|---|
| `g + d` | Go to **definition** | Where the symbol is defined |
| `g + D` | Go to **declaration** | Declaration vs definition |
| `g + i` | Go to **implementation** | For interfaces/abstract methods |
| `g + r` | Find all **references** | Opens list of every usage |
| `g + y` | Go to **type definition** | Follows the type of a variable |
| `K` | **Hover documentation** | Press twice to enter the popup |
| `Ctrl + o` | **Jump back** | Returns to where you came from |
| `Ctrl + i` | **Jump forward** | Opposite of jump back |

> `Ctrl + o` and `Ctrl + i` work like browser back/forward. Always use them after `g + d` to return to your original location.

### Angular-specific navigation

In Angular projects, `angularls` provides additional navigation:

| Action | How |
|---|---|
| Jump from template to component | `g + d` on a method/property used in the `.html` template |
| Jump from component to template | Use `Space + f + f` and type the component name + `.html` |
| Navigate to a component from its selector | `g + d` on an `<app-my-component>` tag in a template |
| See all usages of a component | `g + r` on the component class or its selector |

### Inside Hover Popup (`K` twice to enter)

| Key | Action |
|---|---|
| `q` | Close popup |
| `Ctrl + f / b` | Scroll documentation |

### Peek (without leaving current file)

| Key | Action |
|---|---|
| `Space + l + p` | Peek definition in a floating window |

---

## 3. Symbols & Project Structure

### Symbol Search

| Key | Action |
|---|---|
| `Space + l + s` | **Document symbols** — all classes/methods in current file |
| `Space + l + S` | **Workspace symbols** — search across the entire project |
| `Space + f + s` | Find symbol by name (Telescope) |

> When exploring an unfamiliar codebase, use `Space + l + S` and type a class name to jump directly to it — faster than grepping. Works across Java, TypeScript, and Angular.

### Outline View

| Key | Action |
|---|---|
| `Space + l + s` | Opens a structured view of the current file |

This shows the full class structure: fields, constructors, methods — great for large Java service classes, Angular component classes, or Next.js page components.

---

## 4. Diagnostics & Errors

### Navigating Errors

| Key | Action |
|---|---|
| `] + d` | Jump to **next** diagnostic (error / warning) |
| `[ + d` | Jump to **previous** diagnostic |
| `gl` | Show diagnostic message for **current line** |
| `Space + l + d` | All diagnostics for **current file** |
| `Space + l + D` | All diagnostics for **entire workspace** |

> In Angular projects, `angularls` provides template diagnostics (e.g., unknown properties, missing bindings, type mismatches in templates). These appear alongside TypeScript errors from `vtsls`.

### Quickfix List

When you have multiple errors, use the quickfix list to step through them:

| Key | Action |
|---|---|
| `Space + l + d` | Populate quickfix with file diagnostics |
| `] + q` | Next item in quickfix list |
| `[ + q` | Previous item in quickfix list |

---

## 5. Refactoring

### The Main Refactor Command

```
Space + l + a   →   Code Actions
```

This is the **most important refactoring key**. It opens a context-aware menu depending on where your cursor is. Always try this first when you want to refactor something.

### Java Code Actions (via jdtls)

Place your cursor on a class, method, variable, or import and press `Space + l + a`:

```
Organize Imports              ← removes unused, adds missing
Generate Getter / Setter
Generate Constructor
Generate toString()
Generate hashCode() and equals()
Override / Implement Methods  ← great for interfaces
Extract Method                ← select code in visual mode first
Extract Variable
Extract Constant
Extract Interface
Move Class to Another Package
Change Method Signature
Inline Variable
Convert to Lambda Expression
Add Javadoc
```

### TypeScript / Next.js Code Actions (via vtsls)

Press `Space + l + a` on any TS/TSX code:

```
Add missing import
Remove unused imports
Extract to function            ← select JSX/code in visual mode first
Extract to constant
Extract to new file
Infer function return type
Convert to async function
Convert default export to named export
Add explicit return type
React: wrap in fragment <>
React: convert class to function component
```

### Angular Code Actions (via angularls + vtsls)

In **Angular TypeScript files** (`.ts`), press `Space + l + a`:

```
Add missing import
Remove unused imports
Organize imports
Extract to function           ← select code in visual mode first
Extract to constant
Extract to new file
Infer function return type
Convert to async function
Add explicit return type
Generate component/service/pipe  ← Angular-specific
```

In **Angular template files** (`.html`), press `Space + l + a`:

```
Add missing property to component
Create method in component     ← when referencing an undefined method
Fix template binding errors
Convert to structural directive
```

> Angular templates benefit from both `angularls` and ESLint. If a method is used in a template but doesn't exist in the component, `angularls` will flag it and offer to create it.

### Rename Symbol

```
Space + l + r
```

Renames the symbol under the cursor **everywhere in the project** — across all files. Works for Java class names, method names, TypeScript variables, React component names, Angular component selectors, service names, etc.

> Always use this instead of find-and-replace for code. It's AST-aware and only renames the actual symbol, not random text matches.

### Format File

| Key | Action |
|---|---|
| `Space + l + f` | Format current file |

- Java files are formatted by `jdtls` (Eclipse formatter)
- TypeScript/JSX files are formatted by `prettier`
- Angular templates (`.html`) are formatted by `prettier` (via `htmlangular` filetype)
- SCSS/CSS files are formatted by `prettier`

---

## 6. Code Editing Tricks

### Comments

| Key | Action |
|---|---|
| `g + c + c` | Toggle line comment |
| `g + c` (visual mode) | Toggle comment on selection |
| `g + b + c` | Toggle block comment |

### Text Objects — Select Precisely

Press `v` to enter visual mode, then:

| Key | Selects |
|---|---|
| `i + w` | Current **word** |
| `i + W` | Current **WORD** (space-delimited) |
| `i + p` | Current **paragraph** |
| `i + "` | Inside **double quotes** |
| `i + '` | Inside **single quotes** |
| `i + (` or `i + )` | Inside **parentheses** |
| `i + {` or `i + }` | Inside **curly braces** |
| `i + [` or `i + ]` | Inside **brackets** |
| `a + {` | Curly braces **including** the braces |
| `i + t` | Inside **HTML/JSX/Angular tag** |
| `a + t` | Including **HTML/JSX/Angular tag** |

> `i` = inside (excluding delimiters), `a` = around (including delimiters)

> The `i + t` and `a + t` text objects are particularly useful in Angular templates for selecting content inside tags like `<div>`, `<app-header>`, etc.

### Moving Code

| Key | Action |
|---|---|
| `> + >` | Indent line right |
| `< + <` | Indent line left |
| `> + >` (visual) | Indent selection right |
| `< + <` (visual) | Indent selection left |
| `Alt + j` | Move line / selection **down** |
| `Alt + k` | Move line / selection **up** |
| `J` | Join current line with line below |

### Search & Replace

| Key | Action |
|---|---|
| `/pattern` | Search forward |
| `?pattern` | Search backward |
| `n` / `N` | Next / previous match |
| `*` | Search for word under cursor |
| `#` | Search for word under cursor (backward) |
| `:%s/old/new/g` | Replace all in file |
| `:%s/old/new/gc` | Replace all with confirmation |

### Undo / Redo

| Key | Action |
|---|---|
| `u` | Undo |
| `Ctrl + r` | Redo |

---

## 7. Splits & Windows

Working with multiple files side by side is essential — e.g., viewing a Spring Boot Controller and its Service at the same time, a Next.js page alongside its component, or an Angular component `.ts` alongside its `.html` template.

### Opening Splits

| Key | Action |
|---|---|
| `Space + \|` | Split **vertical** (side by side) |
| `Space + -` | Split **horizontal** (top / bottom) |

### Navigating Splits

| Key | Action |
|---|---|
| `Ctrl + h` | Move to split on the **left** |
| `Ctrl + l` | Move to split on the **right** |
| `Ctrl + j` | Move to split **below** |
| `Ctrl + k` | Move to split **above** |

### Resizing Splits

| Key | Action |
|---|---|
| `Ctrl + Up` | Increase height |
| `Ctrl + Down` | Decrease height |
| `Ctrl + Left` | Decrease width |
| `Ctrl + Right` | Increase width |

### Buffer Management

| Key | Action |
|---|---|
| `] + b` | Next open file |
| `[ + b` | Previous open file |
| `Space + c` | Close current buffer |
| `Space + b + D` | Close all other buffers |

---

## 8. Running Spring Boot

### Option A — Built-in Terminal (Zero Setup)

Toggle a terminal split, then run your project:

| Key | Action |
|---|---|
| `Space + t + h` | Open **horizontal** terminal split |
| `Space + t + v` | Open **vertical** terminal split |
| `Space + t + f` | Open **floating** terminal |

Inside the terminal:

```bash
# Maven
./mvnw spring-boot:run

# Gradle
./gradlew bootRun
```

To exit the terminal insert mode without closing it: press `Ctrl + \ + Ctrl + n`, then navigate away normally with `Ctrl + h/j/k/l`.

### Option B — overseer.nvim (Task Runner)

If you added `overseer.nvim` to your config:

| Key | Action |
|---|---|
| `Space + r + r` | Open task picker → select "Spring Boot Run" |
| `Space + r + t` | Toggle task output panel |

---

## 9. Running Angular

### Custom Keybindings (from angular.lua)

If you configured `~/.config/nvim/lua/plugins/angular.lua`:

| Key | Action |
|---|---|
| `<Leader>rs` | Run `npm start` (ng serve on localhost:4200) |
| `<Leader>rt` | Run `npm test` (Vitest / Karma / Jest) |
| `<Leader>rb` | Run `npm run build` (production build) |
| `<Leader>rg` | Run `ng generate ...` (prompts for what to generate) |
| `Ctrl + \` | Toggle terminal |

### Using ng generate from Neovim

Press `<Leader>rg` and type the Angular CLI schematic:

```
component features/my-feature      → creates a new component
service core/services/my-service    → creates a new service
pipe shared/pipes/my-pipe           → creates a new pipe
guard core/guards/auth              → creates a route guard
directive shared/directives/my-dir  → creates a directive
```

### Built-in Terminal (Alternative)

Use `Space + t + h` to open a terminal and run Angular commands manually:

```bash
npm start          # ng serve
npm test           # run tests
npm run build      # production build
ng generate component features/my-component
```

### Debugging Angular

1. Start the dev server: `<Leader>rs` (runs `npm start`)
2. Set breakpoints in your `.ts` files: `Space` `d` `b`
3. Start the debugger: `Space` `d` `c` → select **"Launch Chrome (Angular localhost:4200)"**
4. Chrome opens attached to the debugger
5. Interact with the app to trigger breakpoints
6. Use debug controls:

| Key | Action |
|---|---|
| `Space` `d` `c` | Continue |
| `Space` `d` `o` | Step over |
| `Space` `d` `i` | Step into |
| `Space` `d` `O` | Step out |
| `Space` `d` `u` | Toggle DAP UI (variables, call stack) |
| `Space` `d` `b` | Toggle breakpoint |

---

## 10. Daily Workflows

### Exploring an Unfamiliar Spring Boot Project

```
1. Space + e          → open file explorer, understand project structure
2. Space + l + S      → search for a class name (e.g. "UserService")
3. g + d              → jump into a dependency or method
4. g + r              → see everywhere that method is called
5. Ctrl + o           → go back to where you came from
6. K                  → read the docs on an annotation or method
```

### Adding a New Spring Boot Feature

```
1. Space + f + f      → open the Controller file
2. Space + \ |        → split vertically, open the Service file
3. Space + l + a      → generate method stub, organize imports
4. Space + l + r      → rename a method if needed (updates everywhere)
5. ] + d              → jump to any errors
6. Space + l + a      → fix imports or issues from the code action menu
7. Space + l + f      → format the file before committing
```

### Exploring an Unfamiliar Angular Project

```
1. Space + e          → open file explorer, look at src/app/features/
2. Space + f + f      → find a component file (e.g. "dashboard")
3. Space + \ |        → split vertically: .ts on left, .html on right
4. Space + l + s      → see document symbols (inputs, outputs, methods)
5. g + d              → jump to a service or model from the component
6. g + r              → see everywhere a service method is used
7. Ctrl + o           → go back
```

### Adding a New Angular Feature

```
1. <Leader>rg         → ng generate component features/my-feature
2. Space + f + f      → open the new component .ts file
3. Space + \ |        → split vertically, open the .html template
4. Write component logic in .ts, template in .html side by side
5. Space + l + a      → add missing imports, fix errors
6. g + d              → navigate to services/models you inject
7. <Leader>rs         → run ng serve to preview the feature
8. ] + d              → jump to any template or TypeScript errors
9. Space + l + f      → format both files before committing
```

### Refactoring an Angular Component

```
1. Space + f + g      → grep for the component selector (e.g. "app-dashboard")
2. g + d              → jump to the component class definition
3. Space + l + s      → see all inputs, outputs, and methods
4. Space + l + r      → rename a property (updates template + TS)
5. v + i + {          → visually select a block of logic
6. Space + l + a      → "Extract to function" or move to a service
7. Space + l + a      → organize imports
8. Space + l + f      → format with Prettier
9. Space + l + d      → check for any remaining errors
```

### Refactoring a Next.js Component

```
1. Space + f + g      → grep for the component name to find all usages
2. g + d              → jump to the component definition
3. v + i + {          → visually select the JSX block to extract
4. Space + l + a      → "Extract to function" or "Extract to new file"
5. Space + l + r      → rename the new component everywhere
6. Space + l + f      → format with Prettier
7. Space + l + d      → check for any remaining type errors
```

### Fixing Errors Quickly

```
1. Space + l + d      → open all diagnostics for the file
2. ] + d              → jump to first error
3. gl                 → read the full error message
4. Space + l + a      → apply suggested fix from code actions
5. ] + d              → jump to next error
6. repeat...
```

### Finding Where Something is Defined

```
Method or class you don't recognize?
→ hover over it → K (read the docs)
→ g + d         (go to definition)
→ g + i         (go to implementation if it's an interface)
→ g + r         (see all usages)
→ Ctrl + o      (come back)
```

---

## Quick Reference Card

### The 12 Keys You'll Use Every Day

| # | Key | Action |
|---|---|---|
| 1 | `Space + f + f` | Open any file |
| 2 | `Space + f + g` | Search text in project |
| 3 | `g + d` | Go to definition |
| 4 | `Ctrl + o` | Go back |
| 5 | `K` | Read documentation |
| 6 | `Space + l + a` | Code actions / refactor |
| 7 | `Space + l + r` | Rename symbol |
| 8 | `Space + l + f` | Format file |
| 9 | `] + d` | Next error |
| 10 | `Space + e` | File explorer |
| 11 | `<Leader>rs` | Run Angular/Next.js dev server |
| 12 | `Space + d + c` | Start/continue debugger |

---

*Guide based on AstroNvim v5+, nvim-jdtls (Java), vtsls (TypeScript), angularls (Angular) — March 2026*
