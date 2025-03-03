# nvim-fzf

A lua api for using fzf in neovim >= 0.5.

Allow for full asynchronicity for UI speed and usability.

Supports Linux and MacOS.

## Usage

```lua
local nvim_fzf = require("fzf")

coroutine.wrap(function()
  local result = nvim_fzf.fzf({"choice 1", "choice 2"}, "--preview 'cat
  {}'")  
  -- result is a list of lines that fzf returns, if the user has chosen
  if result then
    print(result[1])
  end
end)()
```

## Table of contents

* [Usage](#usage)
* [Installation](#installation)
* [Important information](#important-information)
* [API Functions](#api-functions)
* [Main API](#main-api)
* [Action API](#action-api)
* [Examples](#examples)
* [How it works](#how-it-works)
* [Todo](#todo)

## Installation

```vimscript
Plug 'vijaymarupudi/nvim-fzf'
```

## Important information

**You should run all the functions in this module in a coroutine. This
allows for an easy api**

Example: 

```lua
local nvim_fzf = require("fzf")

coroutine.wrap(function()
  local result = nvim_fzf.fzf({"choice 1", "choice 2"})  
  if result then
    print(result[1])
  end
end)()
```

## API Functions

Require this plugin using `local nvim_fzf = require('fzf')`


* `nvim_fzf.raw_fzf(contents, options)`

  An fzf function that runs fzf in the current window. See `Main API`
  for more details about the general API.

* `nvim_fzf.provided_win_fzf(contents, options)`

  Runs fzf in the current window, and closes it after the user has
  chosen. Allows for the user to provide the fzf window.

  ```lua
  -- for a vertical fzf
  vim.cmd [[ vertical new ]]
  nvim_fzf.provided_win_fzf(contents, options)
  ```

* `nvim_fzf.fzf(contents, options)`

  An fzf function that opens a centered floating window and closes it
  after the user has chosen.


## Main API

`fzf(contents, options)`

* `contents`

  * if **string**: a shell command 

    ```lua
    local result = fzf("fd")
    ```

  * if **table**: a list of strings or string convertables

    ```lua
    local result = fzf({1, 2, "item"})
    ```

  * if **function**: calls the function with a callback function to
    write vals to the fzf pipe. This api is asynchronous, making it
    possible to use fzf for long running applications and making the
    user interface snappy. Callbacks can be concurrent.

    * `cb(value, finished_cb)`

      * `value`: A value to write to fzf
      * `finished_cb(err)`: A callback called with an err if there is an
        error writing the value to fzf. This can occur if the user has
        already picked a value in fzf.

    ```lua
    local result = fzf(function(cb)
      cb("value_1", function(err)
        -- this error can happen if the user can already chosen a value
        -- using fzf
        if err then
          return
        end
        cb("value_2", function(err)
          if err then
            return
          end
          cb(nil) -- to close the pipe to fzf, this removes the loading
                  -- indicated in fzf
        end)
      end)
    end)
    ```

* `options`: **string**, A list of command line options for fzf.

    Can use to expect different key bindings (e.g. `--expect
    ctrl-t,ctrl-v`), previews, and coloring.


* **return value**: **table**, the lines that fzf returns in the shell
  as a table. If not lines are returned by fzf, the function returns nil
  for an easy conditional check.

  ```lua
  local result = fzf("fd")
  if result then
    -- do something with result[1]
  end
  ```

  ```lua
  local result = fzf("fd", "--multi")
  if result then
    -- do something with result[1] to result[#result]
  end
  ```

  ```lua
  local result = fzf("fd", "--expect=ctrl-t")
  if result then
    if result[1] == "ctrl-t" then
      -- do something with result[2]
    else
      -- do something with result[2]
    end
  end
  ```

## Action API

Sometimes you want to use neovim information in fzf (such as previews of
non file buffers, bindings to delete buffers, or change colorschemes).
fzf expects a shell command for these parameters. Making your own shell
command and setting up RPC can be cumbersome. This plugin provides an
easy API to run a lua function / closure in response to these actions.

```lua
local fzf = require "fzf".fzf
local action = require "fzf.actions".action

coroutine.wrap(function()
  -- items is a table of selected or hovered fzf items
  local shell = action(function(items, fzf_lines, fzf_cols)
    -- only one item will be hovered at any time, so get the selection
    -- out and convert it to a number
    local buf = tonumber(items[1])

    -- you can return either a string or a table to show in the preview
    -- window
    return vim.api.nvim_buf_get_lines(buf, 0, -1, false)
  end)

  fzf(vim.api.nvim_list_bufs(), "--preview " .. shell)
end)()
```

`require("fzf.actions").action(fn)`

* `fn(selections, fzf_lines, fzf_cols)`: A function that takes a
  selection, performs an action, and optionally returns either a `table`
  or `string` to print to stdout.

  * `selections`: a `table` of strings selected in fzf
  * `fzf_lines`: number of lines in the preview window i.e.
    `$FZF_PREVIEW_LINES`
  * `fzf_cols`: number of cols in the preview window i.e.
    `$FZF_PREVIEW_COLS`

* **return value**: a shell-escaped string to append to the fzf options
    for fzf to run.

## Examples

**Filetype picker**

```lua
local fts = {
  "typescript",
  "javascript",
  "lua",
  "python",
  "vim",
  "markdown",
  "sh"
}


coroutine.wrap(function()
  local choice = require "fzf".fzf(fts)
  if choice then
    vim.cmd(string.format("set ft=%s", choice[1]))
  end
end)()
```

**Helptags picker** (asynchronous, uses `--expect`)

```lua
local runtimepaths = vim.api.nvim_list_runtime_paths()
local uv = vim.loop
local fzf = require('fzf').fzf

local function readfilecb(path, callback)
  uv.fs_open(path, "r", 438, function(err, fd)
    if err then
      callback(err)
      return
    end
    uv.fs_fstat(fd, function(err, stat)
      if err then
        callback(err)
        return
      end
      uv.fs_read(fd, stat.size, 0, function(err, data)
        if err then
          callback(err)
          return
        end
        uv.fs_close(fd, function(err)
          if err then
            callback(err)
            return
          end
          return callback(nil, data)
        end)
      end)
    end)
  end)
end

local function readfile(name)
  local co = coroutine.running()
  readfilecb(name, function (err, data)
    coroutine.resume(co, err, data)
  end)
  local err, data = coroutine.yield()
  if err then error(err) end
  return data
end

local function deal_with_tags(tagfile, cb)
  local co = coroutine.running()
  coroutine.wrap(function ()
    local success, data = pcall(readfile, tagfile)
    if success then
      for i, line in ipairs(vim.split(data, "\n")) do
        local items = vim.split(line, "\t")
        -- escape codes for grey
        local tag = string.format("%s\t\27[0;37m%s\27[0m", items[1], items[2])
        local co = coroutine.running()
        cb(tag, function ()
          coroutine.resume(co)
        end)
        coroutine.yield()
      end
    end
    coroutine.resume(co)
  end)()
  coroutine.yield()
end

local fzf_function = function (cb)
  local total_done = 0
  for i, rtp in ipairs(runtimepaths) do
    local tagfile = table.concat({rtp, "doc", "tags"}, "/")
    -- wrapping to make all the file reading concurrent
    coroutine.wrap(function ()
      deal_with_tags(tagfile, cb)
      total_done = total_done + 1 
      if total_done == #runtimepaths then
        cb(nil)
      end
    end)()
  end
end

coroutine.wrap(function ()
  local result = fzf(fzf_function, "--nth 1 --ansi --expect=ctrl-t,ctrl-s,ctrl-v") 
  if not result then
    return
  end
  local choice = vim.split(result[2], "\t")[1]
  local key = result[1]
  local windowcmd
  if key == "" or key == "ctrl-s" then
    windowcmd = ""
  elseif key == "ctrl-v" then
    windowcmd = "vertical"
  elseif key == "ctrl-t" then
    windowcmd = "tab"
  else
    print("Not implemented!")
    error("Not implemented!")
  end

  vim.cmd(string.format("%s h %s", windowcmd, choice))
end)()
```

## How it works

This plugin uses the `mkfifo` posix program to get a temporary named
pipe, and uses it to communicate to fzf.

Contributions welcome to make this compatible with Windows. I do not
have a Windows machine, so I cannot test it. It should be possible using
the Luajit FFI and the
[`CreateNamedPipeA`](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-createnamedpipea)
function from the win32 api.

## Todo

Currently this plugin allows for neovim to speak to fzf. A similar
technique can be used to get fzf to speak to neovim, for use in previews
and actions. Contributions welcome!
