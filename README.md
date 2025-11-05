-- ==============================================================================
-- Termux专属Neovim配置（Python开发终极稳定版）
-- 核心功能：语法高亮+文件树+LSP补全+自动格式化+终端运行+剪贴板+主题
-- 已修复问题：nvim-tree配置错误、字符串未闭合、格式化失败、主题适配
-- ==============================================================================

-- 1. 前置环境检测（解决依赖缺失，避免后续崩溃）
local function check_termux_deps()
  local deps = {
    { cmd = "git",       msg = "pkg install git -y",       tip = "Lazy插件管理器依赖" },
    { cmd = "python3",   msg = "pkg install python -y",    tip = "Python开发核心" },
    { cmd = "python3 -m pip", msg = "pkg install python-pip -y", tip = "Black格式化工具依赖" }
  }

  for _, dep in ipairs(deps) do
    if vim.fn.system(dep.cmd .. " --version"):find("not found") then
      vim.notify(
        string.format("❌ 缺少%s依赖！\n请在Termux执行：%s", dep.tip, dep.msg),
        vim.log.levels.ERROR
      )
      return false
    end
  end
  return true
end
-- 环境不满足则终止配置，避免无效加载
if not check_termux_deps() then return end

-- 2. Lazy.nvim安装（Termux专属优化：浅克隆提速，失败提示清晰）
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.notify("🔄 正在安装Lazy.nvim（约1分钟，耐心等待）...", vim.log.levels.INFO)
  local install_cmd = {
    "git", "clone", "--filter=blob:none", "--depth=1",
    "https://github.com/folke/lazy.nvim.git", "--branch=stable", lazypath
  }
  local result = vim.fn.system(install_cmd)
  if vim.v.shell_error ~= 0 then
    vim.notify(
      string.format("❌ Lazy安装失败！手动执行：\n%s", table.concat(install_cmd, " ")),
      vim.log.levels.ERROR
    )
    return
  end
  vim.notify("✅ Lazy.nvim安装成功！", vim.log.levels.SUCCESS)
end
vim.opt.rtp:prepend(lazypath)

