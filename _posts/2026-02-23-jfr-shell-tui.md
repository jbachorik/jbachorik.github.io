---
layout: post
title: "JFR Shell Gets a TUI: Because Scrolling Is for Mortals"
date: 2026-02-23
categories: [java, jfr, tui, performance]
tags: [jfr, jfr-shell, tui, terminal, profiling, jafar]
---

1. TOC
{:toc}

## The Problem with Scrolling

If you have used [jfr-shell](https://github.com/jbachorik/jafar) for any non-trivial analysis, you know the drill. You run a query, get a wall of text, scroll up to see the header, lose your place, scroll back down, squint at a column that wrapped awkwardly, then run the query again with `--limit 10` because your terminal buffer just swallowed the first thousand rows.

It works. It's fine. It's also the terminal equivalent of reading a spreadsheet by printing it on a receipt roll.

Starting with version **0.14.0**, jfr-shell ships with a full-screen terminal UI mode built on [TamboUI](https://github.com/jbachorik/tamboui), a ratatui-inspired widget framework for Java. Launch it with `--tui` and the scrolling stops. Everything stays put. You navigate, filter, drill down, and export - all without losing context.

## Getting In

```bash
jfr-shell --tui recording.jfr
```

That's it. Same queries, same JfrPath, same everything. The difference is that the output no longer scrolls off into the void.

<!-- SCREENSHOT: tui-overview.png
     Capture: Launch jfr-shell --tui with a recording loaded.
     Run a query like: events/jdk.ExecutionSample | groupBy(thread/name)
     Show the full TUI layout: status bar at top, table results in the middle,
     command input at the bottom, tips line and hints bar visible.
     Terminal should be at least 120x40 for a good shot. -->
![TUI overview]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-overview.png)

## The Layout

The screen is divided into a few fixed zones, top to bottom:

- **Status bar** - shows the active session and recording name
- **Results pane** - your query output, rendered as a table or tree
- **Command input** - where you type, with the familiar `jfr>` prompt
- **Tips & hints** - rotating usage tips and context-aware keyboard shortcuts

The results pane is the main attraction. It holds a navigable table with row selection, column sorting, and horizontal scrolling. No more piping into `less` and pretending that's ergonomic.

## Tabs

By default each command is rendered into a scratch-tab — it will get replaced by the next command. Pin the tab with `Ctrl+P` to prevent it from being overwritten. Pinned tabs stick around and you can switch between them with `{` and `}`.

<!-- SCREENSHOT: tui-tabs.png
     Capture: Run 3-4 different queries to create multiple tabs.
     Pin one tab (Ctrl+P). The pinned tab should show the pin icon.
     Focus should be on a tab that is NOT the first one, so the tab bar
     shows multiple tabs with the active one highlighted. -->
![Multiple tabs with pinning]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-tabs.png)

## Search and Filter

Press `/` and start typing. The table filters in real time - only rows containing your search term are shown, with matches highlighted in yellow. The status line shows the hit count. Press `Enter` to lock the filter, `Esc` to cancel.

This works the same way you would expect from any reasonable tool: case-insensitive, substring match, instant feedback.

<!-- SCREENSHOT: tui-search.png
     Capture: Run a query with many rows (eg. events/jdk.ExecutionSample | groupBy(thread/name)).
     Press / and type a partial thread name (e.g. "Fork" or "GC").
     Show the search bar active with the filter applied,
     matching rows visible, and the match count displayed. -->
![Live search and filter]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-search.png)

## Sorting

`<` and `>` cycle through sort columns. `Alt+r` reverses the sort direction. The currently sorted column is indicated in the header. Combined with filtering, this lets you answer questions like "which thread had the most samples" without writing a `top()` aggregation.

## The Detail Pane

Select a row and press `Enter`. The results pane splits: the table stays on the left (60%), and a detail view appears on the right (40%). The detail view shows the full structure of the selected row, rendered as a tree, with nested fields, annotations, and type information all expanded.

This is where the TUI earns its keep. In the old REPL, inspecting a complex event meant running `show metadata` in a separate query, cross-referencing field names, and hoping your mental model held. Now you just arrow down to a row and hit Enter.

<!-- SCREENSHOT: tui-detail.png
     Capture: Run a query that produces rows with complex/nested data.
     Good candidates: events/jdk.ExecutionSample (has stackTrace field),
     or events/jdk.ObjectAllocationSample.
     Select a row and press Enter to open the detail pane.
     Show the split view: table on left, detail tree on right. -->
![Detail pane with split view]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-detail.png)

Detail subtabs (switch with `[` and `]`) let you inspect different aspects of the selected row. `Shift+Tab` moves focus between the results table and the detail pane. The detail pane has its own scrolling, its own search (type `/` while focused there), and its own cursor navigation.

## Constant Pool Browser

The constant pool browser is one of those features that sounds niche until you need it - and then it's the only thing that matters.

Type `constants jdk.types.Symbol` (or any CP type) and the TUI opens a split view: a sidebar listing all constant pool types on the left, and the entries for the selected type on the right. Navigate the sidebar with arrow keys, press `Enter` to view entries. Press `/` in the sidebar to filter the type list.

For large constant pools, entries are paginated automatically so the UI stays responsive even when a recording has hundreds of thousands of entries.

