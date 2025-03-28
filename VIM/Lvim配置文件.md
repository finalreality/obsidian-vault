~/.config/lvim/config.lua
```lua
-- Read the docs: https://www.lunarvim.org/docs/configuration
-- Example configs: https://github.com/LunarVim/starter.lvim
-- Video Tutorials: https://www.youtube.com/watch?v=sFA9kX-Ud_c&list=PLhoH5vyxr6QqGu0i7tt_XoVK9v-KvZ3m6
-- Forum: https://www.reddit.com/r/lunarvim/
-- Discord: https://discord.com/invite/Xb9B4Ny

lvim.leader = "\\"

lvim.tabstop = 4

vim.opt.tabstop = 4
vim.opt.shiftwidth = 4
vim.opt.softtabstop = 4
vim.opt.expandtab = false
vim.opt.relativenumber = true
vim.opt_local.formatoptions = vim.opt_local.formatoptions - { "r", "c", "o" }
vim.g.file_encodings = { "utf-8", "ucs-bom", "gb2312", "cp936", "cp932", "gb18030", "utf-16le", "utf-16be", "latin1" }

local dap = require('dap')
dap.adapters.lldb = {
	type = 'executable',
	command = '/usr/bin/lldb-vscode-16', -- adjust as needed, must be absolute path
	name = 'lldb'
}

local get_args = function()
	-- 获取输入命令行参数
	local cmd_args = vim.fn.input('Input commandline arguments:')
	local params = {}

	-- 定义分隔符(%s在lua内表示任何空白符号)
	local sep = "%s"
	for param in string.gmatch(cmd_args, "[^%s]+") do
		table.insert(params, param)
	end

	return params
end;

local function get_executable_from_cmake(path)
	-- 使用awk获取CMakeLists.txt文件内要生成的可执行文件的名字
	-- 有需求可以自己改成别的
	local get_executable = 'awk "BEGIN {IGNORECASE=1} /add_executable\\s*\\([^)]+\\)/ {match(\\$0, /\\(([^\\)]+)\\)/,m);match(m[1], /([A-Za-z_]+)/, n);printf(\\"%s\\", n[1]);}" ' .. path .. "CMakeLists.txt"
	return vim.fn.system(get_executable)
end

local dap = require('dap')
dap.configurations.cpp =
{
	{
		name = 'Launch',
		type = 'lldb',
		request = 'launch',

		program = function()
			local current_path = vim.fn.getcwd() .. "/"

			-- 使用find命令找到Makefile或者makefile
			local fd_make = string.format('find %s -maxdepth 1 -name [m\\|M]akefile', current_path)
			local fd_make_result = vim.fn.system(fd_make)

			if (fd_make_result ~= "")
			then
				local mkf = vim.fn.system(fd_make)
				-- 使用awk默认提取Makefile(makefile)中第一个的将要生成的可执行文件名称
				-- 有需求可以自己改成别的
				local cmd = 'awk "\\$0 ~ /:/ { match(\\$1, \\"([A-Za-z_]+)\\", m); printf(\\"%s\\", m[1]); exit; }" ' ..
				mkf
				local exe = vim.fn.system(cmd)
				-- 执行make命令
				-- Makefile里面需要设置CXXFLAGS变量哦~
				if (os.execute('make CXXFLAGS="-g"'))
				then
					return current_path .. exe
				end
			end

			-- 查找CMakeLists.txt文件
			local fd_cmake = string.format("find %s -name CMakeLists.txt -type f", current_path)
			local fd_cmake_result = vim.fn.system(fd_cmake)
			if (fd_cmake_result == "")
			then
				return vim.fn.input("Path to executable: ", current_path, "file")
			end

			-- 查找build文件夹
			local fd_build = string.format("find %s -name build -type d", current_path)
			local fd_build_result = vim.fn.system(fd_build)
			if (fd_build_result == "")
			then
				-- 不存在则创建build文件夹
				if (not os.execute(string.format('mkdir -p %sbuild', current_path)))
				then
					return vim.fn.input("Path to executable: ", current_path, "file")
				end
			end

			local cmd = 'cd ' .. current_path .. "build && cmake .. -DCMAKE_BUILD_TYPE=Debug"

			-- 开始构建项目
			print("Building The Project...")
			vim.fn.system(cmd)
			local exec = get_executable_from_cmake(current_path)
			local make = 'cd ' .. current_path .. 'build && make'
			local res = vim.fn.system(make)

			if (exec == "" or res == "")
			then
				return vim.fn.input("Path to executable: ", current_path, "file")
			end

			return current_path .. "build/" .. exec
		end,

		-- cwd = "${workspaceFolder}",
		-- stopOnEntry = false,
		-- args = get_args,


		program = function()
		  return vim.fn.input('Path to executable: ', vim.fn.getcwd() .. '/', 'file')
		end,
		cwd = '${workspaceFolder}',
		stopOnEntry = false,
		args = {},
	},
}
dap.configurations.c = dap.configurations.cpp

local dap = require('dap')

--[[
当然还有其他更多选项可以用啦，看标题就知道这只是简单配置一下，我的目前时间也不多就暂时能用就行，
主要还是向能脱离IDE
]]

dap.adapters.python = {
	type = "executable";
	command = '/usr/bin/python';
	args = { '-m', 'debugpy.adapter' };
}

local get_args = function()
	-- 获取输入命令行参数
	local cmd_args = vim.fn.input('CommandLine Args:')
	local params = {}
	-- 定义分隔符(%s在lua内表示任何空白符号)
	local sep = "%s"
	for param in string.gmatch(cmd_args, "[^%s]+") do
		table.insert(params, param)
	end
	return params
end;


dap.configurations.python = {
	{
		type = 'python';
		request = 'launch';
		name = 'launch file';
		-- 此处指向当前文件
		program = '${file}';
		args = get_args;
		pythonpath = function()
			return '/usr/bin/python'
		end;
	},
}


lvim.plugins = {
	{
		"theHamsta/nvim-dap-virtual-text",
		config = function()
			require("nvim-dap-virtual-text").setup({
				enabled = true,         -- enable this plugin (the default)
				enabled_commands = true, -- create commands DapVirtualTextEnable, DapVirtualTextDisable, DapVirtualTextToggle, (DapVirtualTextForceRefresh for refreshing when debug adapter did not notify its termination)
				highlight_changed_variables = true, -- highlight changed values with NvimDapVirtualTextChanged, else always NvimDapVirtualText
				highlight_new_as_changed = false, -- highlight new variables in the same way as changed variables (if highlight_changed_variables)
				show_stop_reason = true, -- show stop reason when stopped for exceptions
				commented = false,      -- prefix virtual text with comment string
				only_first_definition = true, -- only show virtual text at first definition (if there are multiple)
				all_references = false, -- show virtual text on all all references of the variable (not only definitions)
				clear_on_continue = false, -- clear virtual text on "continue" (might cause flickering when stepping)
				--- A callback that determines how a variable is displayed or whether it should be omitted
				--- @param variable Variable https://microsoft.github.io/debug-adapter-protocol/specification#Types_Variable
				--- @param buf number
				--- @param stackframe dap.StackFrame https://microsoft.github.io/debug-adapter-protocol/specification#Types_StackFrame
				--- @param node userdata tree-sitter node identified as variable definition of reference (see `:h tsnode`)
				--- @param options nvim_dap_virtual_text_options Current options for nvim-dap-virtual-text
				--- @return string|nil A text how the virtual text should be displayed or nil, if this variable shouldn't be displayed
				display_callback = function(variable, buf, stackframe, node, options)
					-- by default, strip out new line characters
					if options.virt_text_pos == 'inline' then
						return ' = ' .. variable.value:gsub("%s+", " ")
					else
						return variable.name .. ' = ' .. variable.value:gsub("%s+", " ")
					end
				end,
				-- position of virtual text, see `:h nvim_buf_set_extmark()`, default tries to inline the virtual text. Use 'eol' to set to end of line
				virt_text_pos = vim.fn.has 'nvim-0.10' == 1 and 'inline' or 'eol',

				-- experimental features:
				all_frames = false, -- show virtual text for all stack frames not only current. Only works for debugpy on my machine.
				virt_lines = false, -- show virtual lines instead of virtual text (will flicker!)
				virt_text_win_col = nil -- position the virtual text at a fixed window column (starting from the first text column) ,
				-- e.g. 80 to position at column 80, see `:h nvim_buf_set_extmark()`
			})
		end,
	},
	{
		"nvimdev/lspsaga.nvim",
		after = 'nvim-lspconfig',
		config = function()
			require('lspsaga').setup({
				debug = false,
				use_saga_diagnostic_sign = true,
				-- diagnostic sign
				error_sign = "",
				warn_sign = "",
				hint_sign = "",
				infor_sign = "",
				diagnostic_header_icon = "   ",
				-- code action title icon
				code_action_icon = " ",
				code_action_prompt = {
					enable = true,
					sign = true,
					sign_priority = 40,
					virtual_text = true,
				},
				finder_definition_icon = "  ",
				finder_reference_icon = "  ",
				max_preview_lines = 10,
				finder_action_keys = {
					open = "o",
					vsplit = "s",
					split = "i",
					quit = "q",
					scroll_down = "<C-f>",
					scroll_up = "<C-b>",
				},
				code_action_keys = {
					quit = "q",
					exec = "<CR>",
				},
				rename_action_keys = {
					quit = "<C-c>",
					exec = "<CR>",
				},
				definition_preview_icon = "  ",
				border_style = "single",
				rename_prompt_prefix = "➤",
				rename_output_qflist = {
					enable = false,
					auto_open_qflist = false,
				},
				server_filetype_map = {},
				diagnostic_prefix_format = "%d. ",
				diagnostic_message_format = "%m %c",
				highlight_prefix = false,
			})
		end,
	},
	{
		"ethanholz/nvim-lastplace",
		event = "BufRead",
		config = function()
			require("nvim-lastplace").setup({
				lastplace_ignore_buftype = { "quickfix", "nofile", "help" },
				lastplace_ignore_filetype = {
					"gitcommit", "gitrebase", "svn", "hgcommit",
				},
				lastplace_open_folds = true,
			})
		end,
	},
	{
		"folke/persistence.nvim",
		event = "BufReadPre", -- this will only start session saving when an actual file was opened
		config = function()
			require("persistence").setup {
				dir = vim.fn.expand(vim.fn.stdpath "config" .. "/session/"),
				options = { "buffers", "curdir", "tabpages", "winsize" },
			}
		end,
	},
	{
		"ahmedkhalf/lsp-rooter.nvim",
		event = "BufRead",
		config = function()
			require("lsp-rooter").setup()
		end,
	},
	{
		"ray-x/lsp_signature.nvim",
		config = function()
			require "lsp_signature".setup({
			})
		end,
	},
	{
		"wakatime/vim-wakatime"
	},
	{
		"ggandor/lightspeed.nvim",
		event = "BufRead",
	},
	{
		"npxbr/glow.nvim",
		ft = { "markdown" }
		-- build = "yay -S glow"
	},
	{
		"iamcco/markdown-preview.nvim",
		build = "cd app && npm install",
		ft = "markdown",
		config = function()
			vim.g.mkdp_auto_start = 1
		end,
	},
}

lvim.builtin.bufferline.options.numbers = "buffer_id"

lvim.lsp.buffer_mappings.normal_mode['gd'] = nil
vim.keymap.set('n', 'gd', '<cmd>Lspsaga peek_definition<CR>')
lvim.lsp.buffer_mappings.normal_mode['gD'] = nil
vim.keymap.set('n', 'gD', '<cmd>lua vim.lsp.buf.definition()<CR>')
lvim.lsp.buffer_mappings.normal_mode['gh'] = nil
vim.keymap.set('n', 'gh', '<cmd>Lspsaga finder<CR>')
lvim.lsp.buffer_mappings.normal_mode['gr'] = nil
vim.keymap.set('n', 'gr', '<cmd>Lspsaga rename<CR>')

lvim.lsp.buffer_mappings.normal_mode['gb'] = nil
vim.keymap.set('n', 'gb', '<cmd>Lspsaga supertypes<CR>')

lvim.lsp.buffer_mappings.normal_mode['gi'] = nil
vim.keymap.set('n', 'gi', '<cmd>Lspsaga subtypes<CR>')

vim.keymap.set('n', '<leader>lo', '<cmd>Lspsaga outline<CR>')

vim.api.nvim_set_var("lspsaga_definition_preview_quit", "<ESC>") -- 可选：设置退出预览的快捷键
vim.api.nvim_set_var("lspsaga_hover_delay", 300) -- 悬停提示防抖
vim.api.nvim_set_var("lspsaga_rename_confirm_save_timeout", 500) -- 重命名确认延迟

lvim.builtin.treesitter.ensure_installed = { --这是已有的, 修改为你需要的语言即可
	"bash",
	"vim",
	"lua",
	"c",
	"hpp",
	"cpp",
	"cmake",
	"go",
	"python",
	"javascript",
	"typescript",
	"tsx",
	"html",
	"css",
	"markdown",
	"json",
	"yaml",
}

-- persistence.nvim
vim.keymap.set('n', '<leader>as', function() require("persistence").load() end)
vim.keymap.set('n', '<leader>aS', function() require("persistence").select() end)
vim.keymap.set('n', '<leader>al', function() require("persistence").load({ last = true }) end)
vim.keymap.set('n', '<leader>ad', function() require("persistence").stop() end)

-- lsp_signature.nvim
lvim.lsp.on_attach_callback = function(client, bufnr)
	require "lsp_signature".on_attach()
end

-- switch header and source
vim.keymap.set('n', '<leader>aa', ':ClangdSwitchSourceHeader<CR>', { noremap = true, silent = true })

lvim.builtin.which_key.mappings["dT"] = {
	"<cmd>:DapVirtualTextToggle<CR>", "Toggle VirtualText"
}

local glow = require('glow').setup({
	style = "dark",
	width = 120
})
```