-- 3. 插件配置（所有插件符合最新语法，花括号成对闭合）
require("lazy").setup({
  -- ==================== 主题模块（优先加载，避免界面闪烁） ====================
  {
    "folke/tokyonight.nvim",
    lazy = false,
    priority = 1000,
    config = function()
      require("tokyonight").setup({
        style = "night", -- 暗色模式，适配Termux
        transparent = false, -- 关闭透明（避免文字模糊）
        terminal_colors = true, -- 适配终端颜色
        styles = {
          comments = { italic = false }, -- 关闭斜体（Termux字体渲染问题）
          keywords = { italic = false },
          functions = { bold = false },
        },
        dim_inactive = true, -- 弱化非活跃窗口，突出编辑区
        hide_inactive_statusline = true, -- 隐藏非活跃状态栏，省空间
      })
      vim.cmd("colorscheme tokyonight-night") -- 应用主题
    end,
  },

  {
    "nvim-lualine/lualine.nvim", -- 主题配套状态栏
    lazy = true,
    event = "BufEnter",
    dependencies = { "nvim-tree/nvim-web-devicons" },
    config = function()
      require("lualine").setup({
        options = {
          theme = "tokyonight", -- 与主题配色同步
          globalstatus = true, -- 单状态栏，适配小屏
          disabled_filetypes = { "NvimTree" }, -- 文件树隐藏状态栏
          component_separators = { left = "│", right = "│" },
          section_separators = { left = "", right = "" },
        },
        sections = {
          lualine_a = { "mode" },
          lualine_b = { "branch" },
          lualine_c = { { "filename", path = 1 } }, -- 显示相对路径
          lualine_x = { "filetype" },
          lualine_y = { "progress" },
          lualine_z = { "location" },
        },
      })
    end,
  },

-- ==================== 文件管理（低版本 nvim-tree 最终兼容配置） ====================
{
  "nvim-tree/nvim-tree.lua",
  lazy = false,
  priority = 500,
  config = function()
    require("nvim-tree").setup({
      view = {
        width = 28,
        signcolumn = "no",
      },
      actions = {
        open_file = {
          quit_on_open = true,
          resize_window = false
        }
      },
      renderer = {
        indent_markers = { enable = true },
        icons = { show = { file = false, folder = false, git = false } }
      },
      filters = { custom = { "^.git$", "^__pycache__$", "*.pyc" } },
      -- 关键：修正低版本 API 方法名，避免 rhs 为 nil
      on_attach = function(bufnr)
        local api = require("nvim-tree.api")
        local keymap = vim.keymap.set
        local opts = { buffer = bufnr, noremap = true, silent = true, nowait = true }

        -- 1. 修正“关闭文件树”：低版本用 api.tree.toggle 或直接调用 vim.cmd
        keymap("n", "q", function() api.tree.toggle() end, opts) -- q 切换（关闭）文件树
        keymap("n", "<ESC>", function() api.tree.toggle() end, opts) -- ESC 切换（关闭）

        -- 2. 修正“打开/关闭节点”：确保方法名存在
        if api.node.open.edit then
          keymap("n", "l", api.node.open.edit, opts) -- l 打开文件（存在则绑定）
          keymap("n", "<CR>", api.node.open.edit, opts) -- 回车打开文件
        else
          keymap("n", "l", api.node.open, opts) -- 低版本兼容：用 api.node.open
          keymap("n", "<CR>", api.node.open, opts)
        end

        if api.node.close then
          keymap("n", "h", api.node.close, opts) -- h 关闭节点（存在则绑定）
        else
          keymap("n", "h", function() vim.cmd("normal! h") end, opts) -- 兼容：用原生 h
        end

        -- 3. 保留默认快捷键（若存在，避免覆盖）
        if api.config.mappings.default_on_attach then
          api.config.mappings.default_on_attach(bufnr)
        end
      end
    })

    -- 全局函数：切换文件树（兼容低版本 API）
    _G.toggle_nvim_tree = function()
      local api = require("nvim-tree.api")
      local dir = vim.fn.expand("%:p") == "" and vim.fn.getcwd() or vim.fn.expand("%:p:h")
      -- 低版本用 api.tree.open 或 toggle，避免路径参数冲突
      if api.tree.toggle then
        api.tree.toggle({ path = dir })
      else
        api.tree.open({ path = dir })
      end
    end
  end,
},

-- ==================== 基础编辑增强 ====================
{
  "nvim-treesitter/nvim-treesitter", -- Python语法高亮+自动缩进
  -- 原有配置不变...
},

-- 新增：括号补全插件（轻量适配Termux）
{
  "windwp/nvim-autopairs",
  lazy = true,
  event = "InsertEnter", -- 插入模式时加载，不占用启动资源
  config = function()
    -- 基础配置：自动补全括号、引号、大括号等
    require("nvim-autopairs").setup({
      check_ts = true, -- 结合 Treesitter 智能补全（如Python函数括号）
      ts_config = {
        python = { "string", "comment" }, -- Python中只在字符串/注释外补全
      },
      disable_filetype = { "TelescopePrompt", "NvimTree" }, -- 特定窗口禁用（避免干扰）
      fast_wrap = {
        map = "<M-e>", -- Alt+e 快速包裹选中内容（触控可忽略，可选配置）
        chars = { "{", "[", "(", '"', "'" },
        pattern = string.gsub([[ [%'%"%)%>%]%)%}%,] ]], "%s+", ""),
        offset = 0, -- 从当前光标位置开始
        end_key = "$",
        keys = "qwertyuiopasdfghjklzxcvbnm",
        check_comma = true,
        highlight = "PmenuSel",
        highlight_grey = "LineNr",
      },
    })

    -- 关键：让补全的括号能被退格键删除（如输入"("后按退格，自动删除配对的")"）
    local cmp_autopairs = require("nvim-autopairs.completion.cmp")
    local cmp = require("cmp")
    cmp.event:on("confirm_done", cmp_autopairs.on_confirm_done())
  end,
  dependencies = { "hrsh7th/nvim-cmp" }, -- 依赖cmp，实现补全后自动配对
},

{
  "nvim-telescope/telescope.nvim", -- 文件查找（原有配置不变...）
},


  -- ==================== 基础编辑增强 ====================
  {
    "nvim-treesitter/nvim-treesitter", -- Python语法高亮+自动缩进
    build = ":TSUpdate python",
    lazy = true,
    event = "BufReadPost *.py",
    config = function()
      require("nvim-treesitter.configs").setup({
        ensure_installed = { "python" }, -- 强制安装Python语法支持
        highlight = { enable = true },
        indent = { enable = true }, -- 关键：Python自动缩进
        incremental_selection = { enable = false } -- 禁用高资源功能
      })
    end
  },

  -- ==================== Python开发核心 ====================
  {
    "williamboman/mason.nvim", -- LSP包管理
    lazy = true,
    cmd = "Mason",
    config = function()
      require("mason").setup({
        install_root_dir = vim.fn.stdpath("data") .. "/mason", -- 固定路径，避免权限问题
        ui = { border = "single", icons = { package_installed = "✓", package_pending = "⟳" } }
      })
    end
  },

  {
    "hrsh7th/nvim-cmp", -- Python代码补全
    lazy = true,
    event = "InsertEnter *.py",
    dependencies = { "hrsh7th/cmp-nvim-lsp" },
    config = function()
      local cmp = require("cmp")
      cmp.setup({
        enabled = function()
          -- 仅Python文件+插入模式启用，减少资源占用
          return vim.bo.filetype == "python" and vim.api.nvim_get_mode().mode:sub(1,1) == "i"
        end,
        mapping = cmp.mapping.preset.insert({
          ["<Tab>"] = cmp.mapping.select_next_item(),
          ["<S-Tab>"] = cmp.mapping.select_prev_item(),
          ["<CR>"] = cmp.mapping.confirm({ select = true }),
          ["<ESC>"] = cmp.mapping.abort() -- 快速关闭，适配触控
        }),
        sources = { { name = "nvim_lsp" } },
        window = {
          completion = cmp.config.window.bordered({
            winhighlight = "Normal:Normal,FloatBorder:Border",
            max_width = 35 -- 限制宽度，不遮挡编辑区
          })
        },
        performance = { debounce = 1500 } -- 适配Termux低性能
      })
    end
  },

  {
    "stevearc/conform.nvim", -- Python自动格式化（极简配置，避免冲突）
    lazy = true,
    event = "BufWritePre *.py",
    config = function()
      -- 检测Black是否安装（简单命令，适配Termux）
      local black_installed = vim.fn.system("command -v black") ~= ""
      if not black_installed then
        vim.notify(
          "⚠️ 未安装Black格式化工具！\n请在Termux执行：pip install black --user",
          vim.log.levels.WARN
        )
        return
      end

      -- 调用Conform内置Black配置，避免手动路径错误
      require("conform").setup({
        formatters_by_ft = { python = { "black" } },
        format_on_save = { timeout_ms = 20000 }, -- 适配Termux慢速度
        log_level = vim.log.levels.ERROR -- 只显示错误，减少干扰
      })
    end
  },

  -- ==================== 文件查找 ====================
  {
    "nvim-telescope/telescope.nvim",
    dependencies = { "nvim-lua/plenary.nvim" },
    lazy = true,
    cmd = "Telescope",
    config = function()
      require("telescope").setup({
        defaults = {
          previewer = false, -- 禁用预览，Termux加载快不卡顿
          layout_config = { width = 0.9, height = 0.7 }, -- 适配小屏
          sorting_strategy = "ascending", -- 结果从上到下，触控易选
          mappings = { i = { ["<ESC>"] = "close" }, n = { ["q"] = "close" } },
          file_ignore_patterns = { "__pycache__/", "venv/", "*.pyc" } -- 过滤Python无用文件
        }
      })
    end
  }
}) -- Lazy插件列表闭合（所有插件配置结束）

-- 4. Pyright LSP配置（手动安装+低性能优化）
local function setup_pyright()
  -- 手动安装Pyright命令（避免启动时自动下载卡顿）
  vim.api.nvim_create_user_command("InstallPyright", function()
    local registry_ok, registry = pcall(require, "mason-registry")
    if not registry_ok then
      vim.notify("❌ 请先执行 :Mason 加载包管理器！", vim.log.levels.ERROR)
      return
    end
    if not registry.is_installed("pyright") then
      vim.notify("🔄 正在安装Pyright（约5分钟，勿退出）...", vim.log.levels.INFO)
      registry.get_package("pyright"):install()
    else
      vim.notify("✅ Pyright已安装！", vim.log.levels.INFO)
    end
  end, { desc = "Termux中手动安装Python LSP（Pyright）" })

  -- Python文件打开时自动启动LSP（增加路径容错）
  vim.api.nvim_create_autocmd("FileType", {
    pattern = "python",
    callback = function()
      local pyright_cmd = { vim.fn.stdpath("data") .. "/mason/bin/pyright-langserver", "--stdio" }
      -- 检测Pyright是否存在，不存在则提示手动安装
      if vim.fn.filereadable(pyright_cmd[1]) ~= 1 then
        vim.notify("❌ Pyright未安装！执行 :InstallPyright 安装", vim.log.levels.WARN)
        return
      end

      vim.lsp.start({
        name = "pyright",
        cmd = pyright_cmd,
        capabilities = require("cmp_nvim_lsp").default_capabilities(),
        settings = {
          python = {
            analysis = {
              autoSearchPaths = true,
              typeCheckingMode = "off", -- 关闭类型检查，减少Termux卡顿
              reportMissingImports = false, -- 忽略虚拟环境导入报错
              diagnosticMode = "openFilesOnly" -- 只检查当前文件，省资源
            },
            pythonPath = vim.fn.exepath("python3") -- 固定用系统Python路径
          }
        },
        on_attach = function(client)
          client.server_capabilities.documentFormattingProvider = false -- 禁用LSP格式化（避免与Conform冲突）
          client.log_level = vim.log.levels.OFF -- 关闭LSP日志，节省资源
        end,
        flags = { debounce_text_changes = 2000 } -- 延长响应，适配低性能设备
      })
    end
  })
end
setup_pyright()

-- 5. Termux触控友好快捷键（所有绑定完整，无语法错误）
local keymap = vim.keymap.set
local opts = { noremap = true, silent = true }

-- F2: 切换文件树（调用全局函数，避免API未初始化）
keymap("n", "<F2>", function()
  if _G.toggle_nvim_tree then
    _G.toggle_nvim_tree()
  else
    vim.notify("❌ 文件树未初始化！请重启Neovim", vim.log.levels.ERROR)
  end
end, vim.tbl_extend("force", opts, { desc = "F2: 切换文件树" }))

-- F7: 手动格式化Python文件（保存失败时备用）
keymap("n", "<F7>", function()
  if vim.bo.filetype ~= "python" then
    vim.notify("❌ 仅支持Python文件格式化！", vim.log.levels.WARN)
    return
  end
  local ok, conform = pcall(require, "conform")
  if not ok then
    vim.notify("❌ 格式化插件未加载！请重启Neovim", vim.log.levels.ERROR)
    return
  end
  conform.format({
    name = "black",
    async = false,
    timeout_ms = 20000,
    callback = function(err)
      if err then
        vim.notify(string.format("❌ 格式化失败：%s", err), vim.log.levels.ERROR)
      else
        vim.notify("✅ Python文件格式化成功！", vim.log.levels.INFO)
      end
    end
  })
end, vim.tbl_extend("force", opts, { desc = "F7: 手动格式化Python文件" }))
-- F12: 运行当前Python脚本（Termux终端专属优化）
keymap("n", "<F12>", function()
  local filetype = vim.bo.filetype
  local file_path = vim.fn.expand("%:p")

  -- 三重校验，避免无效操作
  if filetype ~= "python" then
    vim.notify("❌ 仅Python文件可运行！", vim.log.levels.WARN)
    return
  end
  if file_path == "" then
    vim.notify("❌ 请先保存文件（:w）再运行！", vim.log.levels.ERROR)
    return
  end
  if vim.fn.filereadable(file_path) ~= 1 then
    vim.notify("❌ 文件不存在或已删除！", vim.log.levels.ERROR)
    return
  end

  -- 复用终端，减少触控切换
  local run_cmd = string.format("python3 '%s'", vim.fn.escape(file_path, "'"))
  local term_buf = nil
  for _, buf in ipairs(vim.api.nvim_list_bufs()) do
    if vim.bo[buf].filetype == "terminal" and vim.api.nvim_buf_is_loaded(buf) then
      term_buf = buf
      break
    end
  end

  -- 固定终端位置（下方10行，不遮挡编辑区）
  if term_buf then
    local win = vim.fn.bufwinid(term_buf)
    if win == -1 then
      vim.cmd("belowright 10split")
      vim.api.nvim_set_current_buf(term_buf)
    else
      vim.api.nvim_set_current_win(win)
    end
  else
    vim.cmd("belowright 10split | terminal")
  end

  -- 发送命令，增加终端就绪校验
  if vim.b.terminal_job_id then
    vim.fn.chansend(vim.b.terminal_job_id, "clear\n" .. run_cmd .. "\n")
    vim.cmd("startinsert") -- 自动进入插入模式，直接查看输出
  else
    vim.notify("❌ 终端未就绪！请重试", vim.log.levels.ERROR)
  end
end, vim.tbl_extend("force", opts, { desc = "F12: 运行当前Python脚本" }))

-- Ctrl+F: 全局文件查找
keymap("n", "<C-f>", function()
  local ok, builtin = pcall(require, "telescope.builtin")
  if not ok then
    vim.notify("❌ 查找插件未加载！执行 :Telescope 试试", vim.log.levels.ERROR)
    return
  end
  builtin.find_files({ hidden = true, prompt_title = "🔍 查找文件" })
end, vim.tbl_extend("force", opts, { desc = "Ctrl+F: 全局文件查找" }))

-- Ctrl+S: 全局内容查找
keymap("n", "<C-s>", function()
  local ok, builtin = pcall(require, "telescope.builtin")
  if not ok then
    vim.notify("❌ 查找插件未加载！执行 :Telescope 试试", vim.log.levels.ERROR)
    return
  end
  builtin.live_grep({ prompt_title = "🔍 查找内容" })
end, vim.tbl_extend("force", opts, { desc = "Ctrl+S: 全局内容查找" }))

-- 6. 剪贴板配置（完整无语法错误）
vim.opt.clipboard:append("unnamedplus") -- 启用系统剪贴板，支持跨应用复制

-- 复制（Visual/Normal模式）
local function copy(visual_mode)
  local content = visual_mode and vim.fn.getvisualselection() or vim.fn.getline(".")
  if content and content ~= "" then
    vim.fn.setreg("+", content) -- 复制到系统剪贴板
    vim.notify(visual_mode and "✅ 复制选中内容到剪贴板" or "✅ 复制当前行到剪贴板", vim.log.levels.INFO)
  else
    vim.notify(visual_mode and "❌ 无选中内容可复制" or "❌ 当前行无内容可复制", vim.log.levels.WARN)
  end
end
-- 绑定复制快捷键
keymap("v", "<C-c>", function() copy(true) end, { desc = "Ctrl+C: 复制选中内容" })
keymap("n", "<C-c>", function() copy(false) end, { desc = "Ctrl+C: 复制当前行" })

-- 剪切（Visual/Normal模式）
local function cut(visual_mode)
  local content = visual_mode and vim.fn.getvisualselection() or vim.fn.getline(".")
  if content and content ~= "" then
    vim.fn.setreg("+", content) -- 剪切到系统剪贴板
    if visual_mode then
      vim.cmd('normal! "_x') -- 用黑洞寄存器删除，不污染默认寄存器
    else
      vim.cmd("normal! dd") -- 正常删除当前行
    end
    vim.notify(visual_mode and "✅ 剪切选中内容到剪贴板" or "✅ 剪切当前行到剪贴板", vim.log.levels.INFO)
  else
    vim.notify(visual_mode and "❌ 无选中内容可剪切" or "❌ 当前行无内容可剪切", vim.log.levels.WARN)
  end
end
-- 绑定剪切快捷键
keymap("v", "<C-x>", function() cut(true) end, { desc = "Ctrl+X: 剪切选中内容" })
keymap("n", "<C-x>", function() cut(false) end, { desc = "Ctrl+X: 剪切当前行" })

-- 粘贴（Normal/Insert模式）
local function paste()
  local content = vim.fn.getreg("+") -- 从系统剪贴板获取内容
  if content and content ~= "" then
    return content
  end
  vim.notify("❌ 系统剪贴板为空，无法粘贴", vim.log.levels.WARN)
  return ""
end
-- 绑定粘贴快捷键
keymap("n", "<C-y>", function()
  local content = paste()
  if content ~= "" then vim.cmd("normal! p") end -- 正常模式粘贴到光标后
end, { desc = "Ctrl+Y: 正常模式粘贴（光标后）" })
keymap("i", "<C-y>", function()
  local content = paste()
  if content ~= "" then vim.api.nvim_put({ content }, "c", false, true) end -- 插入模式粘贴
end, { desc = "Ctrl+Y: 插入模式粘贴" })

-- 7. Termux基础体验优化（适配手机小屏+低性能）
vim.opt.number = true -- 显示行号，触控定位方便
vim.opt.relativenumber = false -- 关闭相对行号，节省屏幕空间
vim.opt.signcolumn = "no" -- 隐藏符号列，增加编辑区域宽度
vim.opt.wrap = true -- 自动换行，避免横向滚动
vim.opt.scrolloff = 2 -- 滚动时保留2行上下文，减少视觉跳跃
vim.opt.cmdheight = 1 -- 命令行高度设为1行，节省空间
vim.opt.updatetime = 500 -- 缩短LSP响应时间，适配低性能设备
vim.opt.cursorline = false -- 关闭光标行高亮，减少资源占用