<!-- SCREENSHOT: tui-browser.png
     Capture: Open a recording and run: constants jdk.types.Symbol
     (or another CP type that has many entries).
     Show the browser split: type sidebar on the left, entries on the right.
     Ideally with a few types visible in the sidebar. -->
![Constant pool browser]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-browser.png)

## Metadata Tree

`show metadata/type --tree` renders the recording's type system as a collapsible tree with Unicode box-drawing characters. Field types, dimensions, annotations - it's all there, structured instead of flattened into a table.

Add `--depth N` to control how deep the recursion goes. The tree renderer tracks visited types to avoid infinite loops on recursive structures.

<!-- SCREENSHOT: tui-metadata.png
     Capture: Run: show metadata/type --tree --depth 3
     Show the tree output with Unicode guide lines (├─ ┴ etc),
     field names, types, and annotations visible. -->
![Metadata tree view]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-metadata.png)

## Session Switching

Working with multiple recordings is common when doing comparative analysis - "before" and "after" a change, or multiple services in a distributed trace. `Alt+s` opens a session picker overlay. Arrow to the session you want, hit Enter. The results pane updates to show results from the new active session.

## Completion

`Tab` in the command input opens a completion popup. It's context-aware: event type names after `events/`, field paths in filters, function names in aggregations, file paths after `open`. Type to narrow the list, arrow keys to select, Enter to accept.

<!-- SCREENSHOT: tui-completion.png
     Capture: Type "events/jdk." in the command input and press Tab.
     Show the completion popup with event type suggestions filtered
     to the jdk namespace. -->
![Tab completion popup]({{ site.baseurl }}/assets/images/2026-02-23-jfr-shell-tui/tui-completion.png)

## Export

`Ctrl+E` exports the active tab's table data to a CSV file. A popup opens with a default path under `~/.jfr-shell/exports/`; edit it or just press Enter. Done. No more `--format csv > output.csv` and rebuilding the query from memory.

## History

`Ctrl+R` opens reverse-i-search, exactly like bash. Start typing and it finds the most recent matching command. `Ctrl+R` again jumps to the next match. `Enter` accepts, `Esc` cancels. Command history persists across sessions in `~/.jfr-shell/history`.

Combined with `Shift+Up/Down` for simple history scrolling, you never have to retype a query.

## Cell Picker

Press `@` to open a popup listing every field and value of the currently selected row. Arrow to the one you want, press Enter — the value is inserted into the command input and copied to the clipboard. Useful for grabbing a thread name, stack trace hash, or constant pool ID without retyping it.

## The Keyboard Cheat Sheet

The hints bar at the bottom of the screen adapts to the current focus. When you're in the results pane, it shows shortcuts like `↑↓:row  <>:sort col  /:search  Ctrl+P:pin`. When you're in the command input, it shows `Enter:run  Tab:complete  Ctrl+R:search history`. It's always telling you what's available without you having to memorize anything.

Here's the full set:

| Context | Key | Action |
|---------|-----|--------|
| Global | `{` / `}` | Switch tabs |
| Global | `Ctrl+P` | Pin/unpin tab |
| Global | `Ctrl+E` | Export to CSV |
| Global | `Ctrl+R` | History search |
| Global | `Alt+s` | Session picker |
| Results | `/` | Search/filter |
| Results | `<` / `>` | Sort by column |
| Results | `Alt+r` | Reverse sort |
| Results | `Enter` | Open detail pane |
| Results | `Alt+d` | Jump to detail |
| Results | `Shift+Tab` | Cycle focus |
| Detail | `[` / `]` | Switch subtabs |
| Detail | `/` | Search in detail |
| Detail | `Alt+r` | Jump to results |
| Input | `Tab` | Completion |
| Input | `@` | Cell picker |
| Input | `Alt+r` | Jump to results |
| Input | `Alt+c` | Jump to command |
| Search | `Ctrl+L` | Apply to both panes |

## Under the Hood

The TUI is built on [TamboUI](https://github.com/jbachorik/tamboui), a Java terminal UI framework inspired by Rust's [ratatui](https://ratatui.rs/). TamboUI provides the widget set (tables, trees, text inputs, tabs, scrollbars, blocks with borders), the constraint-based layout system, and the styling primitives. jfr-shell composes these into the full-screen application.

The rendering loop is simple: draw the frame, wait for input with a 100ms timeout, handle the keystroke, repeat. Commands execute asynchronously in a background thread, with a braille spinner animation in the status area while they run. Results stream into the active tab as they arrive.

The terminal backend is JLine-based, using raw mode for direct keystroke capture and the alternate screen buffer so your shell history stays clean when you exit.

No external GUI dependencies. No Electron. No web browser. Just ANSI escape codes and a well-organized widget tree.

## Try It

```bash
# JBang (easiest)
jbang jfr-shell@btraceio --tui recording.jfr

# Or build from source
./gradlew :jfr-shell:shadowJar
java -jar jfr-shell/build/libs/jfr-shell-*-all.jar --tui recording.jfr
```

If you have been using jfr-shell in REPL mode and getting by just fine, the TUI won't change what you can do. It changes how it feels to do it. Queries that used to involve scrolling, re-running, and squinting now involve pointing and pressing Enter.

The REPL is still there, unchanged, for scripts and non-interactive use. `--tui` is strictly additive.

---

*jfr-shell 0.14.0 is available on [Maven Central](https://central.sonatype.com/) and via [JBang](https://jbang.dev). Source on [GitHub](https://github.com/jbachorik/jafar).*
