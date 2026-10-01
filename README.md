
thread_status = {
    update_world = { last_run = 0, count = 0, status = "Stopped" },
    update_npcs   = { last_run = 0, count = 0, status = "Stopped" },
    scan_players_inv = { last_run = 0, count = 0, status = "Stopped" },
    slf_scan      = { last_run = 0, count = 0, status = "Stopped" },
    update_bosses = { last_run = 0, count = 0, status = "Stopped" },
}


console_log_buffer = {}
console_log_buffer_max = 200
console_messages = {}
console_win = console_win or {}
console_win.content = {}
console_max_lines = 7        
console_win = {
    x = 710,
    y = 10,
    w = 500,
    title = "Console",
    visible = false,
    dragging = false,
    drag_ox = 0,
    drag_oy = 0,
    content = {}  
}
function addConsoleMessage(text, color)
    color = color or {1, 1, 1, 1}
    local timestamp = os.date("%H:%M:%S")
    local full_text = "[" .. timestamp .. "] " .. text

    if #console_messages > 0 then
        local last = console_messages[#console_messages]

        if last.text == text then
            last.count = (last.count or 1) + 1
            last.display_text = full_text .. " (x" .. last.count .. ")"

            if console_log_buffer and #console_log_buffer > 0 then
                local last_log = console_log_buffer[#console_log_buffer]
                if last_log.text == text then
                    last_log.count = (last_log.count or 1) + 1
                    last_log.time = timestamp
                    return
                end
            end

            return
        end
    end

    table.insert(console_messages, {
        text = text,                 
        full_text = full_text,       
        color = color,
        count = 1,
        display_text = full_text     
    })
    if #console_messages > console_max_lines then
        table.remove(console_messages, 1)
    end
    if console_log_buffer then
        table.insert(console_log_buffer, {
            time = timestamp,
            text = text,
            count = 1,
        })
        if #console_log_buffer > (console_log_buffer_max or 200) then
            table.remove(console_log_buffer, 1)
        end
    end
end
function drawConsoleWindow()
    if not menu.get("show_console") then
        console_win.visible = false
        console_win.dragging = false
        return
    end
    console_win.visible = true

    
    local content = {}
    for _, msg in ipairs(console_messages) do
        table.insert(content, {
            text = msg.display_text or msg.full_text,
            color = msg.color
        })
    end
    if #content == 0 then
        table.insert(content, { text = "No messages", color = {0.45, 0.45, 0.50, 1} })
    end
    console_win.content = content

    drawStyledConsoleWindow(console_win)
end
function drawStyledConsoleWindow(win)
    if not win.visible then
        win.dragging = false
        return
    end

    local content = win.content or {}
    local num_lines = #content
    local px, py = win.x, win.y
    local pw = win.w
    local title_h = 24
    local pad = 8
    local line_h = 16
    local panel_h = title_h + pad + (num_lines * line_h) + pad + 6


    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    if lmb then
        if not win.dragging then
            if mx >= px - 2 and mx <= px + pw + 2 and my >= py - 2 and my <= py + title_h + 6 then
                win.dragging = true
                win.drag_ox = mx - px
                win.drag_oy = my - py
            end
        else
            win.x = mx - win.drag_ox
            win.y = my - win.drag_oy
            px, py = win.x, win.y
        end
    else
        win.dragging = false
    end

    local font        = SLF_FONT or 13
    local font_s      = SLF_FONT_S or 11

    draw.Rect(px-1, py-1, pw+2, panel_h+2, STYLE.border, 5, 1)
    draw.RectFilled(px, py, pw, panel_h, STYLE.bg_main, 4)
    draw.RectFilled(px, py, pw, title_h, STYLE.bg_title , 4)
    draw.RectFilled(px, py, 2, title_h, STYLE.border, 0)
    draw.Line(px, py + title_h, px + pw, py + title_h, STYLE.border, 1)
    draw.Rect(px, py, pw, panel_h, STYLE.border, 4, 1)

    draw.Text(px + pad + 2, py + 5, win.title, STYLE.accent, font)
    local hint = "drag ::"
    local hw = draw.GetTextSize(hint, 10)
    draw.Text(px + pw - hw - pad, py + 6, hint, STYLE.text_dim, 10)

    local ly = py + title_h + pad
    for i, entry in ipairs(content) do
        local color = entry.color or {1,1,1,1}
        draw.Text(px + pad, ly, entry.text, color, font_s)
        ly = ly + line_h
        if i < num_lines then
            --draw.Line(px + pad, ly + 1, px + pw - pad, ly + 1, col_sep, 1)
        end
    end
end
local old_print = print
print = function(...)
    local args = {...}
    local msg = table.concat(args, " ")
    old_print(...) 
    addConsoleMessage(msg, {0.8, 0.8, 0.9, 1})  
end


-- ===== PDH.FUNCT ===== --

CONFIG_SLOTS = {"Main", "Rage", "DEBUG"}
loaded_config_name = "none"

---------------------------

-- ===== NO RECOIL ===== --
norecoil_active = false
norecoil_timer_id = nil
norecoil_status = "OFF"
norecoil_nodes = 0
norecoil_message = ""

norecoil_keys = {
    "RecoilPatternLastTick",
    "RecoilPatternRecoverySpeed",
    "RecoilReduction",
    "MaxRecoil",
    "RecoilTValueMax",
    "RecoilReductionMax",
    "RecoilTValue",
    "RecoilPatternPos"
}

function start_norecoil()
    if norecoil_active then return end
    norecoil_active = true
    norecoil_status = "STARTING"
    norecoil_message = "Searching for tables..."
    if norecoil_timer_id then
        thread.Stop(norecoil_timer_id)
        norecoil_timer_id = nil
    end

    norecoil_timer_id = thread.Create(function()
        if not norecoil_active then return end
        norecoil_status = "RUNNING"
        local found = false
        refreshgc()
        local n = getgc(norecoil_keys)
        norecoil_nodes = n
        if n > 0 then
            local strength = menu.get("norecoil_strength") or 100
            local factor = strength / 100
            local base = 5
            local patches = {}
            for _, key in ipairs(norecoil_keys) do
                patches[key] = math.floor(base * (1 - factor) + 0.5)
            end
            applygc(patches)
            norecoil_status = "ACTIVE"
            norecoil_message = "Strength: " .. strength .. "%  |  Value: " .. (base * (1 - factor))
            print("[NoRecoil] Applied to", n, "nodes, strength", strength, "%, value", (base * (1 - factor)))
            addNotification("[NoRecoil] Applied", "success", 6)
            addConsoleMessage("[NoRecoil] Applied", {0,1,0,1})
            found = true
            if norecoil_timer_id then
                thread.Stop(norecoil_timer_id)
                norecoil_timer_id = nil
            end
        else
            norecoil_status = "WAITING"
            norecoil_message = "No tables found, retrying..."
            addConsoleMessage("[NoRecoil] No tables found, retrying...", {0, 1, 1, 1})
        end
    end, 500)
end

function stop_norecoil()
    if not norecoil_active then
        --addConsoleMessage("[NoRecoil] Not active, nothing to stop", {0.7,0.7,0.7,1})
        return
    end
    norecoil_active = false
    norecoil_status = "STOPPING"
    norecoil_message = "Disabling..."
	--addConsoleMessage("[NoRecoil][info] Stopped", {0, 1, 1, 1})
    if norecoil_timer_id then
        thread.Stop(norecoil_timer_id)
        norecoil_timer_id = nil
    end
    norecoil_status = "OFF"
    norecoil_message = "Disabled"
    addConsoleMessage("[NoRecoil] Disabled", {0,1,0.3,1})
end

function main()

if not math.clamp then
    math.clamp = function(v, min, max) return math.max(min, math.min(max, v)) end
end
if not math.round then
    math.round = function(v) return math.floor(v + 0.5) end
end


STYLE = {
    bg_main        = {0.09, 0.02, 0.09, 0.96},
    bg_title       = {0.07, 0.07, 0.10, 1.00},
    border         = {0.58, 0.22, 0.84, 0.5},
    accent         = {0.78, 0.42, 0.84, 0.8},
    accent_light   = {0.65, 0.40, 0.95, 0.8},
    text_light     = {1.00, 1.00, 1.00, 0.90},
    text_dim       = {0.25, 0.25, 0.30, 1.00},
    text_group     = {0.90, 0.85, 1.00, 0.95},
    hover_bg       = {0.25, 0.25, 0.35, 0.9},
    toggle_off     = {0.09, 0.02, 0.09, 0.96},
    toggle_on      = {0.58, 0.22, 0.84, 0.3},
    knob           = {1.0, 1.0, 1.0, 1.0},
    slider_track   = {0.20, 0.15, 0.30, 0.7},
    slider_fill    = {0.55, 0.30, 0.90, 1.0},
    combo_bg       = {0.15, 0.10, 0.22, 0.8},
    combo_hover    = {0.25, 0.18, 0.35, 0.9},
    button_bg      = {0.08, 0.06, 0.14, 0.95},
    button_hover   = {0.65, 0.40, 0.95, 0.3},
    group_bg       = {0.12, 0.08, 0.20, 0.6},
    group_border   = {0.48, 0.28, 0.80, 0.4},
    dropdown_bg    = {0.08, 0.06, 0.14, 0.95},
	label_col	   = {0.55, 0.55, 0.55, 1.00},
	tab_def		   = {0.55, 0.20, 0.84, 0.88},
	QB_COL_OFF     = { 0.09, 0.02, 0.09, 0.96 },
	QB_COL_ON      = { 0.58, 0.22, 0.84, 0.3 },
	QB_COL_LINE    = { 0.58, 0.22, 0.84, 1.0 },
	QB_COL_ICON    = { 1.00, 1.00, 1.00, 0.70 },
	QB_COL_BG_ICON = { 1.00, 1.00, 1.00, 0.20 }
}

local WIN_W = 600
local TITLE_H = 28
local PAD = 10
local TAB_H = 28
local FONT = 12
local FONT_S = 11

local CTRL_WIDTHS = {
    toggle = 50,
    slider = 110,
    combo = 90,
    keybind = 90,
    button = 100,
    colorpicker = 60,
}

win = {
    x = 660,
    y = 300,
    w = WIN_W,
    h = 460,
    title = "PDH v2.0",
    visible = true,
    dragging = false,
    drag_ox = 0,
    drag_oy = 0,
}

local active_tab = 1
tabs = {}
states = {}
local click_handled = false

local toggle_anim = {}
local ANIM_SPEED = 0.12

local slider_drag = nil

local combo_open = nil
local combo_dropdown_pos = nil

local keybind_waiting = nil

local colorpicker_open = nil
local colorpicker_panel_pos = nil

local menu_toggle_key = 0x50
local menu_key_was_down = false

local function distributeGroups(groups)
    local left = {}
    local right = {}
    local full = {}
    for _, group in ipairs(groups) do
        local side = group.side
        if side == "left" then
            table.insert(left, group)
        elseif side == "right" then
            table.insert(right, group)
        elseif side == "full" then
            table.insert(full, group)
        else
            table.insert(left, group)
        end
    end
    return left, right, full
end

local function initTabs()
    tabs = {
			--== <PLAYER> ==--
        {
            name = "Player",
            groups = {
                {
                    title = "Player Esp",
                    side = "left",
                    items = {
                        { type = "toggle", id = "delta_player_enabled", label = "Enable Player ESP", default = false },

                        { type = "toggle", id = "delta_esp_enabled", label = "Chams", default = false, parent = "delta_player_enabled" },
						{ type = "toggle", id = "esp_player_box", label = "2D Box", default = false, parent = "delta_player_enabled" },
						{ type = "combo", id = "esp_box_style", label = "Box Style", options = {"Normal", "Corner"}, default = 0, parent = "delta_player_enabled" },
						{ type = "colorpicker", id = "esp_box_color", label = "Box Color", default = {1,0,0,1}, parent = "delta_player_enabled" },
						{ type = "toggle", id = "esp_skeleton", label = "Skeleton", default = false, parent = "delta_player_enabled" },
						{ type = "colorpicker", id = "esp_skeleton_color", label = "Skeleton Color", default = {1, 1, 1, 1}, parent = "delta_player_enabled" },
                        { type = "toggle", id = "delta_player_dist", label = "Player Distance", default = false, parent = "delta_player_enabled", colorpicker_default = {0.6058,0.7497,1.0000,1.0000} },
                        { type = "slider", id = "delta_player_range", label = "Player Range (m)", min = 10, max = 1500, step = 10, default = 500, suffix = "m", parent = "delta_player_enabled" }, 
                    }
                },
{
                    title = "Radar",
                    side = "left",
                    items = {
                        { type = "toggle", id = "delta_radar_enabled", label = "Enable Radar", default = false },
						--{ type = "label", label = "", parent = "delta_radar_enabled" },
						{ type = "slider", id = "radar_range", label = "Radar Range (m)", min = 100, max = 800, step = 10, default = 500, suffix = "m", parent = "delta_radar_enabled" },              
                    }
                },
                {
                    title = "Inventory Checker",
                    side = "right",
                    items = {
                        { type = "toggle", id = "delta_inv_enabled", label = "Inventory Checker", default = false, key_default = 2 },
                        { type = "combo", id = "delta_inv_mode", label = "Show Mode", options = {"Always", "Toggle", "Hold"}, default = 0, parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_guns", label = "Show Guns", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_guns", label = "Max Gun Lines", min = 1, max = 20, step = 1, default = 10, suffix = "", parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_sights", label = "Show Attachments", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_sights", label = "Max Attachments Lines", min = 1, max = 20, step = 1, default = 10, suffix = "", parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_armor", label = "Show armor", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_armor", label = "Max armor Lines", min = 1, max = 20, step = 1, default = 10, suffix = "", parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_val", label = "Show dorogo", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_val", label = "Max Show dorogo Lines", min = 1, max = 20, step = 1, default = 10, suffix = "", parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_serun", label = "Show Serum-X", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_serun", label = "Max Show Serum-X Lines", min = 1, max = 4, step = 1, default = 4, suffix = "", parent = "delta_inv_enabled" },
                        { type = "toggle", id = "delta_inv_show_other", label = "Show Other", default = false, parent = "delta_inv_enabled" },
                        { type = "slider", id = "delta_inv_max_other", label = "Max Other Lines", min = 1, max = 50, step = 1, default = 5, suffix = "", parent = "delta_inv_enabled" },
                    }
                },
            }
        },
			--== <COMBAT> ==--
        {
            name = "Combat",
            groups = {
                {
            	title = "Crosshair Settings",
            	side = "left",
            	items = {
                	{ type = "toggle", id = "crosshair_enabled", label = "Enable Crosshair", default = false },
                	{ type = "combo", id = "crosshair_style", label = "Style", options = {"Dot", "Dot + 4 Lines", "Dot + T-Shape"}, default = 0, parent = "crosshair_enabled" },
                	{ type = "slider", id = "crosshair_dot_size", label = "Dot Size", min = 1, max = 16, step = 1, default = 4, suffix = "px", parent = "crosshair_enabled" },
               		{ type = "slider", id = "crosshair_line_len", label = "Line Length", min = 2, max = 30, step = 1, default = 10, suffix = "px", parent = "crosshair_enabled" },
               	 	{ type = "slider", id = "crosshair_line_gap", label = "Gap (dot to line)", min = 0, max = 20, step = 1, default = 4, suffix = "px", parent = "crosshair_enabled" },
                	{ type = "slider", id = "crosshair_line_thick", label = "Line Thickness", min = 1, max = 6, step = 1, default = 2, suffix = "px", parent = "crosshair_enabled" },
                	{ type = "colorpicker", id = "crosshair_color", label = "Color", default = {0, 1, 0, 1}, parent = "crosshair_enabled" },

          			}
       			},
        		{
            		title = "No Recoil",
            		side = "left",
           			items = {
						{ type = "toggle", id = "norecoil_enabled", label = "No Recoil", default = false },
						{ type = "slider", id = "norecoil_strength", label = "Strength", min = 0, max = 100, step = 1, default = 0, suffix = "%" },
						{ type = "label", label = "<< standard recoi = 0 | WMod recoil = 100 >>", color = {0.60, 0.50, 1, 1} },

           			}
       			},
        		{
            		title = "Silent Aim",
            		side = "right",
            		items = {
                		{ type = "toggle", id = "silent_enabled", label = "Enable Silent Aim", default = false },
                		{ type = "keybind", id = "silent_key", label = "Aim Key", default = 0x01, parent = "silent_enabled" },
                		{ type = "combo", id = "silent_bone", label = "Target Bone", options = {"Head", "UpperTorso", "LowerTorso"}, default = 0, parent = "silent_enabled" },
                		{ type = "slider", id = "silent_fov", label = "FOV Radius", min = 50, max = 500, step = 10, default = 150, suffix = "px", parent = "silent_enabled" },
                		{ type = "toggle", id = "silent_visible_check", label = "Check Visibility", default = false, parent = "silent_enabled" },
                		{ type = "toggle", id = "silent_draw_fov", label = "Show FOV Circle", default = false, parent = "silent_enabled" },
               		 	{ type = "colorpicker", id = "silent_fov_color", label = "FOV Color", default = {1,1,1,0.3}, parent = "silent_enabled" },
                		{ type = "slider", id = "silent_prediction", label = "Prediction (ms)", min = 0, max = 500, step = 10, default = 0, suffix = "ms", parent = "silent_enabled" },
            		}
        		},
        		{
           		 	title = "Player Aimbot",
           			side = "right",
           		 	items = {
                		{ type = "toggle", id = "pl_aim_enabled", label = "Enable Aimbot", default = false },
                		{ type = "keybind", id = "pl_aim_key", label = "Aim Key", default = 0x01, parent = "pl_aim_enabled" },
                		{ type = "combo", id = "pl_aim_bone", label = "Target Bone", options = {"Head", "UpperTorso", "LowerTorso"}, default = 0, parent = "pl_aim_enabled" },
               		    { type = "slider", id = "pl_aim_fov", label = "FOV Radius", min = 50, max = 500, step = 10, default = 150, suffix = "px", parent = "pl_aim_enabled" },
               		 	{ type = "slider", id = "pl_aim_smooth", label = "Smoothing", min = 1, max = 20, step = 1, default = 5, suffix = "", parent = "pl_aim_enabled" },
                		{ type = "toggle", id = "pl_aim_prediction", label = "Velocity Prediction", default = false, parent = "pl_aim_enabled" },
                		{ type = "toggle", id = "pl_aim_draw_fov", label = "Show FOV Circle", default = false, parent = "pl_aim_enabled" },
                		{ type = "colorpicker", id = "pl_aim_fov_color", label = "FOV Color", default = {1,1,1,0.3}, parent = "pl_aim_enabled" },
                		{ type = "toggle", id = "pl_aim_visible_check", label = "Check Visibility", default = false, parent = "pl_aim_enabled" },
            }
        },
            }
        },
			--== <NPC> ==--
        {
            name = "NPC",
            groups = {
                {
                    title = "NPC Esp",
                    side = "left",
                    items = {
                        { type = "toggle", id = "delta_npc_enabled", label = "Enable NPC ESP", default = false },
                        { type = "toggle", id = "delta_npc_box", label = "Box", default = false, parent = "delta_npc_enabled", colorpicker_default = {1, 0.3, 0.3, 1} },
						{ type = "colorpicker", id = "delta_npccol_box", label = "box Color", default = {0.55,0.30,0.90,0.7}, parent = "delta_npc_enabled" },
                        { type = "toggle", id = "delta_npc_health", label = "Health Bar", default = false, parent = "delta_npc_enabled" },
                        { type = "toggle", id = "delta_npc_name", label = "Name", default = false, parent = "delta_npc_enabled", colorpicker_default = {1, 0.3, 0.3, 1} },
						{ type = "colorpicker", id = "delta_npccol_name", label = "Name Color", default = {0.55,0.30,0.90,0.7}, parent = "delta_npc_enabled" },
                        { type = "toggle", id = "delta_npc_dist", label = "NPC Distance", default = false, parent = "delta_npc_enabled", colorpicker_default = {0.7, 0.7, 0.7, 1} },
						{ type = "colorpicker", id = "delta_npccol_dist", label = "Distance Color", default = {0.55,0.30,0.90,0.7}, parent = "delta_npc_enabled" },
                        { type = "toggle", id = "delta_npc_skeleton", label = "Skeleton", default = false, parent = "delta_npc_enabled", colorpicker_default = {1, 1, 1, 1} },
						{ type = "colorpicker", id = "delta_npccol_skeleton", label = "Skeleton Color", default = {0.55,0.30,0.90,0.7}, parent = "delta_npc_enabled" },
                        { type = "slider", id = "delta_npc_range", label = "NPC Range (m)", min = 10, max = 600, step = 10, default = 140, suffix = "m", parent = "delta_npc_enabled" },
                    }
                },
                {
                    title = "NPC Aimbot",
                    side = "right",
                    items = {
                        { type = "toggle", id = "delta_npc_aim_enabled", label = "Enable NPC Aimbot", default = false, key_default = 2 },
						{ type = "keybind", id = "delta_npc_aimkey_enabled", label = "Aim Key", default = 0x01, parent = "delta_npc_aim_enabled" },
                        { type = "slider", id = "delta_npc_aim_fov", label = "FOV Radius", min = 50, max = 500, step = 10, default = 150, suffix = "px", parent = "delta_npc_aim_enabled" },
                        { type = "slider", id = "delta_npc_aim_smooth", label = "Smoothing", min = 1, max = 20, step = 1, default = 5, suffix = "", parent = "delta_npc_aim_enabled" },
                        { type = "combo", id = "delta_npc_aim_bone", label = "Target Bone", options = {"Head", "Chest", "Pelvis"}, default = 0, parent = "delta_npc_aim_enabled" },
                        { type = "toggle", id = "delta_npc_aim_prediction", label = "Velocity Prediction", default = false, parent = "delta_npc_aim_enabled" },
                        { type = "toggle", id = "delta_npc_aim_draw_fov", label = "Show FOV Circle", default = false, parent = "delta_npc_aim_enabled" },
                    }
                },
            }
        },
			--== <WORLD> ==--
        {
            name = "World",
            groups = {
                {
                    title = "World ESP",
                    side = "full",
                    items = {
                        { type = "toggle", id = "delta_extract_enabled", label = "Extracts", default = false, colorpicker_default = {0, 1, 0.5, 1} },
						{ type = "colorpicker", id = "delta_extract_col", label = "Color", default = {0, 1, 0, 1}, parent = "delta_extract_enabled" },
                        { type = "slider", id = "delta_extract_range", label = "Extract Range (m)", min = 10, max = 1500, step = 10, default = 500, suffix = "m", parent = "delta_extract_enabled" },
                        { type = "toggle", id = "delta_container_enabled", label = "Containers", default = false, colorpicker_default = {0.5, 0.5, 1, 1} },
                        { type = "toggle", id = "delta_container_contents", label = "Show Contents", default = false, parent = "delta_container_enabled" },
                        { type = "slider", id = "delta_container_range", label = "Container Range (m)", min = 10, max = 600, step = 10, default = 84, suffix = "m", parent = "delta_container_enabled" },
                        { type = "toggle", id = "delta_quest_enabled", label = "Quest Items", default = false, colorpicker_default = {1, 0.5, 1, 1} },
                        { type = "slider", id = "delta_quest_range", label = "Quest Range (m)", min = 10, max = 1500, step = 10, default = 140, suffix = "m", parent = "delta_quest_enabled" },
                        { type = "toggle", id = "delta_items_enabled", label = "Dropped Items", default = false, colorpicker_default = {1, 0.8, 0, 1} },
                        { type = "slider", id = "delta_items_range", label = "Item Range (m)", min = 10, max = 600, step = 10, default = 56, suffix = "m", parent = "delta_items_enabled" },
                        { type = "toggle", id = "delta_corpse_enabled", label = "Corpses", default = false, colorpicker_default = {1, 0.2, 0.2, 1} },
                        { type = "slider", id = "delta_corpse_range", label = "Corpse Range (m)", min = 10, max = 600, step = 10, default = 140, suffix = "m", parent = "delta_corpse_enabled" },
                        { type = "toggle", id = "delta_claymore_enabled", label = "Claymores", default = false, colorpicker_default = {1, 0.3, 0.1, 1} },
                        { type = "toggle", id = "delta_claymore_names", label = "Show Name + Distance", default = false, parent = "delta_claymore_enabled" },
                        { type = "slider", id = "delta_claymore_range", label = "Claymore Range (m)", min = 10, max = 600, step = 10, default = 140, suffix = "m", parent = "delta_claymore_enabled" },
                    }
                },
            }
        },
			--== <MISC> ==--
        {
            name = "Misc",
            groups = {
                {
                    title = "Misc",
                    side = "left",
                    items = {
                        { type = "toggle", id = "delta_boss_enabled", label = "Enable Boss Tracker", default = false },
                        { type = "toggle", id = "delta_mod_alert_enabled", label = "Mod Alert", default = false },
                        { type = "combo", id = "delta_mod_alert_mode", label = "Show Mode", options = {"Always", "Mods Only"}, default = 1, parent = "delta_mod_alert_enabled" },
                        { type = "toggle", id = "delta_slf_enabled", label = "Enable Loot Finder", default = false },
                    }
                },
                {
                    title = "Config",
                    side = "right",
                    items = {
                        { type = "button", id = "delta_save_config", label = "Save Config", callback = function() save_config() end },
                        { type = "button", id = "delta_load_config", label = "Load Config", callback = function() load_config() end },
						{ type = "keybind", id = "menu_toggle_key", label = "Toggle Menu Key", default = 0x50 },
						{ type = "combo", id = "config_slot", label = "Config Slot", options = CONFIG_SLOTS, default = 0 },
                    }
                },
                {
                    title = "Debug",
                    side = "right",
                    items = {
						--{ type = "label", label = "", color = {0.78, 0.42, 0.84, 0.8} },
                        { type = "toggle", id = "delta_debug_enabled", label = "Enable DEBUG", default = false },
						{ type = "toggle", id = "show_console", label = "Show Console", default = true },
						{ type = "button", id = "clear_console", label = "Clear Console", parent = "show_console", callback = function()
    						console_messages = {}
    						addConsoleMessage("[CONSOLE] cleared", {0.5, 0.5, 1, 1})
						end },
						{ type = "button", id = "test2", label = "testnotif", parent = "show_console", callback = function()
            				addNotification("[TEST] TEST", "test", 22)
						end },
						{ type = "toggle", id = "show_settings_window", label = "Settings Window", default = true },
                    }
                },
       		   {

                    title = "Logs",
                    side = "left",
                    items = {
						{ type = "button", id = "save_log", label = "Save Log File", callback = function() save_logs() end },
						{ type = "button", id = "clear_log", label = "Clear Log", callback = function()
    						logs = {}
    						addConsoleMessage("[LOG] cleared", {0.5, 0.5, 1, 1})
						end },
                    }
                },
            }
        },
    }

    for _, tab in ipairs(tabs) do
        for _, group in ipairs(tab.groups or {}) do
            for _, item in ipairs(group.items) do
				 if item.type == "label" then
            	elseif item.type == "toggle" then
                    states[item.id] = item.default
                    toggle_anim[item.id] = {
                        current = item.default and 1 or 0,
                        target = item.default and 1 or 0,
                    }
                elseif item.type == "slider" then
                    states[item.id] = item.default
                elseif item.type == "combo" then
                    states[item.id] = item.default
                elseif item.type == "keybind" then
                    states[item.id] = item.default
					states[item.id .. "_key"] = item.default
                elseif item.type == "colorpicker" then
                    states[item.id] = item.default
                elseif item.type == "input" then
                    states[item.id] = item.default
                end
                if item.colorpicker_default then
                    states[item.id .. "_color"] = item.colorpicker_default
                end
                if item.key_default then
                    states[item.id .. "_key"] = item.key_default
                end
            end
        end
    end
end
initTabs()

-- ============================================
-- SETTINGS: данные 4 вкладок
-- ============================================
SETTINGS_TABS = {
    {
        name = "Visual",
        icon = "Main",
        groups = {
            {
                title = "Box Style", side = "full",
                items = {
                    { type = "colorpicker", id = "menu_accent",    label = "Menu Button",    default = {0.58, 0.22, 0.84, 0.3} },
					{ type = "colorpicker", id = "menu_bg",        label = "Main Background", default = {0.09, 0.02, 0.09, 0.96} },
					{ type = "colorpicker", id = "menu_border",    label = "Main Border",    default = {0.58, 0.22, 0.84, 0.5} },
					{ type = "colorpicker", id = "menu_title_bg",  label = "Title Background", default = {0.07, 0.07, 0.10, 1.00} },
					{ type = "colorpicker", id = "menu_text",  label = "Text color", default = {0.64, 0.39, 1.00, 1.00} },
                }
            },
        }
    },
    {
        name = "INFO",
        icon = "",
        groups = {
            {
                title = "INFO", side = "full",
                items = {
					{ type = "label", label = "TEST MAKAR GAMING TEXT!", color = {1.0, 0.00, 0, 1} },
                }
            },
        }
    },
}

-- Регистрируем все items в tabs, чтобы drawSlider/drawCombo/drawKeybind/drawButton их нашли
local settings_lookup = { name = "__settings__", hidden = true, groups = {} }
for _, stab in ipairs(SETTINGS_TABS) do
    for _, g in ipairs(stab.groups) do
        table.insert(settings_lookup.groups, g)
    end
end
table.insert(tabs, settings_lookup)

-- Инициализируем states для новых виджетов
for _, g in ipairs(settings_lookup.groups) do
    for _, item in ipairs(g.items) do
        if item.type == "toggle" then
            states[item.id] = item.default or false
            toggle_anim[item.id] = { current = item.default and 1 or 0, target = item.default and 1 or 0 }
        elseif item.type == "slider" or item.type == "combo" or item.type == "keybind" then
            states[item.id] = item.default or 0
        elseif item.type == "colorpicker" then
            states[item.id] = item.default or {1,1,1,1}
            states[item.id .. "_color"] = item.default or {1,1,1,1}
        end
    end
end


local function col_alpha(col, a)
    return {col[1], col[2], col[3], col[4] * a}
end

local function sign(x)
    if x > 0 then return 1 elseif x < 0 then return -1 else return 0 end
end

local function clamp(v, min, max)
    return math.max(min, math.min(max, v))
end

local function updateToggleAnimations()
    for id, anim in pairs(toggle_anim) do
        if anim.current ~= anim.target then
            local diff = anim.target - anim.current
            local step = sign(diff) * math.min(math.abs(diff), ANIM_SPEED)
            anim.current = anim.current + step
            if math.abs(anim.current - anim.target) < 0.001 then
                anim.current = anim.target
            end
        end
    end
end

local function getVisibleItems(group)
    local visible = {}
    for _, item in ipairs(group.items) do
        local parent = item.parent
        if parent and not states[parent] then
        else
            table.insert(visible, item)
        end
    end
    return visible
end

local function getSliderValueFromMouse(item, mouse_x, slider_x, slider_w)
    if not item then return end
    local min, max = item.min, item.max
    local progress = math.clamp((mouse_x - slider_x) / slider_w, 0, 1)
    local val = min + (max - min) * progress
    val = math.round(val / item.step) * item.step
    val = math.clamp(val, min, max)
    return val
end

local function drawToggle(x, y, w, h, id, alpha)
    local anim = toggle_anim[id]
    local progress = anim and anim.current or (states[id] and 1 or 0)
    local r = STYLE.toggle_off[1] + (STYLE.toggle_on[1] - STYLE.toggle_off[1]) * progress
    local g = STYLE.toggle_off[2] + (STYLE.toggle_on[2] - STYLE.toggle_off[2]) * progress
    local b = STYLE.toggle_off[3] + (STYLE.toggle_on[3] - STYLE.toggle_off[3]) * progress
    local bg_color = {r, g, b, alpha * 0.9}
    local knob_offset = progress * (w - 16 - 6)
    local knob_x = x + 3 + knob_offset
    local knob_y = y + 3
    local knob_size = h - 6
    draw.RectFilled(x, y, w, h, bg_color, h/2)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), h/2, 1)
    draw.CircleFilled(knob_x + knob_size/2, knob_y + knob_size/2, knob_size/2, col_alpha(STYLE.knob, alpha), 12)
end

local function drawSlider(x, y, w, h, id, alpha)
    local val = states[id] or 0
    local item = nil
    for _, tab in ipairs(tabs) do
        for _, group in ipairs(tab.groups or {}) do
            for _, it in ipairs(group.items) do
                if it.id == id then item = it; break end
            end
            if item then break end
        end
        if item then break end
    end
    if not item then return end
    local min, max = item.min, item.max
    local progress = (val - min) / (max - min)
    local track_w = w - 18
    local track_h = 4
    local track_x = x
    local track_y = y + (h - track_h)/2
    draw.RectFilled(track_x, track_y, track_w, track_h, col_alpha(STYLE.slider_track, alpha), 2)
    draw.RectFilled(track_x, track_y, track_w * progress, track_h, col_alpha(STYLE.slider_fill, alpha), 2)
    local knob_x = track_x + track_w * progress
    draw.CircleFilled(knob_x, track_y + track_h/2, 5, col_alpha(STYLE.accent, alpha), 8)
    draw.Circle(knob_x, track_y + track_h/2, 5, col_alpha(STYLE.border, alpha), 8, 1)
    local text = tostring(val)
    if item.suffix then text = text .. item.suffix end
    local tw, th = draw.GetTextSize(text, FONT_S)
    draw.Text(x + w - tw - 2, y + (h - th)/2, text, col_alpha(STYLE.text_light, alpha), FONT_S)
end

local function drawComboDropdown(px, py, pw, id, options, alpha)
    if not options or #options == 0 then return end
    local item_h = 20
    local gap = 1
    local pad = 4
    local list_h = #options * (item_h + gap) + pad * 2
    local list_w = pw

    draw.RectFilled(px, py, list_w, list_h, col_alpha(STYLE.dropdown_bg, alpha), 4)
    draw.Rect(px, py, list_w, list_h, col_alpha(STYLE.border, alpha), 4, 1)

    local y_offset = py + pad
    for i, opt in ipairs(options) do
        local ix = px + pad
        local iy = y_offset
        local iw = list_w - pad * 2
        local ih = item_h

        local mx, my = utility.GetMousePos()
        local hover = (mx >= ix and mx <= ix + iw and my >= iy and my <= iy + ih)

        local bg = hover and col_alpha(STYLE.hover_bg, alpha) or col_alpha({0.1,0.1,0.15,0.5}, alpha)
        draw.RectFilled(ix, iy, iw, ih, bg, 3)
        draw.Text(ix + 4, iy + (ih - FONT_S)/2, opt, col_alpha(STYLE.text_light, alpha), FONT_S)

        y_offset = y_offset + ih + gap
    end
end

local function drawCombo(x, y, w, h, id, alpha)
    local state = states[id] or 0
    local item = nil
    for _, tab in ipairs(tabs) do
        for _, group in ipairs(tab.groups or {}) do
            for _, it in ipairs(group.items) do
                if it.id == id then item = it; break end
            end
            if item then break end
        end
        if item then break end
    end
    if not item or not item.options then return end
    local options = item.options
    local selected = options[state+1] or "?"
    draw.RectFilled(x, y, w, h, col_alpha(STYLE.combo_bg, alpha), 4)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), 4, 1)
    draw.Text(x + 4, y + (h - FONT_S)/2, selected, col_alpha(STYLE.text_light, alpha), FONT_S)
    local arrow = "▼"
    draw.Text(x + w - 14, y + (h - FONT_S)/2, arrow, col_alpha(STYLE.text_dim, alpha), FONT_S)
end

local function drawKeybind(x, y, w, h, id, alpha)
    local state = states[id] or 0
    local item = nil
    for _, tab in ipairs(tabs) do
        for _, group in ipairs(tab.groups or {}) do
            for _, it in ipairs(group.items) do
                if it.id == id then item = it; break end
            end
            if item then break end
        end
        if item then break end
    end
    if not item then return end

    local display_text
    if keybind_waiting == id then
        display_text = "Press any key..."
    else
        local key_code = state
        if key_code == 0x01 then display_text = "LMB"
        elseif key_code == 0x02 then display_text = "RMB"
        elseif key_code == 0x04 then display_text = "MMB"
        elseif key_code == 0x0D then display_text = "Enter"
        elseif key_code == 0x20 then display_text = "Space"
        elseif key_code == 0x2D then display_text = "Insert"
        elseif key_code >= 0x30 and key_code <= 0x39 then display_text = string.char(0x30 + (key_code - 0x30))
        elseif key_code >= 0x41 and key_code <= 0x5A then display_text = string.char(0x41 + (key_code - 0x41))
        elseif key_code >= 0x70 and key_code <= 0x7B then display_text = "F" .. (key_code - 0x70 + 1)
        elseif key_code == 0xA0 then display_text = "LShift"
        elseif key_code == 0xA1 then display_text = "RShift"
        elseif key_code == 0xA2 then display_text = "LCtrl"
        elseif key_code == 0xA3 then display_text = "RCtrl"
        elseif key_code == 0xA4 then display_text = "LAlt"
        elseif key_code == 0xA5 then display_text = "RAlt"
        else
            display_text = string.format("0x%02X", key_code)
        end
    end

    draw.RectFilled(x, y, w, h, col_alpha(STYLE.combo_bg, alpha), 4)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), 4, 1)
    draw.Text(x + 4, y + (h - FONT_S)/2, display_text, col_alpha(STYLE.text_light, alpha), FONT_S)
    local arrow = (keybind_waiting == id) and "..." or "▼"
    draw.Text(x + w - 14, y + (h - FONT_S)/2, arrow, col_alpha(STYLE.text_dim, alpha), FONT_S)
end

local function drawButton(x, y, w, h, id, alpha, hover)
    local item = nil
    for _, tab in ipairs(tabs) do
        for _, group in ipairs(tab.groups or {}) do
            for _, it in ipairs(group.items) do
                if it.id == id then item = it; break end
            end
            if item then break end
        end
        if item then break end
    end
    if not item then return end
    local bg = hover and STYLE.button_hover or STYLE.button_bg
    draw.RectFilled(x, y, w, h, col_alpha(bg, alpha), 4)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), 4, 1)
    draw.Text(x + w/2 - draw.GetTextSize(item.label, FONT_S)/2, y + (h - FONT_S)/2, item.label, col_alpha(STYLE.text_light, alpha), FONT_S)
end

local function drawColorPickerButton(x, y, w, h, id, alpha)
    local col = states[id] or {1,0,0,1}
    local color_key = id .. "_color"
    if states[color_key] then col = states[color_key] end
    draw.RectFilled(x, y, w, h, col_alpha({col[1], col[2], col[3], col[4]}, alpha), 4)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), 4, 1)
    draw.Text(x + 4, y + (h - FONT_S)/2, "", col_alpha({col[1], col[2], col[3], col[4]}, alpha), FONT_S)
end

local function drawInput(x, y, w, h, id, alpha)
    local val = states[id] or ""
    draw.RectFilled(x, y, w, h, col_alpha(STYLE.combo_bg, alpha), 4)
    draw.Rect(x, y, w, h, col_alpha(STYLE.border, alpha), 4, 1)
    draw.Text(x + 4, y + (h - FONT_S)/2, val, col_alpha(STYLE.text_light, alpha), FONT_S)
end

local function drawColorPickerPanel(id, px, py, pw, alpha)
    local col = states[id] or {1,0,0,1}
    local color_key = id .. "_color"
    if states[color_key] then col = states[color_key] end
    local panel_h = 130
    local panel_w = pw
    local pad = 8

    draw.RectFilled(px - 20, py, panel_w, panel_h, col_alpha(STYLE.dropdown_bg, alpha), 6)
    draw.Rect(px - 20, py, panel_w, panel_h, col_alpha(STYLE.border, alpha), 6, 1)

    local title = "Color Picker"
    draw.Text(px + pad - 20, py + 4, title, col_alpha(STYLE.text_group, alpha), FONT)

    --local val_text = string.format("R:%.2f G:%.2f B:%.2f A:%.2f", col[1], col[2], col[3], col[4])
    --draw.Text(px + panel_w - pad - draw.GetTextSize(val_text, FONT_S), py + 4, val_text, col_alpha(STYLE.text_light, alpha), FONT_S)

    local slider_y = py + 24
    local slider_h = 16
    local gap = 4
    local labels = {"R", "G", "B", "A"}
    local values = {col[1], col[2], col[3], col[4]}

    for i, label in ipairs(labels) do
        local slider_x = px + pad
        local track_w = panel_w - 2 * pad - 40
        local track_h = 4
        local track_x = slider_x - 10
        local track_y = slider_y + (slider_h - track_h)/2

        draw.Text(slider_x - 25, slider_y + 2, label, col_alpha(STYLE.text_dim, alpha), FONT_S-1)

        draw.RectFilled(track_x, track_y, track_w, track_h, col_alpha(STYLE.slider_track, alpha), 2)
        local progress = values[i]
        local fill_w = track_w * progress
        draw.RectFilled(track_x, track_y, fill_w, track_h, col_alpha(STYLE.slider_fill, alpha), 2)
        local knob_x = track_x + fill_w
        draw.CircleFilled(knob_x, track_y + track_h/2, 5, col_alpha(STYLE.accent, alpha), 8)
        draw.Circle(knob_x, track_y + track_h/2, 5, col_alpha(STYLE.border, alpha), 8, 1)

        local val_text2 = string.format("%.2f", values[i])
        draw.Text(track_x + track_w + 4, slider_y + 2, val_text2, col_alpha(STYLE.text_light, alpha), FONT_S-1)

        if colorpicker_open == id then
            colorpicker_panel_pos = colorpicker_panel_pos or { id = id, x = px, y = py, w = panel_w, h = panel_h, slots = {} }
            colorpicker_panel_pos.slots[i] = {
                x = track_x,
                y = track_y - 6,
                w = track_w,
                h = track_h + 12,
                min = 0,
                max = 1,
                step = 0.01,
                channel = i,
                id = id,
            }
        end

        slider_y = slider_y + slider_h + gap
    end

    if colorpicker_open == id then
        colorpicker_panel_pos = {
            id = id,
            x = px,
            y = py,
            w = panel_w,
            h = panel_h,
            slots = colorpicker_panel_pos and colorpicker_panel_pos.slots or {},
        }
    end
end

local function drawGroup(px, py, pw, group, alpha)
    local title = group.title or ""
    local items = getVisibleItems(group)
    local group_pad = 6
    local title_h = 18
    local item_h = 22
    local gap = 3
    local group_h = title_h + group_pad
    for _ in ipairs(items) do
        group_h = group_h + item_h + gap
    end
    group_h = group_h + group_pad

    draw.RectFilled(px, py, pw, group_h, col_alpha(STYLE.group_bg, alpha), 6)
    draw.Rect(px, py, pw, group_h, col_alpha(STYLE.group_border, alpha), 6, 1)

    draw.Text(px + group_pad, py + 2, title, col_alpha(STYLE.text_group, alpha), FONT)
    local line_y = py + title_h - 2
    draw.Line(px + group_pad, line_y, px + pw - group_pad, line_y, col_alpha(STYLE.border, alpha), 1)

    local y_offset = py + title_h + group_pad
    for _, item in ipairs(items) do
        local ix = px + group_pad
        local iy = y_offset
        local ih = item_h
        local ctrl_w = CTRL_WIDTHS[item.type] or 90
        local label_w = pw - 2 * group_pad - ctrl_w - 4
        if label_w > 0 then
            draw.Text(ix, iy + (ih - FONT_S)/2 + 1, item.label, col_alpha(STYLE.text_light, alpha), FONT_S)
        end
        local elem_x = ix + (pw - 2 * group_pad) - ctrl_w

        if item.type == "toggle" then
            drawToggle(elem_x, iy, ctrl_w, ih, item.id, alpha)
	elseif item.type == "label" then
    local label_color = item.color or STYLE.label_col
    draw.Text(ix, iy + (ih - FONT_S) / 2, item.label, col_alpha(label_color, alpha), FONT_S)
        elseif item.type == "slider" then
            drawSlider(elem_x, iy, ctrl_w, ih, item.id, alpha)
        elseif item.type == "combo" then
            drawCombo(elem_x, iy, ctrl_w, ih, item.id, alpha)
        elseif item.type == "keybind" then
            drawKeybind(elem_x, iy, ctrl_w, ih, item.id, alpha)
        elseif item.type == "button" then
            local mx_h, my_h = utility.GetMousePos()
            local hover = (mx_h >= elem_x and mx_h <= elem_x + ctrl_w and my_h >= iy and my_h <= iy + ih)
            drawButton(elem_x, iy, ctrl_w, ih, item.id, alpha, hover)
        elseif item.type == "colorpicker" then
            drawColorPickerButton(elem_x, iy, ctrl_w, ih, item.id, alpha)
            if colorpicker_open == item.id then
                colorpicker_panel_pos = {
                    id = item.id,
                    x = elem_x,
                    y = iy + ih + 2,
                    w = ctrl_w + 40,
                    h = 130,
                    slots = {},
                }
            end
        elseif item.type == "input" then
            drawInput(elem_x, iy, ctrl_w, ih, item.id, alpha)
        end

        y_offset = y_offset + ih + gap
    end

    return group_h
end

-- ============================================
-- SETTINGS WINDOW: состояние
-- ============================================
settings_win = {
    x = 350, y = 250,
    w = 300, h = 260,
    visible = true,
    dragging = false,
    drag_ox = 0, drag_oy = 0,
    active_tab = 1,
    clicked = false,
}

-- Иконки для боковых вкладок
local function drawSettingsIcon(icon, cx, cy )
    if icon == "Main" then
			local consoleicon = "MAIN"
			draw.Rect(cx - 14, cy - 11, 28, 22, STYLE.QB_COL_ICON, 3, 2)
			draw.Line(cx - 13, cy - 5, cx + 13, cy + -5, STYLE.QB_COL_ICON, 2)
			draw.Text(cx - 11, cy - 3, consoleicon, STYLE.QB_COL_ICON, 10 )

    elseif icon == "cross" then
        draw.Line(cx - 7, cy - 7, cx + 7, cy + 7, STYLE.QB_COL_ICON, 2)
        draw.Line(cx + 7, cy - 7, cx - 7, cy + 7, STYLE.QB_COL_ICON, 2)
        draw.CircleFilled(cx, cy, 2, col, 8)
    elseif icon == "key" then
        draw.Rect(cx - 8, cy - 4, 13, 8, STYLE.QB_COL_ICON, 2, 1.5)
        draw.Circle(cx - 5, cy, 2.5, STYLE.QB_COL_ICON, 8, 1.5)
        draw.Line(cx + 2, cy, cx + 6, cy, STYLE.QB_COL_ICON, 1.5)
    elseif icon == "gear" then
        draw.Circle(cx, cy, 7, STYLE.QB_COL_ICON, 16, 1.5)
        draw.Circle(cx, cy, 2.5, STYLE.QB_COL_ICON, 10, 1.5)
        for a = 0, 360, 60 do
            local rad = math.rad(a)
            draw.Line(cx + math.cos(rad) * 6, cy + math.sin(rad) * 6,
                      cx + math.cos(rad) * 8, cy + math.sin(rad) * 8, STYLE.QB_COL_ICON, 2)
        end
    end
end
local function drawSettingsWindow()
    local win = settings_win
    if not win.visible then return end

    local px, py = win.x, win.y
    local pw, ph = win.w, win.h
    local TITLE_H = 26
    local TAB_AREA_W = 54      -- узкая панель вкладок
    local TAB_BTN = 42         -- квадратные кнопки
    local TAB_GAP = 4
    local PAD = 8

    -- Перетаскивание
    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    if lmb then
        if not win.dragging then
            if mx >= px and mx <= px + pw and my >= py and my <= py + TITLE_H then
                win.dragging = true
                win.drag_ox = mx - px
                win.drag_oy = my - py
            end
        else
            win.x = mx - win.drag_ox
            win.y = my - win.drag_oy
            px, py = win.x, win.y
        end
    else
        win.dragging = false
    end

    -- Рамка окна + заголовок
    draw.RectFilled(px, py, pw, ph, STYLE.bg_main, 6)
    draw.Rect(px, py, pw, ph, STYLE.border, 6, 1.5)
    draw.RectFilled(px, py, pw, TITLE_H, STYLE.bg_title, 6)
    draw.Line(px, py + TITLE_H, px + pw, py + TITLE_H, STYLE.border, 1)
    draw.Text(px + PAD + 2, py + 5, "Settings", STYLE.accent, 14)
    draw.Text(px + pw - PAD - 30, py + 7, "drag", STYLE.text_dim, 10)

    -- Панель вкладок слева
    draw.RectFilled(px, py + TITLE_H, TAB_AREA_W, ph - TITLE_H, {0.05, 0.02, 0.07, 0.6}, 2)
    draw.Line(px + TAB_AREA_W, py + TITLE_H, px + TAB_AREA_W, py + ph, STYLE.border, 2)

    -- Квадратные кнопки вкладок с иконками
    local tab_x = px + (TAB_AREA_W - TAB_BTN) / 2
    local tab_y0 = py + TITLE_H + 10
    for i, stab in ipairs(SETTINGS_TABS) do
        local ty = tab_y0 + (i - 1) * (TAB_BTN + TAB_GAP)
        local is_active = (win.active_tab == i)
        local hover = (mx >= tab_x and mx <= tab_x + TAB_BTN and my >= ty and my <= ty + TAB_BTN)

        -- Фон кнопки (стиль quickbar)
        local bg
        if is_active then bg = STYLE.QB_COL_ON
        elseif hover then bg = STYLE.QB_COL_OFF
        else bg = STYLE.combo_bg end

        draw.RectFilled(tab_x, ty, TAB_BTN, TAB_BTN, bg, 4)
        draw.Rect(tab_x, ty, TAB_BTN, TAB_BTN, STYLE.border, 4, 1)

        -- Иконка
        local icon_col = is_active and {1, 1, 1, 1} or STYLE.text_dim
        drawSettingsIcon(stab.icon, tab_x + TAB_BTN / 2, ty + TAB_BTN / 2, icon_col)

        -- Полоска под активной
        if is_active then
            draw.Line(tab_x + 4, ty + TAB_BTN + 2, tab_x + TAB_BTN - 4, ty + TAB_BTN + 2, STYLE.QB_COL_LINE, 2)
        end
    end

    -- Контент — группы активной вкладки через drawGroup (та же функция, что в меню)
    local cx = px + TAB_AREA_W + PAD
    local cy = py + TITLE_H + PAD
    local cw = pw - TAB_AREA_W - PAD * 2

    local stab = SETTINGS_TABS[win.active_tab]
    if stab then
        local left, right, full = distributeGroups(stab.groups)
        local gap_between = 10
        local group_w = (cw - gap_between) / 2

        local fy = cy
        for _, g in ipairs(full) do
            local gh = drawGroup(cx, fy, cw, g, 1.0)
            fy = fy + gh + 6
        end

        local ly = fy
        for _, g in ipairs(left) do
            local gh = drawGroup(cx, ly, group_w, g, 1.0)
            ly = ly + gh + 6
        end

        local ry = fy
        local rx = cx + group_w + gap_between
        for _, g in ipairs(right) do
            local gh = drawGroup(rx, ry, group_w, g, 1.0)
            ry = ry + gh + 6
        end
    end
end
local function handleSettingsClicks(mx, my)
    local win = settings_win
    if not win.visible then return false end

    if colorpicker_open and colorpicker_panel_pos then
        local cp = colorpicker_panel_pos
        -- Если клик внутри панели colorpicker — не перехватываем
        if mx >= cp.x - 20 and mx <= cp.x - 20 + cp.w
           and my >= cp.y and my <= cp.y + cp.h then
            return false
        end
        -- Если клик по кнопке самого colorpicker (тот, что открыл панель) —
        -- тоже пропускаем, чтобы handleClicks сам решил: закрыть или оставить
    end

    if combo_open and combo_dropdown_pos then
        local cd = combo_dropdown_pos
        if mx >= cd.x and mx <= cd.x + cd.w
           and my >= cd.y and my <= cd.y + 200 then
            return false
        end
    end

    local px, py = win.x, win.y
    local pw, ph = win.w, win.h
    local TITLE_H = 26
    local TAB_AREA_W = 54
    local TAB_BTN = 42
    local TAB_GAP = 4
    local PAD = 8

    -- Заголовок — drag
    if mx >= px and mx <= px + pw and my >= py and my <= py + TITLE_H then
        win.dragging = true
        win.drag_ox = mx - px
        win.drag_oy = my - py
        return true
    end

    -- Боковые вкладки
    local tab_x = px + (TAB_AREA_W - TAB_BTN) / 2
    local tab_y0 = py + TITLE_H + 10
    for i = 1, #SETTINGS_TABS do
        local ty = tab_y0 + (i - 1) * (TAB_BTN + TAB_GAP)
        if mx >= tab_x and mx <= tab_x + TAB_BTN and my >= ty and my <= ty + TAB_BTN then
            win.active_tab = i
            click_handled = true
            return true
        end
    end

    -- Контент
    local cx = px + TAB_AREA_W + PAD
    local cy = py + TITLE_H + PAD
    local cw = pw - TAB_AREA_W - PAD * 2

    local stab = SETTINGS_TABS[win.active_tab]
    if not stab then return false end

    local left, right, full = distributeGroups(stab.groups)
    local gap_between = 10
    local group_w = (cw - gap_between) / 2

    -- Проходим по каждой группе
    local function processGroup(group, gx, gy, gw)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local inside_y = gy + title_h + group_pad

        for _, item in ipairs(item_parent_visible(group)) do
            local ix = gx + group_pad
            local iy = inside_y
            local ih = item_h
            local ctrl_w = CTRL_WIDTHS[item.type] or 90
            local elem_x = ix + (gw - 2 * group_pad) - ctrl_w

            local row_hit = (mx >= ix and mx <= ix + (gw - 2 * group_pad) and my >= iy and my <= iy + ih)

            -- Slider
            if item.type == "slider" then
                local sw_x = elem_x
                local sw_w = ctrl_w - 18
                local sw_y = iy + (ih - 4) / 2
                local sw_hit = (mx >= sw_x and mx <= sw_x + sw_w and my >= sw_y - 8 and my <= sw_y + 12)
                if sw_hit and not click_handled then
                    local item_data = nil
                    for _, tab in ipairs(tabs) do
                        for _, gg in ipairs(tab.groups or {}) do
                            for _, it in ipairs(gg.items) do
                                if it.id == item.id then item_data = it; break end
                            end
                            if item_data then break end
                        end
                        if item_data then break end
                    end
                    if item_data then
                        slider_drag = {
                            id = item.id, x = sw_x, w = sw_w,
                            min = item_data.min, max = item_data.max, step = item_data.step,
                        }
                        states[item.id] = getSliderValueFromMouse(item_data, mx, sw_x, sw_w)
                    end
                    click_handled = true
                    return true
                end
            end

            if row_hit and not click_handled then
                if item.type == "toggle" then
                    local ns = not states[item.id]
                    states[item.id] = ns
                    if toggle_anim[item.id] then toggle_anim[item.id].target = ns and 1 or 0 end
                    click_handled = true
                    return true
                elseif item.type == "button" then
                    if item.callback then item.callback() end
                    click_handled = true
                    return true
                elseif item.type == "colorpicker" then
                    if colorpicker_open == item.id then
                        colorpicker_open = nil
                        colorpicker_panel_pos = nil
                    else
                        colorpicker_open = item.id
                        colorpicker_panel_pos = nil
                    end
                    click_handled = true
                    return true
                elseif item.type == "combo" then
                    if combo_open == item.id then
                        combo_open = nil
                        combo_dropdown_pos = nil
                    else
                        combo_open = item.id
                        combo_dropdown_pos = { x = elem_x, y = iy + ih + 2, w = ctrl_w, options = item.options }
                    end
                    click_handled = true
                    return true
                elseif item.type == "keybind" then
                    keybind_waiting = (keybind_waiting == item.id) and nil or item.id
                    click_handled = true
                    return true
                end
            end

            inside_y = inside_y + ih + gap
        end
        return false
    end

    local fy = cy
    for _, g in ipairs(full) do
        local gh = 22 + #g.items * 25  -- примерно
        if processGroup(g, cx, fy, cw) then return true end
        -- точную высоту посчитаем так же, как drawGroup
        local gp, th, ih2, gap2 = 6, 18, 22, 3
        local vis = item_parent_visible(g)
        local hh = th + gp + #vis * (ih2 + gap2) + gp
        fy = fy + hh + 6
    end

    local ly = fy
    for _, g in ipairs(left) do
        if processGroup(g, cx, ly, group_w) then return true end
        local gp, th, ih2, gap2 = 6, 18, 22, 3
        local vis = item_parent_visible(g)
        local hh = th + gp + #vis * (ih2 + gap2) + gp
        ly = ly + hh + 6
    end

    local ry = fy
    local rx = cx + group_w + gap_between
    for _, g in ipairs(right) do
        if processGroup(g, rx, ry, group_w) then return true end
        local gp, th, ih2, gap2 = 6, 18, 22, 3
        local vis = item_parent_visible(g)
        local hh = th + gp + #vis * (ih2 + gap2) + gp
        ry = ry + hh + 6
    end

    return false
end

-- Аналог getVisibleItems для отфильтровывания по parent
function item_parent_visible(group)
    local out = {}
    for _, item in ipairs(group.items or {}) do
        if not item.parent or states[item.parent] then
            table.insert(out, item)
        end
    end
    return out
end


local function handleClicks(mx, my)
    if not win.visible then return end
    local px, py = win.x, win.y
    local pw, ph = win.w, win.h

    if mx >= px and mx <= px + pw and my >= py and my <= py + TITLE_H then
        win.dragging = true
        win.drag_ox = mx - px
        win.drag_oy = my - py
        return true
    end
local tab_start_y = py + TITLE_H + 4
local tab_x = px + PAD
local gap_tabs = 6
for i, tab in ipairs(tabs) do
    if tab.hidden then goto continue end
    local tw = draw.GetTextSize(tab.name, FONT)
    local tx = tab_x
    local ty = tab_start_y
    local th = TAB_H
    if mx >= tx and mx <= tx + tw and my >= ty and my <= ty + th then
        active_tab = i
        return true
    end
    tab_x = tab_x + tw + gap_tabs
    ::continue::
end

    if combo_open and combo_dropdown_pos then
        local dpx = combo_dropdown_pos.x
        local dpy = combo_dropdown_pos.y
        local dpw = combo_dropdown_pos.w
        local options = combo_dropdown_pos.options
        if options then
            local item_h = 20
            local gap_opt = 1
            local pad_opt = 4
            for i_opt, opt in ipairs(options) do
                local opt_ix = dpx + pad_opt
                local opt_iy = dpy + pad_opt + (i_opt-1) * (item_h + gap_opt)
                local opt_iw = dpw - pad_opt * 2
                local opt_ih = item_h
                if mx >= opt_ix and mx <= opt_ix + opt_iw and my >= opt_iy and my <= opt_iy + opt_ih then
                    states[combo_open] = i_opt - 1
                    --print("[Combo] " .. combo_open .. " = " .. opt)
                    combo_open = nil
                    combo_dropdown_pos = nil
                    click_handled = true
                    return true
                end
            end
        end
        if not (mx >= dpx and mx <= dpx + dpw and my >= dpy and my <= dpy + #options * (20+1) + 8) then
            combo_open = nil
            combo_dropdown_pos = nil
            return true
        end
    end

    if colorpicker_open and colorpicker_panel_pos then
        local cp = colorpicker_panel_pos
        if mx >= cp.x and mx <= cp.x + cp.w and my >= cp.y and my <= cp.y + cp.h then
            if cp.slots and type(cp.slots) == "table" then
                for i, slot in ipairs(cp.slots) do
                    if slot and mx >= slot.x and mx <= slot.x + slot.w and my >= slot.y and my <= slot.y + slot.h then
                        slider_drag = {
                            id = colorpicker_open,
                            channel = slot.channel,
                            min = slot.min,
                            max = slot.max,
                            step = slot.step,
                            x = slot.x,
                            w = slot.w,
                        }
                        local progress = math.clamp((mx - slot.x) / slot.w, 0, 1)
                        local val = slot.min + (slot.max - slot.min) * progress
                        val = math.round(val / slot.step) * slot.step
                        val = math.clamp(val, slot.min, slot.max)
                        local cur = states[colorpicker_open]
                        local new_col = {cur[1], cur[2], cur[3], cur[4]}
                        new_col[slot.channel] = val
                        states[colorpicker_open] = new_col
                        local color_key = colorpicker_open .. "_color"
                        states[color_key] = new_col
                        --print("[Colorpicker] " .. colorpicker_open .. " ch" .. slot.channel .. " = " .. tostring(val))
                        click_handled = true
                        return true
                    end
                end
            end
        else
            colorpicker_open = nil
            colorpicker_panel_pos = nil
            return true
        end
    end

    if combo_open or colorpicker_open then
        return false
    end

    local content_x = px + PAD
    local content_y = py + TITLE_H + TAB_H + PAD + 8
    local content_w = pw - 2 * PAD
    local gap_between = 10
    local group_w = (content_w - gap_between) / 2

    local groups = tabs[active_tab].groups or {}
	local left_groups, right_groups, full_groups = distributeGroups(groups)

    local full_y = content_y
    for _, group in ipairs(full_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad

        local gx = content_x
        local gy = full_y
        local gw = content_w
        local inside_y = gy + title_h + group_pad
        for _, item in ipairs(visible_items) do
            local ix = gx + group_pad
            local iy = inside_y
            local ih = item_h
            local ctrl_w = CTRL_WIDTHS[item.type] or 90
            local elem_x = ix + (gw - 2 * group_pad) - ctrl_w

            local row_hit = (mx >= ix and mx <= ix + (gw - 2 * group_pad) and my >= iy and my <= iy + ih)

            if item.type == "slider" then
                local slider_x = elem_x
                local slider_w = ctrl_w - 18
                local slider_y = iy + (ih - 4)/2
                local slider_h = 4
                local slider_hit = (mx >= slider_x and mx <= slider_x + slider_w and my >= slider_y - 8 and my <= slider_y + slider_h + 8)
                if slider_hit and not click_handled then
                    local item_data = nil
                    for _, tab in ipairs(tabs) do
                        for _, group in ipairs(tab.groups or {}) do
                            for _, it in ipairs(group.items) do
                                if it.id == item.id then
                                    item_data = it
                                    break
                                end
                            end
                            if item_data then break end
                        end
                        if item_data then break end
                    end
                    if item_data then
                        slider_drag = {
                            id = item.id,
                            x = slider_x,
                            w = slider_w,
                            min = item_data.min,
                            max = item_data.max,
                            step = item_data.step,
                        }
                        local val = getSliderValueFromMouse(item_data, mx, slider_x, slider_w)
                        if val then
                            states[item.id] = val
                        end
                    end
                    click_handled = true
                    return true
                end
            end

            if row_hit and not click_handled then
                if item.type == "toggle" then
                    local new_state = not states[item.id]
                    states[item.id] = new_state
                    if toggle_anim[item.id] then
                        toggle_anim[item.id].target = new_state and 1 or 0
                    end
                    click_handled = true
                    return true
                elseif item.type == "button" then
                    if item.callback then item.callback() end
                    click_handled = true
                    return true
                elseif item.type == "colorpicker" then
                    if colorpicker_open == item.id then
                        colorpicker_open = nil
                        colorpicker_panel_pos = nil
                    else
                        colorpicker_open = item.id
                        colorpicker_panel_pos = nil
                    end
                    click_handled = true
                    return true
                elseif item.type == "combo" then
                    if combo_open == item.id then
                        combo_open = nil
                        combo_dropdown_pos = nil
                    else
                        combo_open = item.id
                        combo_dropdown_pos = {x = elem_x, y = iy + ih + 2, w = ctrl_w, options = item.options}
                    end
                    click_handled = true
                    return true
                elseif item.type == "keybind" then
                    if keybind_waiting == item.id then
                        keybind_waiting = nil
                    else
                        keybind_waiting = item.id
                    end
                    click_handled = true
                    return true
                elseif item.type == "input" then
                    input_focus_id = item.id
                    input_text = states[item.id] or ""
                    keybind_waiting = nil
                    combo_open = nil
                    colorpicker_open = nil
                    click_handled = true
                    return true
                end
            end

            inside_y = inside_y + ih + gap
        end

        full_y = full_y + group_h + 6
    end

    local lx = content_x
    local ly = content_y
    for _, group in ipairs(left_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad

        local gx = lx
        local gy = ly
        local gw = group_w
        local inside_y = gy + title_h + group_pad
        for _, item in ipairs(visible_items) do
            local ix = gx + group_pad
            local iy = inside_y
            local ih = item_h
            local ctrl_w = CTRL_WIDTHS[item.type] or 90
            local elem_x = ix + (gw - 2 * group_pad) - ctrl_w

            local row_hit = (mx >= ix and mx <= ix + (gw - 2 * group_pad) and my >= iy and my <= iy + ih)

            if item.type == "slider" then
                local slider_x = elem_x
                local slider_w = ctrl_w - 18
                local slider_y = iy + (ih - 4)/2
                local slider_h = 4
                local slider_hit = (mx >= slider_x and mx <= slider_x + slider_w and my >= slider_y - 8 and my <= slider_y + slider_h + 8)
                if slider_hit and not click_handled then
                    local item_data = nil
                    for _, tab in ipairs(tabs) do
                        for _, group in ipairs(tab.groups or {}) do
                            for _, it in ipairs(group.items) do
                                if it.id == item.id then
                                    item_data = it
                                    break
                                end
                            end
                            if item_data then break end
                        end
                        if item_data then break end
                    end
                    if item_data then
                        slider_drag = {
                            id = item.id,
                            x = slider_x,
                            w = slider_w,
                            min = item_data.min,
                            max = item_data.max,
                            step = item_data.step,
                        }
                        local val = getSliderValueFromMouse(item_data, mx, slider_x, slider_w)
                        if val then
                            states[item.id] = val
                            --print("[Slider] " .. item.id .. " = " .. tostring(val))
                        end
                    end
                    click_handled = true
                    return true
                end
            end

            if row_hit and not click_handled then
                if item.type == "toggle" then
                    local new_state = not states[item.id]
                    states[item.id] = new_state
                    if toggle_anim[item.id] then
                        toggle_anim[item.id].target = new_state and 1 or 0
                    end
                    --print("[Toggle] " .. item.id .. " = " .. tostring(new_state))
                    click_handled = true
                    return true
				elseif item.type == "label" then
                elseif item.type == "button" then
                    if item.callback then item.callback() end
                    click_handled = true
                    return true
                elseif item.type == "colorpicker" then
                    if colorpicker_open == item.id then
                        colorpicker_open = nil
                        colorpicker_panel_pos = nil
                    else
                        colorpicker_open = item.id
                        colorpicker_panel_pos = nil
                    end
                    click_handled = true
                    return true
                elseif item.type == "combo" then
                    if combo_open == item.id then
                        combo_open = nil
                        combo_dropdown_pos = nil
                    else
                        combo_open = item.id
                        combo_dropdown_pos = {x = elem_x, y = iy + ih + 2, w = ctrl_w, options = item.options}
                    end
                    click_handled = true
                    return true
                elseif item.type == "keybind" then
                    if keybind_waiting == item.id then
                        keybind_waiting = nil
                    else
                        keybind_waiting = item.id
                    end
                    click_handled = true
                    return true
                elseif item.type == "input" then
                    click_handled = true
                    return true
                end
            end

            inside_y = inside_y + ih + gap
        end

        ly = ly + group_h + 6
    end

    local rx = content_x + group_w + gap_between
    local ry = content_y
    for _, group in ipairs(right_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad

        local gx = rx
        local gy = ry
        local gw = group_w
        local inside_y = gy + title_h + group_pad
        for _, item in ipairs(visible_items) do
            local ix = gx + group_pad
            local iy = inside_y
            local ih = item_h
            local ctrl_w = CTRL_WIDTHS[item.type] or 90
            local elem_x = ix + (gw - 2 * group_pad) - ctrl_w

            local row_hit = (mx >= ix and mx <= ix + (gw - 2 * group_pad) and my >= iy and my <= iy + ih)

            if item.type == "slider" then
                local slider_x = elem_x
                local slider_w = ctrl_w - 18
                local slider_y = iy + (ih - 4)/2
                local slider_h = 4
                local slider_hit = (mx >= slider_x and mx <= slider_x + slider_w and my >= slider_y - 8 and my <= slider_y + slider_h + 8)
                if slider_hit and not click_handled then
                    local item_data = nil
                    for _, tab in ipairs(tabs) do
                        for _, group in ipairs(tab.groups or {}) do
                            for _, it in ipairs(group.items) do
                                if it.id == item.id then
                                    item_data = it
                                    break
                                end
                            end
                            if item_data then break end
                        end
                        if item_data then break end
                    end
                    if item_data then
                        slider_drag = {
                            id = item.id,
                            x = slider_x,
                            w = slider_w,
                            min = item_data.min,
                            max = item_data.max,
                            step = item_data.step,
                        }
                        local val = getSliderValueFromMouse(item_data, mx, slider_x, slider_w)
                        if val then
                            states[item.id] = val
                            --print("[Slider] " .. item.id .. " = " .. tostring(val))
                        end
                    end	
                    click_handled = true
                    return true
                end
            end

            if row_hit and not click_handled then
                if item.type == "toggle" then
                    local new_state = not states[item.id]
                    states[item.id] = new_state
                    if toggle_anim[item.id] then
                        toggle_anim[item.id].target = new_state and 1 or 0
                    end
                    --print("[Toggle] " .. item.id .. " = " .. tostring(new_state))
                    click_handled = true
                    return true
                elseif item.type == "button" then
                    if item.callback then item.callback() end
                    click_handled = true
                    return true
                elseif item.type == "colorpicker" then
                    if colorpicker_open == item.id then
                        colorpicker_open = nil
                        colorpicker_panel_pos = nil
                    else
                        colorpicker_open = item.id
                        colorpicker_panel_pos = nil
                    end
                    click_handled = true
                    return true
                elseif item.type == "combo" then
                    if combo_open == item.id then
                        combo_open = nil
                        combo_dropdown_pos = nil
						states[combo_open] = i_opt - 1
						if callbacks[combo_open] then
    						callbacks[combo_open](i_opt - 1)
						end
                    else
                        combo_open = item.id
                        combo_dropdown_pos = {x = elem_x, y = iy + ih + 2, w = ctrl_w, options = item.options}
                    end
                    click_handled = true
                    return true
                elseif item.type == "keybind" then
                    if keybind_waiting == item.id then
                        keybind_waiting = nil
                    else
                        keybind_waiting = item.id
                    end
                    click_handled = true
                    return true
                elseif item.type == "input" then
                    click_handled = true
                    return true
                end
            end

            inside_y = inside_y + ih + gap
        end

        ry = ry + group_h + 6
    end

    return false
end

function drawPDHMenu()
    if not win.visible then return end

    local px, py = win.x, win.y
    local pw, ph = win.w, win.h

    if win.dragging then
        local mx, my = utility.GetMousePos()
        win.x = mx - win.drag_ox
        win.y = my - win.drag_oy
        px, py = win.x, win.y
    end

    local content_x = px + PAD
    local content_y = py + TITLE_H + TAB_H + PAD + 8
    local content_w = pw - 2 * PAD
    local gap_between = 10
    local group_w = (content_w - gap_between) / 2

	local groups = tabs[active_tab].groups or {}
    local full_groups = {}
    local normal_groups = {}
    for _, g in ipairs(groups) do
        if g.side == "full" then
            table.insert(full_groups, g)
        else
            table.insert(normal_groups, g)
        end
    end
    local groups = tabs[active_tab].groups or {}
    local left_groups, right_groups, full_groups = distributeGroups(groups)

	local full_height = 0
    for _, group in ipairs(full_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad
        full_height = full_height + group_h + 6
    end

 	local left_groups, right_groups = distributeGroups(normal_groups)

    local left_height = 0
    for _, group in ipairs(left_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad
        left_height = left_height + group_h + 6
    end

    local right_height = 0
    for _, group in ipairs(right_groups) do
        local visible_items = getVisibleItems(group)
        local group_pad = 6
        local title_h = 18
        local item_h = 22
        local gap = 3
        local group_h = title_h + group_pad
        for _ in ipairs(visible_items) do
            group_h = group_h + item_h + gap
        end
        group_h = group_h + group_pad
        right_height = right_height + group_h + 6
    end

    local max_height = math.max(left_height, right_height)
    local new_h = math.max(full_height + max_height + TITLE_H + TAB_H + PAD + 16, 100)
    win.h = new_h
    ph = new_h

    draw.Rect(px-1, py-1, pw+2, ph+2, STYLE.border, 8, 1.5)
    draw.RectFilled(px, py, pw, ph, STYLE.bg_main, 8)
    draw.RectFilled(px, py, pw, TITLE_H, STYLE.bg_title, 8)
    draw.Line(px, py + TITLE_H, px + pw, py + TITLE_H, STYLE.border, 1)

    draw.Text(px + PAD + 2, py + 6, "PDH - Project Delta", STYLE.accent, FONT+2)
    draw.Text(px + PAD + 2 + 155, py + 8, "v0.53", STYLE.accent, FONT-1)
    draw.Text(px + pw - PAD - 45, py + 6, "drag ::", STYLE.text_dim, FONT)

    local tab_start_y = py + TITLE_H + 4
    local tab_x = px + PAD
    local gap_tabs = 6
    for i, tab in ipairs(tabs) do
		if tab.hidden then goto continue end
        local tw = draw.GetTextSize(tab.name, FONT)
        local tx = tab_x
        local ty = tab_start_y
        local th = TAB_H
        local is_active = (i == active_tab)

        local color = is_active and STYLE.accent or STYLE.tab_def
        draw.Text(tx, ty + 4, tab.name, color, FONT)

        if is_active then
            local line_y = ty + th - 2
            draw.Line(tx, line_y, tx + tw, line_y, STYLE.accent, 2)
        end

        tab_x = tab_x + tw + gap_tabs
		::continue::
    end

    draw.Line(px + PAD, py + TITLE_H + TAB_H + 4, px + pw - PAD, py + TITLE_H + TAB_H + 4, STYLE.border, 0.5)

	local full_y = content_y
	for _, group in ipairs(full_groups) do
    	local gh = drawGroup(content_x, full_y, content_w, group, 1.0)
    	full_y = full_y + gh + 6
	end	

  local lx = content_x
    local ly = content_y
    for _, group in ipairs(left_groups) do
        local gh = drawGroup(lx, ly, group_w, group, 1.0)
        ly = ly + gh + 6
    end

    local rx = content_x + group_w + gap_between
    local ry = content_y
    for _, group in ipairs(right_groups) do
        local gh = drawGroup(rx, ry, group_w, group, 1.0)
        ry = ry + gh + 6
    end

    draw.Text(px + PAD, py + ph - 16, "FPS: " .. math.floor(utility.GetFPS()), col_alpha(STYLE.accent, 0.5), 10)
end

local function drawPopups()
    if combo_open and combo_dropdown_pos then
        local dpx = combo_dropdown_pos.x
        local dpy = combo_dropdown_pos.y
        local dpw = combo_dropdown_pos.w
        local options = combo_dropdown_pos.options
        if options then
            drawComboDropdown(dpx, dpy, dpw, combo_open, options, 1.0)
        end
    end

    if colorpicker_open and colorpicker_panel_pos then
        local cp = colorpicker_panel_pos
        drawColorPickerPanel(cp.id, cp.x, cp.y, cp.w, 1.0)
    end
end

menu = menu or {}
function menu.get(id)
    return states[id]
end

function menu.get_color(id)
    local color_key = id .. "_color"
    return states[color_key] or {1,1,1,1}
end

function menu.get_key(id)
    local key_key = id .. "_key"
    return states[key_key] or 0
end

function menu.set(id, value)
    states[id] = value

    if toggle_anim and toggle_anim[id] then
        toggle_anim[id].target = value and 1 or 0
        toggle_anim[id].current = toggle_anim[id].target
    end
end

function menu.set_color(id, color)
    local color_key = id .. "_color"
    states[color_key] = color
    if states[id] then
        states[id] = color
    end
end

function menu.set_key(id, key)
    local key_key = id .. "_key"
    states[key_key] = key
end
function menu.SetCallback(id, func)
    callbacks[id] = func
end

local NOTIF_WIDTH = 280
local NOTIF_PAD = 8
local NOTIF_TITLE_H = 20
local NOTIF_FONT = 11
local NOTIF_TIMER_HEIGHT = 3
local NOTIF_TIMER_COLOR = { 0.35, 0.16, 1.0, 0.9 }
local NOTIF_TITLES = { info = "Info", success = "Success", warning = "Warning", error = "Error" }
local NOTIF_COLORS = { info = {0.3, 0.6, 1.0, 1.0}, success = {0.3, 0.9, 0.3, 1.0}, warning = {1.0, 0.8, 0.2, 1.0}, error = {1.0, 0.3, 0.3, 1.0} }
local notifications = {}
local MAX_NOTIFICATIONS = 5
local NOTIF_DEFAULT_DURATION = 3.0
local notif_click_handled = false

function addNotification(text, type, duration)
    type = type or "info"
    duration = duration or NOTIF_DEFAULT_DURATION
    table.insert(notifications, { text = text, type = type, time = utility.GetTime(), duration = duration })
    while #notifications > MAX_NOTIFICATIONS do table.remove(notifications, 1) end
end

function draw_Notifications()
    local now = utility.GetTime()
    for i = #notifications, 1, -1 do
        if now - notifications[i].time > notifications[i].duration then table.remove(notifications, i) end
    end
    if #notifications == 0 then return end
    local sw, sh = draw.GetScreenSize()
    local x = sw - NOTIF_WIDTH - 20
    local y = sh - 20
    for i = #notifications, 1, -1 do
        local notif = notifications[i]
        local tw, th = draw.GetTextSize(notif.text, NOTIF_FONT)
        local text_h = math.max(th, 14)
        local panel_h = NOTIF_TITLE_H + NOTIF_PAD + text_h + NOTIF_PAD + NOTIF_TIMER_HEIGHT + 2
        y = y - panel_h - 6
        local px, py = x, y
        local pw = NOTIF_WIDTH
        draw.Rect(px-1, py-1, pw+2, panel_h+2, STYLE.border, 5, 1)
        draw.RectFilled(px, py, pw, panel_h, STYLE.bg_main, 4)
        draw.RectFilled(px, py, pw, NOTIF_TITLE_H, STYLE.bg_title, 4)
        draw.RectFilled(px, py, 2, NOTIF_TITLE_H, STYLE.border, 0)
        draw.Line(px, py + NOTIF_TITLE_H, px + pw, py + NOTIF_TITLE_H, STYLE.border, 1)
        draw.Rect(px, py, pw, panel_h, STYLE.border, 4, 1)
        local title = NOTIF_TITLES[notif.type] or "Info"
        local color = NOTIF_COLORS[notif.type] or {1,1,1,1}
        draw.Text(px + NOTIF_PAD, py + 4, title, color, NOTIF_FONT)
        draw.Text(px + NOTIF_PAD, py + NOTIF_TITLE_H + 4, notif.text, {1,1,1,1}, NOTIF_FONT)
        local elapsed = now - notif.time
        local progress = 1 - (elapsed / notif.duration)
        if progress > 0 then
            draw.RectFilled(px, py + panel_h - NOTIF_TIMER_HEIGHT - 1, pw * progress, NOTIF_TIMER_HEIGHT, NOTIF_TIMER_COLOR, 0)
        end
        local mx, my = utility.GetMousePos()
        if input.IsKeyDown(0x01) and not notif_click_handled then
            if mx >= px and mx <= px + pw and my >= py and my <= py + panel_h then
                table.remove(notifications, i)
                notif_click_handled = true
            end
        end
    end
    if not input.IsKeyDown(0x01) then notif_click_handled = false end
end

-- ===== INVISIBILITY CHECKER =====
invis_tracker = {}
logs = {}

local function is_player_invisible(char)
    if not char then return false end
    local parts = char:GetChildren()
    local has_body_part = false
    for _, part in ipairs(parts) do
        if part:IsA("BasePart") then
            has_body_part = true
            local ok, trans = pcall(function() return part.Transparency end)
            if ok and trans and trans < 1 then
                return false
            end
        end
    end
    return has_body_part
end

local function check_invisibility()
    local players = entity.GetPlayers()
    if not players then 
        return 
    end
    
    local now = utility.GetTime()
    
    for _, p in ipairs(players) do
        if p.IsLocal or not p.IsAlive then goto continue end
        
        local char = p.Character
		--print("[INVIS] Player: " .. tostring(p.Name) .. " | Character: " .. tostring(char ~= nil))
        local uid = p.UserId
        if not uid or uid == 0 then uid = p.Name end
            local parts = char:GetChildren()

        invisible = is_player_invisible(char)
        track = invis_tracker[uid]
        
        if invisible then
            if not track then
                invis_tracker[uid] = {
                    name = p.Name,
                    start_time = now,
                    logged = false,
                }
                addNotification("INVISIBLE: " .. p.Name, "warning", 4)
                addConsoleMessage("[INVLog] " .. p.Name .. " detect inv", {1, 0.6, 0, 1})
		elseif not track.logged and (now - track.start_time) >= 5 then
    			track.logged = true
    		timestamp = os.date("%H:%M:%S")
   			log_entry = string.format("[%s] %s (UID: %s) ivisible %.1f sec.",
        			timestamp, track.name, tostring(uid), now - track.start_time)
    
    			addConsoleMessage("[INVLog] " .. track.name, {1, 0.3, 0.3, 1})
    			--print("[INVLog] " .. log_entry)
    
    		table.insert(logs, {
        	time = timestamp,
        	text = log_entry,
    })
end
        else
            if track then
                invis_tracker[uid] = nil
            end
        end
        
        ::continue::
    end
end

thread.create(check_invisibility, 300)

local function apply_menu_colors()
    local accent = states["menu_accent"]
    if accent then
        --STYLE.accent = accent
        --STYLE.accent_light = accent
        STYLE.toggle_on = accent
        STYLE.slider_fill = accent
        STYLE.button_hover = accent
		STYLE.QB_COL_ON  = accent
    end

    local bg = states["menu_bg"]
    if bg then
        STYLE.bg_main = bg
		STYLE.dropdown_bg = bg
    end

    local border = states["menu_border"]
    if border then
        STYLE.border = border
    end

    local text = states["menu_text"]
    if text then
        STYLE.text_dim  = text
		STYLE.text_light  = text
    end

    local title_bg = states["menu_title_bg"]
    if title_bg then
        STYLE.bg_title = title_bg
    end
end

local oldOnFrame = OnFrame
OnFrame = function()
    if oldOnFrame then oldOnFrame() end


    updateToggleAnimations()
	apply_menu_colors()  

settings_win.visible = menu.get("show_settings_window") and true or false

-- Сначала обрабатываем клики по настройкам, потом по меню
if input.IsKeyDown(0x01) and not click_handled and settings_win.visible then
    handleSettingsClicks(utility.GetMousePos())
end

local current = menu.get("norecoil_enabled")
if current ~= norecoil_prev_state then
    norecoil_prev_state = current
    if current then
        start_norecoil()
    else
        stop_norecoil()
    end
end
    menu_toggle_key = states["menu_toggle_key"] or 0x50
    if input.IsKeyDown(menu_toggle_key) then
        if not menu_key_was_down then
            win.visible = not win.visible
            menu_key_was_down = true
        end
    else
        menu_key_was_down = false
    end

    if slider_drag then
        if input.IsKeyDown(0x01) then
            local mx, my = utility.GetMousePos()
            if slider_drag.channel then
                local progress = math.clamp((mx - slider_drag.x) / slider_drag.w, 0, 1)
                local val = slider_drag.min + (slider_drag.max - slider_drag.min) * progress
                val = math.round(val / slider_drag.step) * slider_drag.step
                val = math.clamp(val, slider_drag.min, slider_drag.max)
                local cur = states[slider_drag.id]
                local new_col = {cur[1], cur[2], cur[3], cur[4]}
                new_col[slider_drag.channel] = val
                states[slider_drag.id] = new_col
                local color_key = slider_drag.id .. "_color"
                states[color_key] = new_col
            else
                local item = nil
                for _, tab in ipairs(tabs) do
                    for _, group in ipairs(tab.groups or {}) do
                        for _, it in ipairs(group.items) do
                            if it.id == slider_drag.id then item = it; break end
                        end
                        if item then break end
                    end
                    if item then break end
                end
                if item then
                    local new_val = getSliderValueFromMouse(item, mx, slider_drag.x, slider_drag.w)
                    if new_val then
                        local old_val = states[slider_drag.id] or 0
                        if new_val ~= old_val then
                            states[slider_drag.id] = new_val
                            --print("[Slider] " .. slider_drag.id .. " = " .. tostring(new_val))
                        end
                    end
                end
            end
        else
            slider_drag = nil
        end
    end

    if keybind_waiting then
        for code = 0x01, 0xFF do
            if input.IsKeyDown(code) then
                if code ~= 0x01 then
                    states[keybind_waiting] = code
                    local key_key = keybind_waiting .. "_key"
                    states[key_key] = code
                    --print("[Keybind] " .. keybind_waiting .. " = 0x" .. string.format("%02X", code))
                    keybind_waiting = nil
                    break
                end
            end
        end
    end

    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    if lmb then
        if not win.dragging and win.visible then
            if mx >= win.x and mx <= win.x + win.w and my >= win.y and my <= win.y + TITLE_H then
                win.dragging = true
                win.drag_ox = mx - win.x
                win.drag_oy = my - win.y
            end
        end
        if not click_handled and win.visible then
            handleClicks(mx, my)
            click_handled = true
        end
    else
        win.dragging = false
        click_handled = false
        if slider_drag then slider_drag = nil end
    end

    drawPDHMenu()
    draw_Notifications()
	drawQuickBar()
	drawSettingsWindow()
	drawPopups()

    if type(on_frame) == "function" then
        on_frame() 
    end
end
end

main()

local sqrt = math.sqrt
local format = string.format
local CONFIG_NAME = "PDH-PDeltaCFG.txt"

local function get_config_path()
    local slot_index = menu.get("config_slot") or 0
    local slot_name = CONFIG_SLOTS[slot_index + 1] or "?"
    local filename = "PDH-PDeltaCFG_" .. slot_name .. ".txt"
    local appdata = os.getenv("LOCALAPPDATA")
    if appdata then
        return appdata .. "\\Project Vector\\Scripts\\" .. filename
    end
    return filename
end

local function safe_menu_get(id)
    local ok, val = pcall(function() return menu.get(id) end)
    return ok and val or false
end

local function safe_menu_get_color(id)
    local ok, col = pcall(function() return menu.get_color(id) end)
    return ok and col or {1, 1, 1, 1}
end

local function safe_menu_get_key(id)
    local ok, key = pcall(function() return menu.get_key(id) end)
    return ok and key or 0
end

local SAVE_ITEMS = {
    { id = "delta_player_enabled", type = "bool" },
    { id = "delta_esp_enabled", type = "bool" },
	{ id = "esp_player_box", type = "bool" },
	{ id = "esp_box_style", type = "int" },
	{ id = "esp_box_color", type = "color", color = true },
	{ id = "esp_skeleton", type = "bool" },
	{ id = "esp_skeleton_color", type = "color", color = true },
    { id = "delta_player_dist",    type = "bool", color = true },
    { id = "delta_player_range",   type = "int" },
    { id = "delta_extract_enabled", type = "bool", color = false },
    { id = "delta_extract_col", type = "color", color = true },
    { id = "delta_extract_range",  type = "int" },
    { id = "delta_container_enabled", type = "bool", color = true },
    { id = "delta_container_contents", type = "bool" },
    { id = "delta_container_range", type = "int" },
    { id = "delta_quest_enabled",  type = "bool", color = true },
    { id = "delta_quest_range",    type = "int" },
    { id = "delta_items_enabled",  type = "bool", color = true },
    { id = "delta_items_range",    type = "int" },
    { id = "delta_corpse_enabled", type = "bool", color = true },
    { id = "delta_corpse_range",   type = "int" },
    { id = "delta_npc_enabled",    type = "bool" },
    { id = "delta_npc_box",        type = "bool", color = true },
	{ id = "delta_npccol_box", type = "color", color = true },
    { id = "delta_npc_health",     type = "bool" },
    { id = "delta_npc_name",       type = "bool", color = true },
	{ id = "delta_npccol_name", type = "color", color = true },
    { id = "delta_npc_dist",       type = "bool", color = true },
	{ id = "delta_npccol_dist", type = "color", color = true },
    { id = "delta_npc_skeleton",   type = "bool", color = true },
	{ id = "delta_npccol_skeleton", type = "color", color = true },
    { id = "delta_npc_range",      type = "int" },
    { id = "delta_npc_aim_enabled", type = "bool", key = true },
    { id = "delta_npc_aim_fov",    type = "int" },
    { id = "delta_npc_aim_smooth", type = "int" },
    { id = "delta_npc_aim_bone",   type = "int" },
    { id = "delta_npc_aim_prediction", type = "bool" },
    { id = "delta_npc_aim_draw_fov", type = "bool" },
    { id = "delta_inv_enabled",    type = "bool" },
    { id = "delta_inv_mode",       type = "int" },
    { id = "delta_inv_show_guns",  type = "bool" },
    { id = "delta_inv_max_guns",   type = "int" },
    { id = "delta_inv_show_sights",  type = "bool" },
    { id = "delta_inv_max_sights",   type = "int" },
    { id = "delta_inv_show_armor", type = "bool" },
    { id = "delta_inv_max_armor",  type = "int" },
    { id = "delta_inv_show_val",   type = "bool" },
    { id = "delta_inv_max_val",    type = "int" },
    { id = "delta_inv_show_serun",   type = "bool" },
    { id = "delta_inv_max_serun",    type = "int" },
    { id = "delta_inv_show_other", type = "bool" },
    { id = "delta_inv_max_other",  type = "int" },
    { id = "delta_boss_enabled",   type = "bool" },
    { id = "delta_claymore_enabled", type = "bool", color = true },
    { id = "delta_claymore_range", type = "int" },
    { id = "delta_claymore_names", type = "bool" },
    { id = "delta_mod_alert_enabled", type = "bool" },
    { id = "delta_mod_alert_mode", type = "int" },
    { id = "delta_slf_enabled",    type = "bool" },
    { id = "delta_debug_enabled",    type = "bool" },
    { id = "delta_radar_enabled",    type = "bool" },
	{ id = "radar_range", type = "int" },
	{ id = "crosshair_enabled", type = "bool" },
	{ id = "crosshair_style", type = "int" },
	{ id = "crosshair_dot_size", type = "int" },
	{ id = "crosshair_line_len", type = "int" },
	{ id = "crosshair_line_gap", type = "int" },
	{ id = "crosshair_line_thick", type = "int" },
	{ id = "crosshair_color", type = "color", color = true },
	{ id = "pl_aim_enabled", type = "bool" },
	{ id = "pl_aim_key", type = "int", key = true },
	{ id = "pl_aim_bone", type = "int" },
	{ id = "pl_aim_fov", type = "int" },
	{ id = "pl_aim_smooth", type = "int" },
	{ id = "pl_aim_prediction", type = "bool" },
	{ id = "pl_aim_draw_fov", type = "bool" },
	{ id = "pl_aim_fov_color", type = "color", color = true },
	{ id = "pl_aim_visible_check", type = "bool" },
	{ id = "show_keybinds", type = "bool" },
	{ id = "silent_enabled", type = "bool" },
	{ id = "silent_key", type = "int", key = true },
	{ id = "silent_bone", type = "int" },
	{ id = "silent_fov", type = "int" },
	{ id = "silent_visible_check", type = "bool" },
	{ id = "silent_draw_fov", type = "bool" },
	{ id = "silent_fov_color", type = "color", color = true },
	{ id = "silent_prediction", type = "int" },
	{ id = "quick_btn1",      type = "bool" },
	{ id = "quick_btn2",      type = "bool" },
	{ id = "quick_btn3",      type = "bool" },
	{ id = "menu_toggle_key", type = "int", key = true },
	{ id = "menu_accent",   type = "color", color = true },
	{ id = "menu_bg",       type = "color", color = true },
	{ id = "menu_border",   type = "color", color = true },
	{ id = "menu_title_bg", type = "color", color = true },
	{ id = "menu_text", type = "color", color = true },
}

function save_config()
    local ok, result = pcall(function()
        local path = get_config_path()
        local dir = path:match("^(.+)\\[^\\]+$")
        if dir then pcall(function() os.execute('mkdir "' .. dir .. '" 2>nul') end) end
        local file = io.open(path, "w")
        if not file then print("[Project Delta] Failed to open config file: " .. path) return false end
        for _, item in ipairs(SAVE_ITEMS) do
            local id = item.id
            local val = safe_menu_get(id)
            if item.type == "bool" then
                file:write(id .. "=" .. (val and "1" or "0") .. "\n")
            elseif item.type == "int" then
                file:write(id .. "=" .. tostring(tonumber(val) or 0) .. "\n")
            end
            if item.color then
                local col = safe_menu_get_color(id)
                if col then
                    file:write(id .. "_color=" .. string.format("%.4f,%.4f,%.4f,%.4f", col[1] or 1, col[2] or 1, col[3] or 1, col[4] or 1) .. "\n")
                end
            end
            if item.key then
                file:write(id .. "_key=" .. tostring(safe_menu_get_key(id)) .. "\n")
            end
        end
        file:close()
        --print("[Project Delta] Config saved to: " .. path)
		local slot_name = CONFIG_SLOTS[(menu.get("config_slot") or 0) + 1] or "main"
		addConsoleMessage("[MAIN] Config saved to slot '" .. slot_name .. "'", {0,1,0,1})
        return true
    end)
    if not ok then print("[Project Delta] Config save error: " .. tostring(result)) return false end
    return result
end

function load_config()
    local ok, result = pcall(function()
        local path = get_config_path()
        local file = io.open(path, "r")
        if not file then print("[Project Delta] No config file found at: " .. path) return false end
        local data = {}
        for line in file:lines() do
            local key, value = line:match("^([^=]+)=(.*)$")
            if key and value then data[key] = value end
        end
        file:close()
        for _, item in ipairs(SAVE_ITEMS) do
            local id = item.id
            if data[id] then
                if item.type == "bool" then pcall(function() menu.set(id, data[id] == "1") end)
                elseif item.type == "int" then pcall(function() menu.set(id, tonumber(data[id]) or 0) end) end
            end
            if item.color and data[id .. "_color"] then
                local r, g, b, a = data[id .. "_color"]:match("([%d%.%-]+),([%d%.%-]+),([%d%.%-]+),([%d%.%-]+)")
                if r then pcall(function() menu.set_color(id, { tonumber(r), tonumber(g), tonumber(b), tonumber(a) }) end) end
            end
            if item.key and data[id .. "_key"] then
                pcall(function() menu.set_key(id, tonumber(data[id .. "_key"]) or 0) end)
            end
        end
		local slot_name = CONFIG_SLOTS[(menu.get("config_slot") or 0) + 1] or "main"
		loaded_config_name = CONFIG_SLOTS[(menu.get("config_slot") or 0) + 1] or "main"
		addConsoleMessage("[MAIN] Config loaded from slot '" .. slot_name .. "'", {0,1,0,1})
        return true
    end)
    if not ok then print("[Project Delta] Config load error: " .. tostring(result)) return false end
    return result
end

menu.add_button("Project Delta", "Config", "delta_save_config", "Save Config", function() save_config() end)
menu.add_button("Project Delta", "Config", "delta_load_config", "Load Config", function() load_config() end)

function collect_debug_snapshot()
    local now = utility.GetTime()
    local snapshot = {
        threads = {},
        stats = {
            fps = math.floor(utility.GetFPS() or 0),
            players = entity.GetPlayerCount() or 0,
            place_id = game.PlaceId or 0,
            game_id = game.GameId or 0,
            config = loaded_config_name,
            uptime = string.format("%.1f s", now),
        }
    }
    
    for name, data in pairs(thread_status) do
        local last_str
        if data.last_run == 0 then
            last_str = "never"
        else
            last_str = string.format("%.1fs ago", now - data.last_run)
        end
        table.insert(snapshot.threads, {
            name = name,
            status = data.status or "unknown",
            count = data.count or 0,
            last_run = last_str,
        })
    end
    
    table.sort(snapshot.threads, function(a, b) return a.name < b.name end)
    
    return snapshot
end


function save_logs()
    local has_invis = #logs > 0
    local debug_snapshot = collect_debug_snapshot()

    
    local date_str = os.date("%Y-%m-%d_%H-%M-%S")
    local filename = "log_" .. date_str .. ".txt"
    
    local appdata = os.getenv("LOCALAPPDATA")
    local path
    if appdata then
        path = appdata .. "\\Project Vector\\Scripts\\" .. filename
    else
        path = filename
    end
    
    local dir = path:match("^(.+)\\[^\\]+$")
    if dir then
        pcall(function() os.execute('mkdir "' .. dir .. '" 2>nul') end)
    end
    
    local ok, err = pcall(function()
        local file = io.open(path, "w")
        if not file then
            error("Could not open the file: " .. path)
        end
        
        file:write("========================================\n")
        file:write("  Log - PDH Project Delta\n")
        file:write("  Date: " .. os.date("%Y-%m-%d %H:%M:%S") .. "\n")
        file:write("  Total entries: " .. #logs .. "\n")
		file:write("  Info: V0.53  |  FPS: " .. math.floor(utility.GetFPS()) ..  "\n")
		file:write("----------------------------------------\n")
        file:write("  Player count: " .. (entity.GetPlayerCount() or 0) .. "\n")
        file:write("  Loaded config: " .. loaded_config_name .. "\n")
        file:write("  Place ID: " .. tostring(game.PlaceId or 0) .. "\n")
		file:write("  Uptime:         " .. debug_snapshot.stats.uptime .. "\n")
        file:write("========================================\n")
        
        file:write("--- INVISIBILITY LOG ---\n")
        if has_invis then
            file:write("  Total detected: " .. #logs .. "\n\n")
            for _, entry in ipairs(logs) do
                file:write("  " .. entry.text .. "\n")
            end
        else
            file:write("  [NO INFO] - nil\n")
        end
        file:write("\n")
        

        file:write("--- CONSOLE MESSAGES ---\n")
        if console_log_buffer and #console_log_buffer > 0 then
            file:write("  Total: " .. #console_log_buffer .. "\n\n")
            for _, msg in ipairs(console_log_buffer) do
                local count = msg.count or 1
                if count > 1 then
                    file:write(string.format("  [%s] %s (x%d)\n", msg.time, msg.text, count))
                else
                    file:write(string.format("  [%s] %s\n", msg.time, msg.text))
                end
            end
        else
            file:write("  [NO INFO] No console messages.\n")
        end
		file:write("------------------------\n")
        file:write("\n")


        file:write("--- THREADS STATUS ---\n")
        if #debug_snapshot.threads > 0 then
            for _, t in ipairs(debug_snapshot.threads) do
                file:write(string.format("  %-20s | %-8s | Runs: %-5d | Last: %s\n",
                    t.name, t.status, t.count, t.last_run))
            end
        else
            file:write("  [NO INFO] No thread data available.\n")
        end
        file:write("\n")


        file:write("--- DETECTED MODERATORS ---\n")
        if detected_mods and #detected_mods > 0 then
            for _, mod_name in ipairs(detected_mods) do
                file:write("  - " .. mod_name .. "\n")
            end
        else
            file:write("  [NOT DETECTED] No moderators in server.\n")
        end
        file:write("\n")


        file:write("--- SUMMARY ---\n")
        file:write("  Invisibility events: " .. #logs .. "\n")
        file:write("  Active threads:      " .. #debug_snapshot.threads .. "\n")
        file:write("  Console messages:    " .. ((console_log_buffer and #console_log_buffer) or (console_messages and #console_messages) or 0) .. "\n")
        file:write("\n")
        file:write("========================================\n")
        file:write("  End of log\n")
        file:write("========================================\n")
        
        file:close()
        return true
    end)
    
    if ok then
        addConsoleMessage("[LOG] Saved: " .. filename, {0, 1, 0, 1})
        addNotification("Log saved: " .. #logs .. " entries", "success", 4)
        return true
    else
        addConsoleMessage("[LOG] Err: " .. tostring(err), {1, 0, 0, 1})
        addNotification("Failed to save log", "error", 4)
        return false
    end
end

quickbar_win = {
    x = 882,
    y = 1010,
    visible = true,
    dragging =  false,
    drag_ox = 0,
    drag_oy = 0,
    clicked = false,
}

quick_icons = {
    [1] = "Console",
    [2] = "Debug",
    [3] = "Radar",
	[4] = "Find",
}

quick_bindings = {
    [1] = "show_console",
    [2] = "delta_debug_enabled",
    [3] = "delta_radar_enabled",
	[4] = "show_settings_window",
}

states["quick_btn1"] = false
states["quick_btn2"] = false
states["quick_btn3"] = false
states["quick_btn4"] = false

function drawQuickBar()
    --if not menu.get("show_quickbar") then return end
	if not win.visible then return end
    if not quickbar_win.visible then return end

    local btn_size = 40
    local gap = 8
    local padding = 10

    local total_btns = 4
    local total_width = total_btns * btn_size + (total_btns - 1) * gap
    local panel_w = total_width + 2 * padding
    local panel_h = btn_size + 2 * padding

    local px, py = quickbar_win.x, quickbar_win.y
    local pw, ph = panel_w, panel_h

    -- Координаты кнопок
    local start_x = px + padding
    local start_y = py + padding
    local btn_rects = {}
    for i = 1, 4 do
        btn_rects[i] = {
            x = start_x + (i - 1) * (btn_size + gap),
            y = start_y,
            w = btn_size,
            h = btn_size,
        }
    end
    -- Перетаскивание: только если клик не по кнопке
    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    --local over_button = false
    --for _, rect in ipairs(btn_rects) do
        --if mx >= rect.x and mx <= rect.x + rect.w and my >= rect.y and my <= rect.y + rect.h then
            --over_button = true
            --break
        --end
    --end

    --if lmb then
        --if not quickbar_win.dragging and not over_button then
            --if mx >= px and mx <= px + pw and my >= py and my <= py + ph then
                --quickbar_win.dragging = true
                --quickbar_win.drag_ox = mx - px
                --quickbar_win.drag_oy = my - py
            --end
        --elseif quickbar_win.dragging then
            --quickbar_win.x = mx - quickbar_win.drag_ox
            --quickbar_win.y = my - quickbar_win.drag_oy
            --px, py = quickbar_win.x, quickbar_win.y
            --start_x = px + padding
            --start_y = py + padding
            --for i = 1, 3 do
                --btn_rects[i].x = start_x + (i - 1) * (btn_size + gap)
                --btn_rects[i].y = start_y
            --end
        --end
    --else
        --quickbar_win.dragging = false
    --end

    local radius = 5
    draw.RectFilled(px, py, pw, ph, STYLE.bg_main, radius)
    draw.Rect(px, py, pw, ph, STYLE.border, radius, 1.5)

    for i = 1, 4 do
        local id = "quick_btn" .. i
        local binding = quick_bindings[i]

        -- Синхронизация
        if binding and menu.get(binding) ~= nil then
            states[id] = menu.get(binding)
        end

        local state = states[id] or false
        local rect = btn_rects[i]
        local x, y = rect.x, rect.y

        -- Кнопка
        local color = state and STYLE.QB_COL_ON or STYLE.QB_COL_OFF
        draw.RectFilled(x, y, btn_size, btn_size, color, 4)
        draw.Rect(x, y, btn_size, btn_size, STYLE.border, 4, 1)

        -- Иконка
        local icon_type = quick_icons[i] or "text"
        local cx = x + btn_size / 2
        local cy = y + btn_size / 2
        local icon_size = btn_size * 0.5

        if icon_type == "Console" then
            local half = icon_size / 1.5
			local consoleicon = ">_"
            --draw.RectFilled(cx - half, cy - half, icon_size, icon_size, QB_COL_ICON, 0)
			draw.Rect(cx - half, cy - half, icon_size + 6.6, icon_size + 6, STYLE.QB_COL_ICON, 4, 2)
			draw.Line(cx + 12, cy - 6, cx - 12, cy - 6, STYLE.QB_COL_ICON, 2)
			draw.Text(cx-10, cy-6, consoleicon, STYLE.QB_COL_ICON, 13)
        elseif icon_type == "Radar" then
            draw.Circle(cx, cy, icon_size / 1.5, STYLE.QB_COL_ICON, 12, 2)
			draw.CircleFilled(cx, cy, icon_size / 1.5, STYLE.QB_COL_BG_ICON, 10)
			draw.Circle(cx, cy, icon_size / 2.5, STYLE.QB_COL_ICON, 9, 1)
			draw.Line(cx-12, cy, cx+12, cy, STYLE.QB_COL_ICON)
			draw.Line(cx, cy-12, cx, cy+12, STYLE.QB_COL_ICON)
			draw.Circle(cx, cy, icon_size / 6.5, STYLE.QB_COL_ICON, 6, 1)
			draw.Circle(cx+8, cy+5, icon_size / 8.5, STYLE.QB_COL_ICON, 6, 0)
			draw.Circle(cx+2, cy-7, icon_size / 8.5, STYLE.QB_COL_ICON, 6, 0)
        elseif icon_type == "rect" then
            local w = icon_size * 0.9
            local h = icon_size * 1.1
            draw.Rect(cx - w/2, cy - h/2, w, h, STYLE.QB_COL_ICON, 2, 2)
			draw.Line(cx + 9, cy - 6, cx - 9, cy - 6, STYLE.QB_COL_ICON, 2)
	    elseif icon_type == "Debug" then
            local half = icon_size / 1.5
			local consoleicon = "Deb"
            --draw.RectFilled(cx - half, cy - half, icon_size, icon_size, QB_COL_ICON, 0)
			draw.Rect(cx - half, cy - half, icon_size + 6.6, icon_size + 6, STYLE.QB_COL_ICON, 4, 2)
			draw.Line(cx + 12, cy - 6, cx - 12, cy - 6, STYLE.QB_COL_ICON, 2)
			draw.Text(cx-11, cy-4, consoleicon, STYLE.QB_COL_ICON, 13)
        elseif icon_type == "Settings" then
            draw.Circle(cx, cy, icon_size / 1.5, STYLE.QB_COL_ICON, 12, 2)
		elseif icon_type == "Find" then
            local half = icon_size / 1.5
			local consoleicon = "SET"
            --draw.RectFilled(cx - half, cy - half, icon_size, icon_size, QB_COL_ICON, 0)
			draw.Rect(cx - half, cy - half, icon_size + 6.6, icon_size + 6, STYLE.QB_COL_ICON, 4, 2)
			draw.Line(cx + 12, cy - 6, cx - 12, cy - 6, STYLE.QB_COL_ICON, 2)
			draw.Text(cx-11, cy-4, consoleicon, STYLE.QB_COL_ICON, 13)
        else  -- "text"
            local label = "[?]B" .. i
            local tw, th = draw.GetTextSize(label, 12)
            draw.Text(cx - tw/2, cy - th/2, label, STYLE.QB_COL_ICON, 12)
        end

        -- Полоска под активной кнопкой
        if state then
            local line_y = y + btn_size + 2
            draw.Line(x + 4, line_y, x + btn_size - 4, line_y, STYLE.QB_COL_LINE, 2)
        end

        -- Клик
        if lmb and not quickbar_win.dragging and not quickbar_win.clicked then
            if mx >= x and mx <= x + btn_size and my >= y and my <= y + btn_size then
                local new_state = not state
                states[id] = new_state
                if binding then
                    menu.set(binding, new_state)
                    if toggle_anim and toggle_anim[binding] then
                        toggle_anim[binding].target = new_state and 1 or 0
                        toggle_anim[binding].current = toggle_anim[binding].target
                    end
                end
                quickbar_win.clicked = true
                --addConsoleMessage("[QuickBar] " .. id .. " = " .. tostring(new_state), {0.6, 0.5, 1, 1})
            end
        end
    end

    if not lmb then
        quickbar_win.clicked = false
    end
end


local WIN_TITLE_H = 24
local WIN_PAD     = 8
local WIN_FONT    = 13

local WIN_COL_ACCENT  = { 0.78, 0.42, 0.84, 0.8 }
local WIN_COL_DIM     = { 0.25, 0.25, 0.30, 1.00 }
local WIN_COL_PLAYER  = { 1.00, 1.00, 1.00, 1.00 }

local RADAR_SIZE = 220
local RADAR_PAD = 12

local radar_win = {
    x = 1655,
    y = 60,
    w = RADAR_SIZE + 2 * RADAR_PAD,
    h = WIN_TITLE_H + 2 * RADAR_PAD + RADAR_SIZE,
    title = "Radar",
    visible = true,         
    dragging = false,
    drag_ox = 0,
    drag_oy = 0,
}

local radar_settings = {
    range = 500,
    opacity = 0.8,
    grid_size = 277,
    grid_offset_x = 74,          
    grid_offset_z = 221,         
    update_interval = 1,
    model_refresh_interval = 120,
    show_grid_labels = true,
    show_square_names = true,
    grid_start_x = -3500,
    grid_start_z = 3300,
}

local cached_players = {}
local cached_npcs = {}
local frame_counter = 0
local model_refresh_counter = 0
local npc_models = {}


local function refreshNpcModels()
    local aiZones = game.Workspace:FindFirstChild("AiZones")
    if not aiZones then
        npc_models = {}
        return
    end
    local new_models = {}
    local function scan(parent)
        for _, child in ipairs(parent:GetChildren()) do
            if child:IsA("Model") then
                table.insert(new_models, child)
            end
            scan(child)
        end
    end
    scan(aiZones)
    npc_models = new_models
end


local function getNpcPosition(model)
    local hum = model:FindFirstChildOfClass("Humanoid")
    if not hum or hum.Health <= 0 then return nil end
    local head = model:FindFirstChild("Head")
    if head and head:IsA("BasePart") then return head.Position end
    local root = model:FindFirstChild("HumanoidRootPart")
    if root and root:IsA("BasePart") then return root.Position end
    for _, child in ipairs(model:GetChildren()) do
        if child:IsA("BasePart") then return child.Position end
    end
    return nil
end

local function updateCache()
    local players = entity.GetPlayers()
    cached_players = {}
    for _, p in ipairs(players) do
        if p.IsAlive then
            local pos = p.Position
            if pos then
                table.insert(cached_players, { x = pos.X, y = pos.Y, z = pos.Z })
            end
        end
    end

    cached_npcs = {}
    for _, model in ipairs(npc_models) do
        local pos = getNpcPosition(model)
        if pos then
            table.insert(cached_npcs, { x = pos.X, y = pos.Y, z = pos.Z })
        end
    end
end

local function clipLineToRect(x1, y1, x2, y2, rx, ry, rw, rh)
    local dx = x2 - x1
    local dy = y2 - y1
    local t1, t2 = 0, 1
    local function clip(p, q)
        if p == 0 then
            if q < 0 then return false end
        else
            local t = q / p
            if p < 0 then
                if t > t1 then t1 = t end
            else
                if t < t2 then t2 = t end
            end
        end
        return true
    end
    if clip(-dx, x1 - rx) and clip(dx, rx + rw - x1) and
       clip(-dy, y1 - ry) and clip(dy, ry + rh - y1) then
        if t2 > t1 then
            return x1 + t1 * dx, y1 + t1 * dy, x1 + t2 * dx, y1 + t2 * dy, true
        end
    end
    return 0, 0, 0, 0, false
end

local function drawSquareNames(rad_x, rad_y, rad_w, rad_h, lpPos, range, scale, cos_a, sin_a, grid_size, offset_x, offset_z)
    if not radar_settings.show_square_names then return end
    local start_x = radar_settings.grid_start_x or 0
    local start_z = radar_settings.grid_start_z or 0

    local minX = lpPos.X - range
    local maxX = lpPos.X + range
    local minZ = lpPos.Z - range
    local maxZ = lpPos.Z + range

    local idx_min_x = math.floor((minX - start_x) / grid_size)
    local idx_max_x = math.floor((maxX - start_x) / grid_size)
    local idx_min_z = math.floor((minZ - start_z) / grid_size)
    local idx_max_z = math.floor((maxZ - start_z) / grid_size)

    local function indexToLetter(idx)
        local letter_idx = idx % 26
        if letter_idx < 0 then letter_idx = letter_idx + 26 end
        return string.char(65 + letter_idx) 
    end

    for ix = idx_min_x, idx_max_x do
        for iz = idx_min_z, idx_max_z do
            local wx = start_x + ix * grid_size
            local wz = start_z + iz * grid_size
            local dx = wx - lpPos.X
            local dz = wz - lpPos.Z
            local rx = dx * cos_a - dz * sin_a
            local rz = dx * sin_a + dz * cos_a
            local sx = rad_x + rad_w/2 + rx * scale
            local sy = rad_y + rad_h/2 + rz * scale

            if sx >= rad_x and sx <= rad_x + rad_w and sy >= rad_y and sy <= rad_y + rad_h then
                local number = ix
                local letter = indexToLetter(iz)
                local name = letter .. tostring(number)
                local margin = 2
                draw.Text(sx + margin, sy + margin, name, {0.8, 0.8, 0.9, 0.6}, 10)
            end
        end
    end
end


function drawRadarWindow()
    if not menu.get("delta_radar_enabled") then return end
	radar_settings.range = menu.get("radar_range") or 500

    frame_counter = frame_counter + 1
    model_refresh_counter = model_refresh_counter + 1
    if model_refresh_counter >= radar_settings.model_refresh_interval then
        model_refresh_counter = 0
        refreshNpcModels()
    end
    if frame_counter >= (radar_settings.update_interval or 1) then
        frame_counter = 0
        updateCache()
    end

    local px, py = radar_win.x, radar_win.y
    local pw = radar_win.w
    local ph = radar_win.h

    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    if lmb then
        if not radar_win.dragging then
            if mx >= px - 2 and mx <= px + pw + 2 and my >= py - 2 and my <= py + WIN_TITLE_H + 6 then
                radar_win.dragging = true
                radar_win.drag_ox = mx - px
                radar_win.drag_oy = my - py
            end
        else
            radar_win.x = mx - radar_win.drag_ox
            radar_win.y = my - radar_win.drag_oy
            px, py = radar_win.x, radar_win.y
        end
    else
        radar_win.dragging = false
    end


    draw.Rect(px-1, py-1, pw+2, ph+2, STYLE.border, 5, 1)
    draw.RectFilled(px, py, pw, ph, STYLE.bg_main, 4)
    draw.RectFilled(px, py, pw, WIN_TITLE_H, STYLE.bg_title, 4)
    draw.RectFilled(px, py, 2, WIN_TITLE_H, STYLE.border, 0)
    draw.Line(px, py + WIN_TITLE_H, px + pw, py + WIN_TITLE_H, STYLE.border, 1)
    draw.Rect(px, py, pw, ph, STYLE.border, 4, 1)

    draw.Text(px + WIN_PAD + 2, py + 5, radar_win.title, STYLE.accent, WIN_FONT)
    local hint = "drag ::"
    local hw, _ = draw.GetTextSize(hint, 10)
    draw.Text(px + pw - hw - WIN_PAD, py + 6, hint, STYLE.text_dim, 10)

    local rad_x = px + RADAR_PAD
    local rad_y = py + WIN_TITLE_H + RADAR_PAD
    local rad_w = RADAR_SIZE
    local rad_h = RADAR_SIZE

    local lp = entity.GetLocalPlayer()
    if not lp or not lp.IsAlive then
        draw.Text(rad_x + 10, rad_y + 10, "Game not found; if this message", WIN_COL_DIM, 12)
		draw.Text(rad_x + 10, rad_y + 25, "does not disappear after entering", WIN_COL_DIM, 12)
		draw.Text(rad_x + 10, rad_y + 40, "the game, restart the script.", WIN_COL_DIM, 12)
		addConsoleMessage("[RADAR] Error: Local Player not found", {1,0,0,1})
        return
    end

    local range = radar_settings.range
    local opacity = radar_settings.opacity
    local grid_size = radar_settings.grid_size
    local offset_x = radar_settings.grid_offset_x
    local offset_z = radar_settings.grid_offset_z

    local lpPos = lp.Position
    local camLook = camera.GetLookVector()
    local angle = math.atan2(-camLook.X, -camLook.Z)
    local cos_a = math.cos(angle)
    local sin_a = math.sin(angle)
    local scale = rad_w / (2 * range)

    -- Чёрный фон
    draw.RectFilled(rad_x, rad_y, rad_w, rad_h, {0, 0, 0, opacity * 0.9}, 2)
    draw.Rect(rad_x, rad_y, rad_w, rad_h, STYLE.border, 2, 1)


    local cx = rad_x + rad_w / 2
    local cy = rad_y + rad_h / 2



    local grid_color = {0.5, 0.5, 0.5, opacity * 0.4}
    local step = grid_size
    local minX = lpPos.X - range
    local maxX = lpPos.X + range
    local minZ = lpPos.Z - range
    local maxZ = lpPos.Z + range

    local function worldToRadar(wx, wz)
        local dx = wx - lpPos.X
        local dz = wz - lpPos.Z
        local rx = dx * cos_a - dz * sin_a
        local rz = dx * sin_a + dz * cos_a
        return cx + rx * scale, cy + rz * scale
    end

    local x0 = math.floor((minX - offset_x) / step) * step + offset_x
    for x = x0, maxX, step do
        if x >= minX and x <= maxX then
            local sx1, sy1 = worldToRadar(x, minZ)
            local sx2, sy2 = worldToRadar(x, maxZ)
            local cx1, cy1, cx2, cy2, ok = clipLineToRect(sx1, sy1, sx2, sy2, rad_x, rad_y, rad_w, rad_h)
            if ok then
                draw.Line(cx1, cy1, cx2, cy2, grid_color, 1)
            end
        end
    end


    local z0 = math.floor((minZ - offset_z) / step) * step + offset_z
    for z = z0, maxZ, step do
        if z >= minZ and z <= maxZ then
            local sx1, sy1 = worldToRadar(minX, z)
            local sx2, sy2 = worldToRadar(maxX, z)
            local cx1, cy1, cx2, cy2, ok = clipLineToRect(sx1, sy1, sx2, sy2, rad_x, rad_y, rad_w, rad_h)
            if ok then
                draw.Line(cx1, cy1, cx2, cy2, grid_color, 1)
            end
        end
    end

    draw.CircleFilled(cx, cy, 5, {0, 1, 0, opacity}, 12)
    draw.Circle(cx, cy, 6, {0, 0, 0, opacity}, 12, 1.5)


    for _, pos in ipairs(cached_players) do
        local dx = pos.x - lpPos.X
        local dz = pos.z - lpPos.Z
        local rx = dx * cos_a - dz * sin_a
        local rz = dx * sin_a + dz * cos_a
        local dist_sq = rx * rx + rz * rz
        if dist_sq <= range * range then
            local sx = cx + rx * scale
            local sy = cy + rz * scale
            if sx >= rad_x and sx <= rad_x + rad_w and sy >= rad_y and sy <= rad_y + rad_h then
                draw.CircleFilled(sx, sy, 4, {1, 0, 0, opacity}, 8)
                draw.Circle(sx, sy, 5, {0, 0, 0, opacity}, 8, 1)
            end
        end
    end


for _, pos in ipairs(cached_npcs) do
    local dx = pos.x - lpPos.X
    local dz = pos.z - lpPos.Z
    local rx = dx * cos_a - dz * sin_a
    local rz = dx * sin_a + dz * cos_a
    local dist_sq = rx * rx + rz * rz
    if dist_sq <= range * range then
        local sx = cx + rx * scale
        local sy = cy + rz * scale
        if sx >= rad_x and sx <= rad_x + rad_w and sy >= rad_y and sy <= rad_y + rad_h then
            local color = pos.color or {0, 0.5, 1, opacity}
            draw.CircleFilled(sx, sy, 5, color, 8)
            draw.Circle(sx, sy, 6, {0, 0, 0, opacity}, 8, 1)
        end
    end
end

    draw.Circle(cx, cy, rad_w / 2, {1, 1, 1, opacity * 0.3}, 32, 1)

   		 if menu.Get("show_grid_labels") then
        local font_small = 10
        local label_color = {0.8, 0.8, 0.9, 0.7}
        local function getCornerWorldCoords(corner)
            local x_offs, z_offs
            if corner == 1 then x_offs, z_offs = -range, -range
            elseif corner == 2 then x_offs, z_offs = -range, range
            elseif corner == 3 then x_offs, z_offs = range, -range
            else x_offs, z_offs = range, range end
            local rx = x_offs * cos_a - z_offs * sin_a
            local rz = x_offs * sin_a + z_offs * cos_a
            return lpPos.X + rx, lpPos.Z + rz
        end
        local x1, z1 = getCornerWorldCoords(1)
        local x2, z2 = getCornerWorldCoords(2)
        local x3, z3 = getCornerWorldCoords(3)
        local x4, z4 = getCornerWorldCoords(4)
        local margin = 4
        draw.Text(rad_x + margin, rad_y + margin, string.format("%.0f, %.0f", x1, z1), label_color, font_small)
        draw.Text(rad_x + rad_w - margin - draw.GetTextSize(string.format("%.0f, %.0f", x2, z2), font_small), rad_y + margin, string.format("%.0f, %.0f", x2, z2), label_color, font_small)
        draw.Text(rad_x + margin, rad_y + rad_h - margin - 12, string.format("%.0f, %.0f", x3, z3), label_color, font_small)
        draw.Text(rad_x + rad_w - margin - draw.GetTextSize(string.format("%.0f, %.0f", x4, z4), font_small), rad_y + rad_h - margin - 12, string.format("%.0f, %.0f", x4, z4), label_color, font_small)
    end

drawSquareNames(rad_x, rad_y, rad_w, rad_h, lpPos, range, scale, cos_a, sin_a, grid_size, offset_x, offset_z)

    local info = "Players: " .. #cached_players .. " | NPC: " .. #cached_npcs
    local tw, th = draw.GetTextSize(info, 11)
    draw.Text(cx - tw/2, rad_y + rad_h - th - 4, info, STYLE.accent , 11)
end


refreshNpcModels()
updateCache()

-- inv checker


local inv_panel = { x = 1640, y = 340, w = 260, dragging = false, drag_ox = 0, drag_oy = 0 }
local TITLE_H  = 24
local PAD      = 10
local LINE_H   = 16
local LINE_H_L = 20
local FONT     = 13
local FONT_L   = 14
local SEC_H    = 18

local col_dim     = { 0.25, 0.25, 0.30, 1.00 }
local col_sec_bg  = { 0.08, 0.08, 0.11, 1.00 }
local col_sec_line= { 0.58,0.22,0.84,0.5     }
local col_weapon  = { 0.95, 0.30, 0.30, 1.0  }
local col_sight   = { 0.459, 0, 1, 1.0       }
local col_armor   = { 0.18, 0.32, 0.84, 1.0  }
local col_val     = { 0.35, 0.90, 0.45, 1.0  }
local col_serun   = { 1, 1, 1, 1.0           }
local col_junk    = { 0.45, 0.45, 0.50, 1.0  }
local col_sec_gun = { 0.95, 0.30, 0.30, 1.0  }
local col_sec_sight = { 0.341, 0, 1, 1.0     }
local col_sec_arm = { 0.28, 0.472, 0.984, 1.0}
local col_sec_val = { 0.35, 0.90, 0.45, 1.0  }
local col_sec_serun = { 1, 1, 1, 1.0  }
local col_sec_rst = { 0.40, 0.40, 0.45, 1.0  }

local weapons        = { "AsVal", "FAL", "AKMN", "RPG7", "M4", "PKM", "R700", "SVD", "TFZ98S", "Saiga", "NagantRevolver", "Groza" }
local sights         = { "T12W", "Reapir", "PEQ15", "Mag556Rnd100", "SOCOM556", "RK2", "RK6", "StubbyGrip", "AFG"}
local armor_list     = { "Altyn", "MaskaVisor", "Pantsir3", "GP7", "LegArmor", "UNOHelmet", "HeadMount", "UNOVest", "TORS", "GP5", "6B5", "6B43", "FastVisor", "HSPV", "JPC", "QuadNVG", "ScavKingHelment", "ScavKingVest", "IOTV4", "TitanShield", "DozerArmor", "FastMT", "Kulon", "ZSH" }
local valuables_list = { "FlareGun", "MaskaVisor", "GoldSkull", "TitanShield", "FastVisor", "CatToken", "SPSh44", "QuadNVG", "GPU", "Gold50g", "GoldWatch", "GoldTicket", "RepairKit" }


local function is_weapon(name)
    local n = name:lower()
    for _, p in ipairs(weapons) do if n == p:lower() then return true end end
    return false
end

local function is_sights(name)
    local n = name:lower()
    for _, p in ipairs(sights) do if n == p:lower() then return true end end
    return false
end


local function is_armor(name)
    local n = name:lower()
    for _, p in ipairs(armor_list) do if n:find(p:lower(), 1, true) then return true end end
    return false
end

local function is_valuable(name)
    local n = name:lower()
    for _, p in ipairs(valuables_list) do if n:find(p:lower(), 1, true) then return true end end
    return false
end

local function is_serun(name)
    local n = name:lower()
    if n:find("serum", 1, true) then return true end
    return false
end


local function get_item_color(name)
    if is_valuable(name) then return col_val end
    if is_serun(name)    then return col_serun end
    if is_weapon(name)   then return col_weapon end
    if is_sights(name)   then return col_sight end
    if is_armor(name)    then return col_armor end
    return col_junk
end

local cached_players = {}

local function scan_players_inv()
	thread_status.scan_players_inv.last_run = utility.GetTime()
    thread_status.scan_players_inv.count = thread_status.update_world.count + 1
    thread_status.scan_players_inv.status = "Run"
    local result = {}
    local players = entity.get_players()
    if not players then return end
    local rep = game.ReplicatedStorage
    local rep_players = rep and rep:find_first_child("Players")
    for i = 1, #players do
        local p = players[i]
        if p and p.is_valid and not p.is_local then
            local pname = p.name or ""
            local puid  = tostring(p.user_id or "")
            local inv = {}
            local worn = {}
            local seen = {}
            local equipped = nil
            local function add(name, is_top)
                if name ~= "" and not seen[name] then
                    seen[name] = true
                    inv[#inv+1] = name
                    if is_top then worn[name] = true end
                end
            end
            local function scan_folder(folder, depth)
                if not folder then return end
                if depth and depth > 8 then return end
                local ok2, items = pcall(function() return folder:get_children() end)
                if not ok2 or not items then return end
                for j = 1, #items do
                    local item = items[j]
                    local name = item.name or ""
                    if name ~= "" then add(name, depth == 0) end
                    local ok3, children = pcall(function() return item:get_children() end)
                    if ok3 and children and #children > 0 then scan_folder(item, (depth or 0) + 1) end
                end
            end
            if rep_players then
                local data = rep_players:find_first_child(pname)
                if not data and puid ~= "" then
                    data = rep_players:find_first_child(puid)
                end
                if not data then
                    local ok2, folders = pcall(function() return rep_players:get_children() end)
                    if ok2 and folders then
                        for _, folder in ipairs(folders) do
                            if folder.name == pname or folder.name == puid then
                                data = folder
                                break
                            end
                        end
                    end
                end
                if data then
                    scan_folder(data:find_first_child("Inventory"), 0)
                    local status = data:find_first_child("Status")
                    if status then
                        local gv = status:find_first_child("GameplayVariables")
                        if gv then
                            local eq = gv:find_first_child("EquippedTool")
                            if eq and eq.value then
                                local val = eq.value
                                if type(val) == "string" and val ~= "" then equipped = val
                                elseif type(val) == "userdata" then equipped = val.name end
                            end
                        end
                    end
                end
            end
            result[#result+1] = { name = pname, entity = p, inventory = inv, worn = worn, equipped = equipped }
        end
    end
    cached_players = result
end

local function get_closest_to_crosshair()
    if #cached_players == 0 then return nil end
    local sw, sh = utility.get_screen_size()
    local scx, scy = sw * 0.5, sh * 0.5
    local best, best_dist = nil, math.huge
    for _, d in ipairs(cached_players) do
        local p = d.entity
        if p and p.is_valid and p.is_alive then
            local pos = p.position
            if pos then
                local sx, sy, vis = utility.world_to_screen(pos.x, pos.y, pos.z)
                if vis then
                    local dx, dy = sx - scx, sy - scy
                    local dist = math.sqrt(dx*dx + dy*dy)
                    if dist < best_dist then best_dist = dist best = d end
                end
            end
        end
    end
    return best
end

local inv_toggled = false
local inv_last_key = false

local function should_show_inv()
    if not safe_menu_get("delta_inv_enabled") then return false end
    local mode = safe_menu_get("delta_inv_mode") or 0
    local key = safe_menu_get_key("delta_inv_enabled")
    if key == 0 then key = 2 end
    if mode == 0 then return true
    elseif mode == 1 then
        local down = input.is_key_down(key)
        if down and not inv_last_key then inv_toggled = not inv_toggled end
        inv_last_key = down
        return inv_toggled
    else
        inv_last_key = false
        return input.is_key_down(key)
    end
end

local function draw_section_header(px, ly, pw, label, col_left)
    draw.rect_filled(px, ly, pw, SEC_H, col_sec_bg, 0)
    draw.rect_filled(px, ly, 2, SEC_H, col_left, 0)
    draw.line(px, ly + SEC_H, px + pw, ly + SEC_H, col_sec_line, 1)
    local tw = draw.get_text_size(label, 10)
    draw.text(px + (pw - tw) * 0.5, ly + 4, label, col_left, 10)
    return ly + SEC_H
end

local function clamp_list(list, max)
    if #list <= max then return list end
    local out = {}
    for i = 1, max do out[i] = list[i] end
    return out
end

local function draw_inv_panel()
    if not should_show_inv() then inv_panel.dragging = false return end
    local show_guns  = safe_menu_get("delta_inv_show_guns")
    local show_sights  = safe_menu_get("delta_inv_show_sights")
    local show_armor = safe_menu_get("delta_inv_show_armor")
    local show_val   = safe_menu_get("delta_inv_show_val")
    local show_serun   = safe_menu_get("delta_inv_show_serun")
    local show_other = safe_menu_get("delta_inv_show_other")
    local max_guns   = safe_menu_get("delta_inv_max_guns")  or 10
    local max_sights   = safe_menu_get("delta_inv_max_sights")  or 10
    local max_armor  = safe_menu_get("delta_inv_max_armor") or 10
    local max_val    = safe_menu_get("delta_inv_max_val")   or 10
    local max_serun    = safe_menu_get("delta_inv_max_serun")   or 4
    local max_other  = safe_menu_get("delta_inv_max_other") or 5
    local target = get_closest_to_crosshair()
    local equipped_item = nil
    local weapon_items, sight_items, armor_items, valuable_items, seruns, other_items = {}, {}, {}, {}, {}, {}
    if target then
        equipped_item = target.equipped
        for _, item in ipairs(target.inventory) do
            if is_valuable(item) then valuable_items[#valuable_items+1] = item
            elseif is_serun(item) then seruns[#seruns+1] = item
            elseif is_weapon(item) then weapon_items[#weapon_items+1] = item
            elseif is_sights(item) then sight_items[#sight_items+1] = item
            elseif is_armor(item) and target.worn[item] then armor_items[#armor_items+1] = item
            elseif not is_armor(item) then other_items[#other_items+1] = item end
        end
    end
    weapon_items   = clamp_list(weapon_items,   max_guns)
    sight_items   = clamp_list(sight_items,   max_sights)
    armor_items    = clamp_list(armor_items,    max_armor)
    valuable_items = clamp_list(valuable_items, max_val)
    seruns = clamp_list(seruns, max_serun)
    other_items    = clamp_list(other_items,    max_other)
    local panel_h = TITLE_H + PAD
    if equipped_item and equipped_item ~= "" then panel_h = panel_h + LINE_H_L + PAD + 14 end
    if show_guns  and #weapon_items   > 0 then panel_h = panel_h + SEC_H + (#weapon_items   * LINE_H_L) + 4 end
    if show_sights  and #sight_items   > 0 then panel_h = panel_h + SEC_H + (#sight_items   * LINE_H_L) + 4 end
    if show_armor and #armor_items    > 0 then panel_h = panel_h + SEC_H + (#armor_items    * LINE_H_L) + 4 end
    if show_val   and #valuable_items > 0 then panel_h = panel_h + SEC_H + (#valuable_items * LINE_H_L) + 4 end
    if show_serun   and #seruns > 0 then panel_h = panel_h + SEC_H + (#seruns * LINE_H_L) + 4 end
    if show_other and #other_items    > 0 then panel_h = panel_h + SEC_H + (#other_items    * LINE_H)   + 4 end
    if not target then panel_h = panel_h + LINE_H end
    panel_h = panel_h + PAD
    local pw = inv_panel.w
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not inv_panel.dragging then
            if mx >= inv_panel.x and mx <= inv_panel.x + pw and my >= inv_panel.y and my <= inv_panel.y + TITLE_H then
                inv_panel.dragging = true inv_panel.drag_ox = mx - inv_panel.x inv_panel.drag_oy = my - inv_panel.y
            end
        else inv_panel.x = mx - inv_panel.drag_ox inv_panel.y = my - inv_panel.drag_oy end
    else inv_panel.dragging = false end
    local px, py = inv_panel.x, inv_panel.y
    draw.rect(px-1, py-1, pw+2, panel_h+2, STYLE.border, 5, 1)
    draw.rect_filled(px, py, pw, panel_h, STYLE.bg_main, 4)
    draw.rect_filled(px, py, pw, TITLE_H, STYLE.bg_title, 4)
    draw.rect_filled(px, py, 2, TITLE_H, STYLE.border, 0)
    draw.line(px, py+TITLE_H, px+pw, py+TITLE_H, STYLE.border, 1)
    draw.rect(px, py, pw, panel_h, STYLE.border, 4, 1)
    local title = target and target.name or "Inventory Checker"
    draw.text(px+10, py+5, title, STYLE.accent, FONT)
    local hint = "drag ::"
    local hw = draw.get_text_size(hint, 10)
    draw.text(px+pw-hw-PAD, py+6, hint, STYLE.text_dim, 10)
    local ly = py + TITLE_H + PAD
    if not target then draw.text(px+PAD, ly, "No players visible", col_junk, FONT) return end
    if equipped_item and equipped_item ~= "" then
        local col = get_item_color(equipped_item)
        draw.text(px+PAD, ly, "EQUIPPED", STYLE.text_dim, 9)
        draw.text(px+PAD, ly+11, equipped_item, col, FONT_L)
        ly = ly + LINE_H_L + PAD
        draw.line(px+PAD, ly, px+pw-PAD, ly, col_sec_line, 1)
        ly = ly + 4
    end
    if show_guns and #weapon_items > 0 then
        ly = draw_section_header(px, ly, pw, "GUN(S)", col_sec_gun)
        for _, item in ipairs(weapon_items) do draw.text(px+PAD+6, ly+2, item, col_weapon, FONT_L) ly = ly+LINE_H_L end
        ly = ly + 4
    end
    if show_sights and #sight_items > 0 then
        ly = draw_section_header(px, ly, pw, "ATTACHMENT(S)", col_sec_sight)
        for _, item in ipairs(sight_items) do draw.text(px+PAD+6, ly+2, item, col_sight, FONT_L) ly = ly+LINE_H_L end
        ly = ly + 4
    end
    if show_armor and #armor_items > 0 then
        ly = draw_section_header(px, ly, pw, "ARMOR", col_sec_arm)
        for _, item in ipairs(armor_items) do draw.text(px+PAD+6, ly+2, item, col_armor, FONT_L) ly = ly+LINE_H_L end
        ly = ly + 4
    end
    if show_val and #valuable_items > 0 then
        ly = draw_section_header(px, ly, pw, "VALUABLES", col_sec_val)
        for _, item in ipairs(valuable_items) do draw.text(px+PAD+6, ly+2, item, col_val, FONT_L) ly = ly+LINE_H_L end
        ly = ly + 4
    end
    if show_serun and #seruns > 0 then
        ly = draw_section_header(px, ly, pw, "SERUM-X", col_sec_serun)
        for _, item in ipairs(seruns) do draw.text(px+PAD+6, ly+2, item, col_serun, FONT_L) ly = ly+LINE_H_L end
        ly = ly + 4
    end
    if show_other and #other_items > 0 then
        ly = draw_section_header(px, ly, pw, "OTHER", col_sec_rst)
        for _, item in ipairs(other_items) do draw.text(px+PAD+6, ly+2, item, col_junk, FONT) ly = ly+LINE_H end
    end
end



-- boss tracker


local boss_panel = { x = 305, y = 70, w = 170, dragging = false, drag_ox = 0, drag_oy = 0 }
local BOSS_TITLE_H = 24
local BOSS_PAD     = 8
local BOSS_ROW_H   = 18
local BOSS_SEC_H   = 16
local BOSS_FONT    = 12

local bcol_bg      = {0.09, 0.02, 0.09, 0.96}
local bcol_title   = {0.07, 0.07, 0.10, 1.00}
local bcol_border  = {0.58,0.22,0.84,0.5}
local bcol_accent  = {0.78,0.42,0.84,0.8}
local bcol_dim     = {0.25, 0.25, 0.30, 1.00}
local bcol_sec_bg  = {0.09, 0.02, 0.09, 0.96}
local bcol_sec_line= {0.58,0.22,0.84,0.5}
local bcol_alive   = { 0.47, 1.00, 0.63, 1.00 }
local bcol_warn    = { 1.00, 0.78, 0.24, 1.00 }
local bcol_dead    = { 1.00, 0.31, 0.31, 1.00 }
local bcol_none    = { 0.47, 0.47, 0.50, 1.00 }
local bcol_vehicle = { 1.00, 0.85, 0.20, 1.00 }
local bcol_white   = { 1.00, 1.00, 1.00, 1.00 }

local boss_defs = {
    { zone = "Sawmill",    name = "Anton"    },
    { zone = "Factory",    name = "Dozer"    },
    { zone = "Whisper",    name = "Whisper"  },
    { zone = "Death",      name = "Death"    },
    { zone = "ScavCastle", name = "ScavKing" },
}

local vehicle_defs = {
    { name = "MI24V", label = "Mi-24V Hind" },
    { name = "BTR80", label = "BTR-80"       },
}

local boss_data    = {}
local vehicle_data = {}

local function update_bosses()
	thread_status.update_bosses.last_run = utility.GetTime()
    thread_status.update_bosses.count = thread_status.update_world.count + 1
    thread_status.update_bosses.status = "Run"
    local ws = game.workspace
    if not ws then return end
    local ai_zones = ws:find_first_child("AiZones")
    for i, def in ipairs(boss_defs) do
        local entry = { name = def.name, status = "none", hp = 0, max_hp = 0, dist = -1 }
        if ai_zones then
            local zone = ai_zones:find_first_child(def.zone)
            local boss = zone and zone:find_first_child(def.name)
            if boss then
                local hum = boss:find_first_child_of_class("Humanoid")
                if hum then
                    local hp = hum.health or 0
                    local max_hp = hum.max_health or 100
                    if hp > 0 then
                        entry.status = "alive"
                        entry.hp = math.floor(hp)
                        entry.max_hp = math.floor(max_hp)
                        local me = entity.get_local_player()
                        local hrp = boss:find_first_child("HumanoidRootPart")
                        if me and hrp then
                            local mpos = me.position
                            local bpos = hrp.position
                            if mpos and bpos then
                                local dx = mpos.x - bpos.x
                                local dy = mpos.y - bpos.y
                                local dz = mpos.z - bpos.z
                                entry.dist = math.floor(math.sqrt(dx*dx + dy*dy + dz*dz))
                            end
                        end
                    else entry.status = "dead" end
                else entry.status = "dead" end
            end
        end
        boss_data[i] = entry
    end
    for i, def in ipairs(vehicle_defs) do
        local entry = { name = def.label, status = "none", dist = -1 }
        local veh = ws:find_first_child(def.name)
        if not veh and ai_zones then
            local ok2, zones = pcall(function() return ai_zones:get_children() end)
            if ok2 and zones then
                for _, z in ipairs(zones) do
                    local found = z:find_first_child(def.name)
                    if found then veh = found break end
                end
            end
        end
        if veh then
            entry.status = "spawned"
            local root = veh:find_first_child("HumanoidRootPart")
            if not root then
                local ok2, children = pcall(function() return veh:get_children() end)
                if ok2 and children then
                    for _, c in ipairs(children) do
                        if c:is_a("BasePart") then root = c break end
                    end
                end
            end
            local me = entity.get_local_player()
            if me and root then
                local mpos = me.position
                local vpos = root.position
                if mpos and vpos then
                    local dx = mpos.x - vpos.x
                    local dy = mpos.y - vpos.y
                    local dz = mpos.z - vpos.z
                    entry.dist = math.floor(math.sqrt(dx*dx + dy*dy + dz*dz))
                end
            end
        end
        vehicle_data[i] = entry
    end
end

local function draw_boss_section_header(px, ly, pw, label)
    draw.rect_filled(px, ly, pw, BOSS_SEC_H, bcol_sec_bg, 0)
    draw.rect_filled(px, ly, 2, BOSS_SEC_H, bcol_accent, 0)
    draw.line(px, ly + BOSS_SEC_H, px + pw, ly + BOSS_SEC_H, bcol_sec_line, 1)
    local tw = draw.get_text_size(label, 9)
    draw.text(px + (pw - tw) * 0.5, ly + 3, label, bcol_accent, 9)
    return ly + BOSS_SEC_H
end

local function hp_color(hp, max_hp)
    if max_hp <= 0 then return bcol_none end
    local pct = hp / max_hp
    if pct > 0.5 then return bcol_alive end
    if pct > 0.2 then return bcol_warn  end
    return bcol_dead
end

local function draw_boss_panel()
    if not safe_menu_get("delta_boss_enabled") then boss_panel.dragging = false return end
    local num_boss_rows    = #boss_defs
    local num_vehicle_rows = #vehicle_defs
    local panel_h = BOSS_TITLE_H
                  + BOSS_SEC_H + (num_boss_rows    * BOSS_ROW_H) + 4
                  + BOSS_SEC_H + (num_vehicle_rows * BOSS_ROW_H) + 4
                  + BOSS_PAD
    local pw = boss_panel.w
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not boss_panel.dragging then
            if mx >= boss_panel.x and mx <= boss_panel.x + pw and my >= boss_panel.y and my <= boss_panel.y + BOSS_TITLE_H then
                boss_panel.dragging = true boss_panel.drag_ox = mx - boss_panel.x boss_panel.drag_oy = my - boss_panel.y
            end
        else boss_panel.x = mx - boss_panel.drag_ox boss_panel.y = my - boss_panel.drag_oy end
    else boss_panel.dragging = false end
    local px, py = boss_panel.x, boss_panel.y
    draw.rect(px-1, py-1, pw+2, panel_h+2, {0.58,0.22,0.84,0.5}, 5, 1)
    draw.rect_filled(px, py, pw, panel_h, bcol_bg, 4)
    draw.rect_filled(px, py, pw, BOSS_TITLE_H, bcol_title, 4)
    draw.rect_filled(px, py, 2, BOSS_TITLE_H, bcol_border, 0)
    draw.line(px, py+BOSS_TITLE_H, px+pw, py+BOSS_TITLE_H, bcol_border, 1)
    draw.rect(px, py, pw, panel_h, {0.58,0.22,0.84,0.5}, 4, 1)
    draw.text(px+10, py+5, "Boss Tracker", bcol_accent, BOSS_FONT)
    local hint = "drag ::"
    local hw = draw.get_text_size(hint, 10)
    draw.text(px+pw-hw-BOSS_PAD, py+6, hint, bcol_dim, 10)
    local ly = py + BOSS_TITLE_H
    ly = draw_boss_section_header(px, ly, pw, "BOSSES")
    for i, entry in ipairs(boss_data) do
        local row_y = ly + (i-1) * BOSS_ROW_H
        draw.text(px+BOSS_PAD+4, row_y+3, entry.name, bcol_white, BOSS_FONT)
        local status_txt, status_col
        if entry.status == "alive" then
            local pct = entry.max_hp > 0 and math.floor(entry.hp / entry.max_hp * 100) or 0
            status_txt = string.format("%d/%d (%d%%)", entry.hp, entry.max_hp, pct)
            if entry.dist >= 0 then status_txt = status_txt .. "  " .. entry.dist .. "m" end
            status_col = hp_color(entry.hp, entry.max_hp)
        elseif entry.status == "dead" then
            status_txt = "Dead" status_col = bcol_dead
        else
            status_txt = "Not Spawned" status_col = bcol_none
        end
        local sw = draw.get_text_size(entry.name, BOSS_FONT)
        draw.text(px+BOSS_PAD+4+sw+6, row_y+3, status_txt, status_col, BOSS_FONT)
    end
    ly = ly + (num_boss_rows * BOSS_ROW_H) + 4
    ly = draw_boss_section_header(px, ly, pw, "VEHICLES")
    for i, entry in ipairs(vehicle_data) do
        local row_y = ly + (i-1) * BOSS_ROW_H
        draw.text(px+BOSS_PAD+4, row_y+3, entry.name, bcol_white, BOSS_FONT)
        local vstatus_txt, vstatus_col
        if entry.status == "spawned" then
            vstatus_txt = "Spawned"
            if entry.dist >= 0 then vstatus_txt = vstatus_txt .. "  " .. entry.dist .. "m" end
            vstatus_col = bcol_vehicle
        else
            vstatus_txt = "Not Spawned" vstatus_col = bcol_none
        end
        local sw = draw.get_text_size(entry.name, BOSS_FONT)
        draw.text(px+BOSS_PAD+4+sw+6, row_y+3, vstatus_txt, vstatus_col, BOSS_FONT)
    end
end


-- esp


local world = { extracts={}, items={}, corpses={}, containers={}, quests={}, npcs={}, cam_x=0, cam_y=0, cam_z=0 }
local folders = { workspace=nil, exfils=nil, drops=nil, containers=nil, quests=nil, ai_zones=nil, last_update=0 }
local cfg = {}
local colors = { gray = { 0.7, 0.7, 0.7, 1 } }

local skeleton_pairs = {
    {"Head","UpperTorso"},{"UpperTorso","LowerTorso"},
    {"UpperTorso","LeftUpperArm"},{"UpperTorso","RightUpperArm"},
    {"LeftUpperArm","LeftLowerArm"},{"RightUpperArm","RightLowerArm"},
    {"LeftLowerArm","LeftHand"},{"RightLowerArm","RightHand"},
    {"LowerTorso","LeftUpperLeg"},{"LowerTorso","RightUpperLeg"},
    {"LeftUpperLeg","LeftLowerLeg"},{"RightUpperLeg","RightLowerLeg"},
    {"LeftLowerLeg","LeftFoot"},{"RightLowerLeg","RightFoot"}
}

local aim_bones = { [0]="Head", [1]="UpperTorso", [2]="LowerTorso" }

local function is_valid(inst)
    if not inst then return false end
    local ok, parent = pcall(function() return inst.parent end)
    return ok and parent ~= nil
end

local function get_root(model)
    if not model then return nil end
    return model:find_first_child("Root")
        or model:find_first_child("MainPart")
        or model:find_first_child_of_class("BasePart")
end

local function get_dist(x1,y1,z1,x2,y2,z2)
    local dx,dy,dz = x1-x2, y1-y2, z1-z2
    return sqrt(dx*dx+dy*dy+dz*dz)
end

local function refresh_folders()
    local now = utility.get_tick_count()
    if folders.workspace and (now - folders.last_update) < 500 then return true end
    folders.last_update = now
    local ws = game.workspace
    if not ws then return false end
    if folders.workspace ~= ws or not is_valid(folders.workspace) then
        folders.workspace = ws
        folders.exfils=nil folders.drops=nil folders.containers=nil folders.quests=nil folders.ai_zones=nil
        world.extracts={} world.items={} world.corpses={} world.containers={} world.quests={} world.npcs={}
    end
    if not folders.exfils or not is_valid(folders.exfils) then
        local no_collision = ws:find_first_child("NoCollision")
        folders.exfils = no_collision and no_collision:find_first_child("ExitLocations")
    end
    if not folders.drops or not is_valid(folders.drops) then
        folders.drops = ws:find_first_child("DroppedItems")
    end
    if not folders.containers or not is_valid(folders.containers) then
        folders.containers = ws:find_first_child("Containers")
    end
    if not folders.quests or not is_valid(folders.quests) then
        folders.quests = ws:find_first_child("QuestItems")
    end
    if not folders.ai_zones or not is_valid(folders.ai_zones) then
        folders.ai_zones = ws:find_first_child("AiZones")
    end
    return true
end

local function scan_extracts()
    if not folders.exfils then world.extracts={} return end
    local result = {}
    local ok, children = pcall(function() return folders.exfils:get_children() end)
    if not ok or not children then return end
    for i = 1, #children do
        local model = children[i]
        local part = model:find_first_child_of_class("BasePart")
        if not part then
            local ok2, sub = pcall(function() return model:get_children() end)
            if ok2 and sub then
                for _, c in ipairs(sub) do
                    if c:is_a("BasePart") then part = c break end
                end
            end
        end
        if part then result[#result+1] = { name = model.name, part = part } end
    end
    world.extracts = result
end

local function get_inventory_contents(model)
    local ok, inv = pcall(function() return model:find_first_child("Inventory") end)
    if not ok or not inv then return nil end
    local ok2, children = pcall(function() return inv:get_children() end)
    if not ok2 or not children or #children == 0 then return nil end
    local counts = {}
    for i = 1, #children do
        local item = children[i]
        if item.name ~= "" then
            counts[item.name] = (counts[item.name] or 0) + 1
        end
    end
    local out = {}
    for name, count in pairs(counts) do
        out[#out+1] = count > 1 and (name .. " x" .. count) or name
    end
    return #out > 0 and out or nil
end

local function scan_items()
    world.items = {}
    world.corpses = {}
    if not folders.drops then return end
    local items_result = {}
    local corpses_result = {}
    local function process_model(model)
        if model.class_name ~= "Model" or model.name == "" then return end
        local part = get_root(model)
        if not part then return end
        local ok, hum = pcall(function() return model:find_first_child_of_class("Humanoid") end)
        if ok and hum then
            local ok2, hp = pcall(function() return hum.health end)
            if ok2 and hp and hp <= 0 then
                local contents = get_inventory_contents(model)
                corpses_result[#corpses_result+1] = { name = model.name, part = part, contents = contents }
                return
            end
        end
        items_result[#items_result+1] = { name = model.name, part = part }
    end
    local ok, children = pcall(function() return folders.drops:get_children() end)
    if not ok or not children then return end
    for i = 1, #children do
        local child = children[i]
        if child.class_name == "Folder" then
            local ok2, items = pcall(function() return child:get_children() end)
            if ok2 and items then
                for j = 1, #items do
                    process_model(items[j])
                end
            end
        elseif child.class_name == "Model" then
            process_model(child)
        end
    end
    world.items = items_result
    world.corpses = corpses_result
end

local function scan_containers()
    if not folders.containers then world.containers={} return end
    local result = {}
    local function process(model)
        if model.class_name~="Model" or model.name=="" then return end
        local part = get_root(model)
        if not part then return end
        local inv = model:find_first_child("Inventory")
        if not inv then return end
        local ok2, children = pcall(function() return inv:get_children() end)
        if not ok2 or not children or #children==0 then return end
        local counts = {}
        for i=1,#children do
            local item=children[i]
            if item.name~="" then counts[item.name]=(counts[item.name] or 0)+1 end
        end
        local contents={}
        for name,count in pairs(counts) do contents[#contents+1]=count>1 and (name.." x"..count) or name end
        result[#result+1]={name=model.name,part=part,contents=contents}
    end
    local ok, children = pcall(function() return folders.containers:get_children() end)
    if not ok or not children then return end
    for i=1,#children do
        local child=children[i]
        if child.class_name=="Folder" then
            local ok2, models = pcall(function() return child:get_children() end)
            if ok2 and models then for j=1,#models do process(models[j]) end end
        else process(child) end
    end
    world.containers = result
end

local function scan_quests()
    if not folders.quests then world.quests={} return end
    local result = {}
    local function process(model)
        if model.class_name~="Model" or model.name=="" then return end
        local part = get_root(model)
        if part then result[#result+1]={name=model.name,part=part} end
    end
    local ok, children = pcall(function() return folders.quests:get_children() end)
    if not ok or not children then return end
    for i=1,#children do
        local child=children[i]
        if child.class_name=="Folder" then
            local ok2, models = pcall(function() return child:get_children() end)
            if ok2 and models then for j=1,#models do process(models[j]) end end
        else process(child) end
    end
    world.quests = result
end

local function scan_npcs()
    if not folders.ai_zones then world.npcs={} return end
    local result = {}
    local ok, zones = pcall(function() return folders.ai_zones:get_children() end)
    if not ok or not zones then return end
    for i=1,#zones do
        local zone=zones[i]
        if zone.class_name=="Folder" then
            local ok2, npcs = pcall(function() return zone:get_children() end)
            if ok2 and npcs then
                for j=1,#npcs do
                    local npc=npcs[j]
                    if npc.class_name=="Model" and npc.name~="" then
                        local hum=npc:find_first_child_of_class("Humanoid")
                        local head=npc:find_first_child("Head")
                        if hum and head then
                            local hp=hum.health
                            if hp and hp>0 then
                                local parts={}
                                local ok3, npc_children = pcall(function() return npc:get_children() end)
                                if ok3 and npc_children then
                                    for k=1,#npc_children do
                                        local p=npc_children[k]
                                        if p:is_a("BasePart") then
                                            local size=p.size
                                            if size then parts[p.name]={part=p,hx=size.x*0.5,hy=size.y*0.5,hz=size.z*0.5} end
                                        end
                                    end
                                end
                                result[#result+1]={name=npc.name,hum=hum,head=head,parts=parts}
                            end
                        end
                    end
                end
            end
        end
    end
    world.npcs = result
end

local function update_world()
	thread_status.update_world.last_run = utility.GetTime()
    thread_status.update_world.count = thread_status.update_world.count + 1
    thread_status.update_world.status = "Run"
    if not refresh_folders() then return end
    scan_extracts() scan_items() scan_containers() scan_quests()
end

local function update_npcs()
	thread_status.update_npcs.last_run = utility.GetTime()
    thread_status.update_npcs.count = thread_status.update_world.count + 1
    thread_status.update_npcs.status = "Run"
    if not refresh_folders() then return end
    scan_npcs()
end

local function update_camera()
    local pos = camera.get_position()
    if pos then world.cam_x=pos.x world.cam_y=pos.y world.cam_z=pos.z end
end

local function cache_settings()
    -- Player ESP
    cfg.player          = safe_menu_get("delta_player_enabled")
    cfg.esp             = safe_menu_get("delta_esp_enabled")
	cfg.player_box = safe_menu_get("esp_player_box")
	cfg.box_style = safe_menu_get("esp_box_style") or 0
	cfg.box_color = safe_menu_get_color("esp_box_color") or {1,0,0,1}
	cfg.player_skeleton = safe_menu_get("esp_skeleton")
	cfg.player_skeleton_col = safe_menu_get_color("esp_skeleton_color") or {1, 1, 1, 1}
    cfg.player_dist     = safe_menu_get("delta_player_dist")          -- FIX: read player dist independently
    cfg.player_dist_col = safe_menu_get_color("delta_player_dist")
    cfg.player_range    = safe_menu_get("delta_player_range")
    -- Extracts
    cfg.extract         = safe_menu_get("delta_extract_enabled")
    cfg.extract_col     = safe_menu_get_color("delta_extract_col")
    cfg.extract_range   = safe_menu_get("delta_extract_range")
    -- Containers
    cfg.container       = safe_menu_get("delta_container_enabled")
    cfg.container_col   = safe_menu_get_color("delta_container_enabled")
    cfg.container_range = safe_menu_get("delta_container_range")
    cfg.contents        = safe_menu_get("delta_container_contents")
    -- Quests
    cfg.quest           = safe_menu_get("delta_quest_enabled")
    cfg.quest_col       = safe_menu_get_color("delta_quest_enabled")
    cfg.quest_range     = safe_menu_get("delta_quest_range")
    -- Items
    cfg.items           = safe_menu_get("delta_items_enabled")
    cfg.items_col       = safe_menu_get_color("delta_items_enabled")
    cfg.items_range     = safe_menu_get("delta_items_range")
    -- Corpses
    cfg.corpse          = safe_menu_get("delta_corpse_enabled")
    cfg.corpse_col      = safe_menu_get_color("delta_corpse_enabled")
    cfg.corpse_range    = safe_menu_get("delta_corpse_range")
    -- NPC ESP -- FIX: all NPC settings read from their own IDs, not player IDs
    cfg.npc             = safe_menu_get("delta_npc_enabled")
    cfg.npc_range       = safe_menu_get("delta_npc_range")
    cfg.npc_box         = safe_menu_get("delta_npc_box")
	cfg.npc_box_col 	= safe_menu_get_color("delta_npccol_box")
    cfg.npc_hp          = safe_menu_get("delta_npc_health")
    cfg.npc_name        = safe_menu_get("delta_npc_name")
    cfg.npc_name_col    = safe_menu_get_color("delta_npccol_name")
    cfg.npc_dist        = safe_menu_get("delta_npc_dist")             -- FIX: read npc dist independently
    cfg.npc_dist_col    = safe_menu_get_color("delta_npccol_dist")
    cfg.npc_skel        = safe_menu_get("delta_npc_skeleton")
    cfg.npc_skel_col    = safe_menu_get_color("delta_npccol_skeleton")
    -- Aimbot
    cfg.aim             = safe_menu_get("delta_npc_aim_enabled")
    cfg.aim_fov         = safe_menu_get("delta_npc_aim_fov")
    cfg.aim_smooth      = safe_menu_get("delta_npc_aim_smooth")
    cfg.aim_bone        = safe_menu_get("delta_npc_aim_bone")
    cfg.aim_prediction  = safe_menu_get("delta_npc_aim_prediction")
    cfg.aim_draw_fov    = safe_menu_get("delta_npc_aim_draw_fov")
    -- Claymores
    cfg.claymore        = safe_menu_get("delta_claymore_enabled")
    cfg.claymore_col    = safe_menu_get_color("delta_claymore_enabled")
    cfg.claymore_range  = safe_menu_get("delta_claymore_range")
    cfg.claymore_names  = safe_menu_get("delta_claymore_names")
	cfg.Debug			= safe_menu_get("delta_debug_enabled")
	-- Player Aimbot
	cfg.pl_aim = safe_menu_get("pl_aim_enabled")
	cfg.pl_aim_key = safe_menu_get_key("pl_aim_key")
	cfg.pl_aim_bone = safe_menu_get("pl_aim_bone")
	cfg.pl_aim_fov = safe_menu_get("pl_aim_fov")
	cfg.pl_aim_smooth = safe_menu_get("pl_aim_smooth")
	cfg.pl_aim_prediction = safe_menu_get("pl_aim_prediction")
	cfg.pl_aim_draw_fov = safe_menu_get("pl_aim_draw_fov")
	cfg.pl_aim_fov_color = safe_menu_get_color("pl_aim_fov_color")
	cfg.pl_aim_visible = safe_menu_get("pl_aim_visible_check")
end

local function draw_extracts()
    if not cfg.extract then return end
    local ws = game.workspace
    if not ws then return end
    local no_col = ws:find_first_child("NoCollision")
    local exits = no_col and no_col:find_first_child("ExitLocations")
    if not exits then return end
    local ok, children = pcall(function() return exits:get_children() end)
    if not ok or not children then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for _, part in ipairs(children) do
        if part:is_a("BasePart") then
            local pos = part.position
            if pos then
                local dist = get_dist(pos.x, pos.y, pos.z, cx, cy, cz)
                if dist <= cfg.extract_range / 0.28 then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis then
                        local dist_m = math.floor(dist * 0.28)
                        local txt = format("%s [%dm]", part.name, dist_m)
                        local tw = draw.get_text_size(txt, 14)
                        draw.text(sx - tw * 0.5, sy, txt, cfg.extract_col, 14)
                    end
                end
            end
        end
    end
end

local function draw_items()
    if not cfg.items then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for i=1,#world.items do
        local d=world.items[i]
        if d.part then
            local pos=d.part.position
            if pos then
                local dist=get_dist(pos.x,pos.y,pos.z,cx,cy,cz)
                if dist<=cfg.items_range/0.28 then
                    local sx,sy,vis=draw.world_to_screen(pos.x,pos.y,pos.z)
                    if vis then
                        local dist_m=math.floor(dist*0.28)
                        local txt=format("%s [%dm]",d.name,dist_m)
                        local tw=draw.get_text_size(txt,14)
                        draw.text(sx-tw*0.5,sy,txt,cfg.items_col,14)
                    end
                end
            end
        end
    end
end

local function draw_corpses()
    if not cfg.corpse then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for i = 1, #world.corpses do
        local d = world.corpses[i]
        if d.part then
            local pos = d.part.position
            if pos then
                local dist = get_dist(pos.x, pos.y, pos.z, cx, cy, cz)
                if dist <= cfg.corpse_range / 0.28 then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis then
                        local dist_m = math.floor(dist * 0.28)
                        local txt = format("Corpse: %s [%dm]", d.name, dist_m)
                        local tw = draw.get_text_size(txt, 14)
                        draw.text(sx - tw * 0.5, sy, txt, cfg.corpse_col, 14)
                    end
                end
            end
        end
    end
end

local function draw_containers()
    if not cfg.container or #world.containers==0 then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for i=1,#world.containers do
        local d=world.containers[i]
        if d.part then
            local pos=d.part.position
            if pos then
                local dist=get_dist(pos.x,pos.y,pos.z,cx,cy,cz)
                if dist<=cfg.container_range/0.28 then
                    local sx,sy,vis=draw.world_to_screen(pos.x,pos.y,pos.z)
                    if vis then
                        local dist_m=math.floor(dist*0.28)
                        local txt=format("%s [%dm]",d.name,dist_m)
                        local tw=draw.get_text_size(txt,14)
                        draw.text(sx-tw*0.5,sy,txt,cfg.container_col,14)
                        if cfg.contents and d.contents then
                            local ly=sy+14
                            for j=1,#d.contents do
                                local itxt=d.contents[j]
                                local itw=draw.get_text_size(itxt,11)
                                draw.text(sx-itw*0.5,ly,itxt,colors.gray,11)
                                ly=ly+12
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_quests()
    if not cfg.quest or #world.quests==0 then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for i=1,#world.quests do
        local d=world.quests[i]
        if d.part then
            local pos=d.part.position
            if pos then
                local dist=get_dist(pos.x,pos.y,pos.z,cx,cy,cz)
                if dist<=cfg.quest_range/0.28 then
                    local sx,sy,vis=draw.world_to_screen(pos.x,pos.y,pos.z)
                    if vis then
                        local dist_m=math.floor(dist*0.28)
                        local txt=format("%s [%dm]",d.name,dist_m)
                        local tw=draw.get_text_size(txt,14)
                        draw.text(sx-tw*0.5,sy,txt,cfg.quest_col,14)
                    end
                end
            end
        end
    end
end

local function get_bounds(parts)
    local min_x,min_y,max_x,max_y=10000,10000,-10000,-10000
    local valid=false
    for _,d in pairs(parts) do
        if d.part then
            local p=d.part.position
            if p then
                local px,py,pz=p.x,p.y,p.z
                for ox=-1,1,2 do for oy=-1,1,2 do for oz=-1,1,2 do
                    local sx,sy,vis=draw.world_to_screen(px+d.hx*ox,py+d.hy*oy,pz+d.hz*oz)
                    if vis then
                        valid=true
                        if sx<min_x then min_x=sx end if sx>max_x then max_x=sx end
                        if sy<min_y then min_y=sy end if sy>max_y then max_y=sy end
                    end
                end end end
            end
        end
    end
    return valid and {x=min_x,y=min_y,w=max_x-min_x,h=max_y-min_y} or nil
end

local function draw_npcs()
    if not cfg.npc or #world.npcs==0 then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    for i=1,#world.npcs do
        local n=world.npcs[i]
        if n.head and n.hum then
            local hpos=n.head.position
            if hpos then
                local hp=n.hum.health or 0
                local max_hp=n.hum.max_health or 100
                if hp>0 then
                    local dist=get_dist(hpos.x,hpos.y,hpos.z,cx,cy,cz)
                    if dist<=cfg.npc_range/0.28 then
                        local b=get_bounds(n.parts)
                        if b then
                            if cfg.npc_box then draw.box(b.x,b.y,b.w,b.h,cfg.npc_box_col,0) end
                            if cfg.npc_hp then draw.health_bar(b.x,b.y,b.h,hp,max_hp) end
                            local mid=b.x+b.w*0.5
                            if cfg.npc_name then
                                local tw=draw.get_text_size(n.name,14)
                                draw.text(mid-tw*0.5,b.y-16,n.name,cfg.npc_name_col,14)
                            end
                            if cfg.npc_dist then  -- FIX: uses cfg.npc_dist, not cfg.player_dist
                                local dist_m=math.floor(dist*0.28)
                                local txt=format("%dm",dist_m)
                                local tw=draw.get_text_size(txt,12)
                                draw.text(mid-tw*0.5,b.y+b.h+6,txt,cfg.npc_dist_col,12)
                            end
                            if cfg.npc_skel then
                                local screen={}
                                for name,pd in pairs(n.parts) do
                                    if pd.part then
                                        local pos=pd.part.position
                                        if pos then
                                            local bx,by,bv=draw.world_to_screen(pos.x,pos.y,pos.z)
                                            if bv then screen[name]={x=bx,y=by} end
                                        end
                                    end
                                end
                                for j=1,#skeleton_pairs do
                                    local pair=skeleton_pairs[j]
                                    local a,sb=screen[pair[1]],screen[pair[2]]
                                    if a and sb then draw.line(a.x,a.y,sb.x,sb.y,cfg.npc_skel_col,1.5) end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_claymores()
    if not cfg.claymore then return end
    local ws = game.workspace
    if not ws then return end
    local ai_zones = ws:find_first_child("AiZones")
    if not ai_zones then return end
    local claymores_folder = ai_zones:find_first_child("Claymores")
    if not claymores_folder then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    local cx = mpos and mpos.x or world.cam_x
    local cy = mpos and mpos.y or world.cam_y
    local cz = mpos and mpos.z or world.cam_z
    local ok, children = pcall(function() return claymores_folder:get_children() end)
    if not ok or not children then return end
    local red = { 1, 0.15, 0.15, 1 }
    for _, model in ipairs(children) do
        if model.name == "MON50" then
            local mine = model:find_first_child("MeshPart")
            local trig = model:find_first_child("Trigger")
            if mine then
                local minepos = mine.position
                if minepos then
                    local dist = get_dist(minepos.x, minepos.y, minepos.z, cx, cy, cz)
                    if dist <= cfg.claymore_range / 0.28 then
                        local sz = mine.size
                        local hx = sz and sz.x * 0.5 or 1
                        local hy = sz and sz.y * 0.5 or 1
                        local hz = sz and sz.z * 0.5 or 1
                        local min_x, min_y, max_x, max_y = 10000, 10000, -10000, -10000
                        local valid = false
                        for ox=-1,1,2 do for oy=-1,1,2 do for oz=-1,1,2 do
                            local sx, sy, vis = draw.world_to_screen(
                                minepos.x+hx*ox, minepos.y+hy*oy, minepos.z+hz*oz)
                            if vis then
                                valid = true
                                if sx < min_x then min_x = sx end if sx > max_x then max_x = sx end
                                if sy < min_y then min_y = sy end if sy > max_y then max_y = sy end
                            end
                        end end end
                        if valid then
                            local bw = max_x - min_x
                            local bh = max_y - min_y
                            draw.box(min_x, min_y, bw, bh, cfg.claymore_col, 1)
                            if cfg.claymore_names then
                                local dist_m = math.floor(dist * 0.28)
                                local txt = format("Claymore [%dm]", dist_m)
                                local tw = draw.get_text_size(txt, 13)
                                draw.text(min_x + (bw - tw) * 0.5, min_y - 16, txt, cfg.claymore_col, 13)
                            end
                        end
                        if trig then
                            local mx2, my2, mvis = draw.world_to_screen(minepos.x, minepos.y, minepos.z)
                            if mvis then
                                local tpos = trig.position
                                if tpos then
                                    local tx2, ty2, tvis = draw.world_to_screen(tpos.x, tpos.y, tpos.z)
                                    if tvis then draw.line(mx2, my2, tx2, ty2, red, 2) end
                                end
                                local ok2, trig_children = pcall(function() return trig:get_children() end)
                                if ok2 and trig_children then
                                    for _, tp in ipairs(trig_children) do
                                        if tp:is_a("BasePart") then
                                            local tppos = tp.position
                                            if tppos then
                                                local tx3, ty3, tv3 = draw.world_to_screen(tppos.x, tppos.y, tppos.z)
                                                if tv3 then draw.line(mx2, my2, tx3, ty3, red, 2) end
                                            end
                                        end
                                    end
                                end
                            end
                        end
                    end
                end
            end
        end
    end
end

local function draw_esp()
    if not menu.get("delta_player_enabled") then return end

    local lp = entity.GetLocalPlayer()
    if not lp then return end
    local lpPos = lp.Position
    if not lpPos then return end

    local players = entity.GetPlayers()
    local max_dist = cfg.player_range / 0.28 

    for _, p in ipairs(players) do
        if p.IsLocal or not p.IsAlive then goto continue end

        local pPos = p.Position
        if not pPos then goto continue end
        local dist = (pPos - lpPos).Magnitude
        if dist > max_dist then goto continue end

        if menu.get("delta_esp_enabled") then
            draw.ChamsPlayer(p, {0.2, 0.5, 1, 0.6}, {1, 0.1, 1, 0.6}, 2)
        end

        local b = p:GetBounds()
        if not b.valid then goto continue end

        if menu.get("delta_esp_enabled") then
            draw.HealthBar(b.x - 5, b.y, b.h, p.Health, p.MaxHealth)
        end

        if menu.get("delta_esp_enabled") then
            local tw, th = draw.GetTextSize(p.Name, 14)
            draw.Text(b.x + b.w / 2 - tw / 2, b.y - th - 2, p.Name, {1, 1, 1, 1})
        end

        if cfg.player_box then
            local color = cfg.box_color or {1, 0, 0, 1}
            if cfg.box_style == 0 then

                draw.Rect(b.x, b.y, b.w, b.h, color, 0, 1.5)
            else

                draw.CornerBox(b.x, b.y, b.w, b.h, color)
            end
        end

if cfg.player_skeleton then
    local bones = p:GetBonesScreen()
    for _, pair in ipairs(skeleton_pairs) do
        local a = bones[pair[1]]
        local b = bones[pair[2]]
        if a and b then
            draw.Line(a[1], a[2], b[1], b[2], cfg.player_skeleton_col, 1.5)
        end
    end
end

        ::continue::
    end
end


local function draw_players()
    if not cfg.player then return end
    local players = entity.get_players()
    if not players then return end
    local me = entity.get_local_player()
    local mpos = me and me.position
    if not mpos then return end
    for i = 1, #players do
        local p = players[i]
        if p and p.is_valid and p.is_alive and not p.is_local then
            local pos = p.position
            if pos then
                local dx = pos.x - mpos.x
                local dy = pos.y - mpos.y
                local dz = pos.z - mpos.z
                local dist_studs = math.sqrt(dx*dx + dy*dy + dz*dz)
                if dist_studs <= cfg.player_range / 0.28 then
                    local sx, sy, vis = draw.world_to_screen(pos.x, pos.y, pos.z)
                    if vis and cfg.player_dist then
                        local dist_m = math.floor(dist_studs * 0.28)
                        local txt = format("%dm", dist_m)
                        local tw = draw.get_text_size(txt, 14)
                        draw.text(sx - tw * 0.5, sy, txt, cfg.player_dist_col, 14)
                    end
                end
            end
        end
    end
end




local function run_aimbot()
    if not cfg.aim then return end
    local scx,scy=input.get_screen_center()
    if cfg.aim_draw_fov then
        draw.circle(scx,scy,cfg.aim_fov,{1,1,1,0.3},64)
        if cfg.aim_prediction then
            local tw=draw.get_text_size("[PREDICT]",12)
            draw.text(scx-tw*0.5,scy+cfg.aim_fov+5,"[PREDICT]",{0,1,1,1},12)
        end
    end
    local key=safe_menu_get_key("delta_npc_aimkey_enabled")
    if key == 0 then key = 0x01 end 
    if not input.is_key_down(key) then return end
    local cx,cy,cz=world.cam_x,world.cam_y,world.cam_z
    local best_fov=cfg.aim_fov
    local best_pos=nil
    local bone_name=aim_bones[cfg.aim_bone]
    for i=1,#world.npcs do
        local n=world.npcs[i]
        if n.hum and n.hum.health>0 then
            local pd=n.parts[bone_name]
            if pd and pd.part then
                local pos=pd.part.position
                if pos then
                    local dist_to_target=get_dist(pos.x,pos.y,pos.z,cx,cy,cz)
                    local aim_x,aim_y,aim_z=pos.x,pos.y,pos.z
                    if cfg.aim_prediction then
                        local vel=pd.part.velocity
                        if vel then
                            local pred_scale=dist_to_target/1000
                            aim_x=pos.x+vel.x*pred_scale
                            aim_y=pos.y+vel.y*pred_scale
                            aim_z=pos.z+vel.z*pred_scale
                        end
                    end
                    local sx,sy,vis=draw.world_to_screen(aim_x,aim_y,aim_z)
                    if vis then
                        local dx,dy=sx-scx,sy-scy
                        local fov=sqrt(dx*dx+dy*dy)
                        if fov<best_fov then best_fov=fov best_pos={x=aim_x,y=aim_y,z=aim_z} end
                    end
                end
            end
        end
    end
    if best_pos then camera.look_at(best_pos.x,best_pos.y,best_pos.z,cfg.aim_smooth) end
end

local function run_player_aimbot()
    if not cfg.pl_aim then return end

    local sw, sh = draw.GetScreenSize()
    local scx, scy = sw / 2, sh / 2

    if cfg.pl_aim_draw_fov then
        draw.Circle(scx, scy, cfg.pl_aim_fov, cfg.pl_aim_fov_color or {1,1,1,0.3}, 64)
        if cfg.pl_aim_prediction then
            local tw = draw.GetTextSize("[PREDICT]", 12)
            draw.Text(scx - tw / 2, scy + cfg.pl_aim_fov + 5, "[PREDICT]", {0,1,1,1}, 12)
        end
    end

    local key = cfg.pl_aim_key
    if key == 0 then key = 0x01 end
    if not input.IsKeyDown(key) then return end

    local me = entity.GetLocalPlayer()
    if not me then return end

    local camPos = camera.GetPosition()
    local bone_names = { [0] = "Head", [1] = "UpperTorso", [2] = "LowerTorso" }
    local bone_name = bone_names[cfg.pl_aim_bone] or "Head"

    local best_fov = cfg.pl_aim_fov
    local best_pos = nil

    local players = entity.GetPlayers()
    for _, p in ipairs(players) do
        if p.IsLocal or not p.IsAlive then goto continue end

        if cfg.pl_aim_visible then
            if not raycast.IsPlayerVisible(p.Character) then goto continue end
        end

        local char = p.Character
        if not char then goto continue end

        local bone = char:FindFirstChild(bone_name)
        if not bone or not bone:IsA("BasePart") then goto continue end

        local pos = bone.Position
        if not pos then goto continue end

        local target_pos = pos

        if cfg.pl_aim_prediction then
            local vel = bone.Velocity
            if vel then
                local dist = (pos - camPos).Magnitude
                local pred_scale = dist / 1000  
                target_pos = pos + vel * pred_scale
            end
        end

        local sx, sy, vis = draw.WorldToScreen(target_pos.X, target_pos.Y, target_pos.Z)
        if vis then
            local dx, dy = sx - scx, sy - scy
            local fov = math.sqrt(dx * dx + dy * dy)
            if fov < best_fov then
                best_fov = fov
                best_pos = target_pos
            end
        end

        ::continue::
    end

    if best_pos then
        camera.LookAt(best_pos.X, best_pos.Y, best_pos.Z, cfg.pl_aim_smooth)
    end
end



silent_prev_state = false
silent_hook_active = false

local function run_silent_aim()
    if not menu.get("silent_enabled") then
        if silent_hook_active then
            raycast.DisableSilentHook()
            silent_hook_active = false
        end
        return
    end

    if not silent_hook_active then
        local ok = raycast.EnableSilentHook()
        if ok then
            silent_hook_active = true
            print("[Silent Aim] Hook enabled")
        else
            print("[Silent Aim] Failed to enable hook")
            return
        end
    end

    local key = menu.get_key("silent_key") or 0x01
    if key == 0 then key = 0x01 end
    if not input.IsKeyDown(key) then
        raycast.ClearSilentTarget()
        return
    end

    local me = entity.GetLocalPlayer()
    if not me then return end

    local camPos = camera.GetPosition()
    if not camPos then return end

    local fov = menu.get("silent_fov") or 150
    local bone_names = { [0]="Head", [1]="UpperTorso", [2]="LowerTorso" }
    local bone_name = bone_names[menu.get("silent_bone") or 0] or "Head"
    local visible_check = menu.get("silent_visible_check") or false
    local prediction_ms = menu.get("silent_prediction") or 0

    local sw, sh = draw.GetScreenSize()
    local scx, scy = sw/2, sh/2

    if menu.get("silent_draw_fov") then
        local color = menu.get_color("silent_fov_color") or {1,1,1,0.3}
        draw.Circle(scx, scy, fov, color, 64)
    end

    local best_fov = fov
    local best_target = nil
    local best_origin = nil

    local players = entity.GetPlayers()
    for _, p in ipairs(players) do
        if p.IsLocal or not p.IsAlive then goto continue end

        if visible_check then
            if not raycast.IsPlayerVisible(p.Character) then goto continue end
        end

        local char = p.Character
        if not char then goto continue end

        local bone = char:FindFirstChild(bone_name)
        if not bone or not bone:IsA("BasePart") then goto continue end

        local pos = bone.Position
        if not pos then goto continue end

        local target_pos = pos
        if prediction_ms > 0 then
            local vel = bone.Velocity
            if vel then
                target_pos = pos + vel * (prediction_ms / 1000)
            end
        end

        local sx, sy, vis = draw.WorldToScreen(target_pos.X, target_pos.Y, target_pos.Z)
        if vis then
            local dx, dy = sx - scx, sy - scy
            local dist = math.sqrt(dx*dx + dy*dy)
            if dist < best_fov then
                best_fov = dist
                best_target = target_pos
                best_origin = camPos
            end
        end

        ::continue::
    end

    if best_target and best_origin then
        local dir = best_target - best_origin
        raycast.SetSilentTarget(best_origin, dir)
    else
        raycast.ClearSilentTarget()
    end
end

local MOD_NAMES = {
    ["Solter"]=true,["kupczakk"]=true,["nutsock112"]=true,["thecrafter103"]=true,
    ["forkycheck"]=true,["necanwx"]=true,["kbhgaming480"]=true,["lumivoxi"]=true,
    ["natan_113"]=true,["ianeyeballs"]=true,["imblu1"]=true,["gribler1"]=true,
    ["e2equl"]=true,["hhauoli"]=true,["farlan9"]=true,["zondering"]=true,
    ["strawberry_banana0"]=true,["exp_l0"]=true,["raechevel00"]=true,["yuckey47"]=true,
    ["shawnybear6"]=true,["deadlymagicaz"]=true,["jannu2010"]=true,["duhacek1112"]=true,
    ["sparky_deeny"]=true,["aidendenkule13"]=true,["hakaji"]=true,["c7asher"]=true,
    ["dominik7772"]=true,["atlas_thegamer"]=true,["accldentshappen"]=true,["libertadwar"]=true,
}

local detected_mods = {}

local function update_mod_check()
    local found = {}
    local players = entity.get_players()
    if players then
        for i = 1, #players do
            local p = players[i]
            if p and p.is_valid and p.name then
                local lower = p.name:lower()
                if MOD_NAMES[lower] then
                    found[#found + 1] = p.name
                end
            end
        end
    end
    detected_mods = found
end

local function draw_mod_alert()
    if not safe_menu_get("delta_mod_alert_enabled") then return end
    local mode = safe_menu_get("delta_mod_alert_mode") or 1
    local count = #detected_mods
    if mode == 1 and count == 0 then return end
    local sw, sh = utility.get_screen_size()
    local label = count > 0 and ("MODS IN SERVER: " .. count) or "NO MODS IN SERVER"
    local fs = 18
    local tw, th = draw.get_text_size(label, fs)
    local x = sw * 0.5 - tw * 0.5
    local y = 12
    local text_col   = count > 0 and {1, 0.08, 0.08, 1} or {0.3, 1, 0.3, 1}
    local border_col = count > 0 and {1, 0.1, 0.1, 0.9} or {0.2, 0.9, 0.2, 0.9}
    draw.rect_filled(x - 8, y - 4, tw + 16, th + 8, {0, 0, 0, 0.7}, 4)
    draw.rect(x - 8, y - 4, tw + 16, th + 8, border_col, 3, 1.5)
    draw.text(x, y, label, text_col, fs)
end


local slf_panel = { x = 20, y = 70, w = 280, dragging = false, drag_ox = 0, drag_oy = 0 }
local SLF_TITLE_H = 24
local SLF_PAD     = 8
local SLF_LINE_H  = 16
local SLF_FONT    = 13
local SLF_FONT_S  = 11

local slf_col_bg      = { 0.09, 0.02, 0.09, 0.96 }
local slf_col_title   = { 0.07, 0.07, 0.10, 1.00 }
local slf_col_border  = { 0.58,0.22,0.84,0.5 }
local slf_col_accent  = { 0.78,0.42,0.84,0.8}
local slf_col_dim     = { 0.25, 0.25, 0.30, 1.00 }
local slf_col_player  = { 1.00, 1.00, 1.00, 1.00 }
local slf_col_item    = { 0.35, 0.90, 0.45, 1.00 }
local slf_col_none    = { 0.45, 0.45, 0.50, 1.00 }
local slf_col_sep     = { 0.58, 0.22, 0.84, 0.5 }

local SLF_TRACKED_ITEMS = {
    "GPU", "Gold50g", "SPSh44", "FlareGun", "Mag556Rnd100",
    "SOCOM556", "Reapir", "HSPV", "TitanShield",
}

local slf_results = {}

local function slf_scan()
	thread_status.slf_scan.last_run = utility.GetTime()
    thread_status.slf_scan.count = thread_status.update_world.count + 1
    thread_status.slf_scan.status = "Run"
    local found = {}
    for _, d in ipairs(cached_players) do
        local p = d.entity
        if p and p.is_valid and p.is_alive and not p.is_local then
            local matched = {}
            for _, item in ipairs(d.inventory) do
                local item_lower = item:lower()
                for _, tracked in ipairs(SLF_TRACKED_ITEMS) do
                    if item_lower:find(tracked:lower(), 1, true) then
                        matched[#matched + 1] = item
                        break
                    end
                end
            end
            if #matched > 0 then
                found[#found + 1] = { name = d.name, items = matched }
            end
        end
    end
    slf_results = found
end

local function draw_slf_panel()
    if not safe_menu_get("delta_slf_enabled") then slf_panel.dragging = false return end
    local results = slf_results
    local num_results = #results
    local panel_h = SLF_TITLE_H + SLF_PAD
    if num_results == 0 then
        panel_h = panel_h + SLF_LINE_H + SLF_PAD
    else
        for _, entry in ipairs(results) do
            panel_h = panel_h + SLF_LINE_H
            panel_h = panel_h + (#entry.items * SLF_LINE_H)
            panel_h = panel_h + 6
        end
        panel_h = panel_h + SLF_PAD
    end
    local pw = slf_panel.w
    local mx, my = utility.get_mouse_pos()
    local lmb = input.is_key_down(0x01)
    if lmb then
        if not slf_panel.dragging then
            if mx >= slf_panel.x and mx <= slf_panel.x + pw and my >= slf_panel.y and my <= slf_panel.y + SLF_TITLE_H then
                slf_panel.dragging = true
                slf_panel.drag_ox = mx - slf_panel.x
                slf_panel.drag_oy = my - slf_panel.y
            end
        else
            slf_panel.x = mx - slf_panel.drag_ox
            slf_panel.y = my - slf_panel.drag_oy
        end
    else
        slf_panel.dragging = false
    end
    local px, py = slf_panel.x, slf_panel.y
    draw.rect(px-1, py-1, pw+2, panel_h+2, {0.58,0.22,0.84,0.5}, 5, 1)
    draw.rect_filled(px, py, pw, panel_h, slf_col_bg, 4)
    draw.rect_filled(px, py, pw, SLF_TITLE_H, slf_col_title, 4)
    draw.rect_filled(px, py, 2, SLF_TITLE_H, slf_col_border, 0)
    draw.line(px, py + SLF_TITLE_H, px + pw, py + SLF_TITLE_H, slf_col_border, 1)
    draw.rect(px, py, pw, panel_h, {0.58,0.22,0.84,0.5}, 4, 1)
    local title = "Server Loot Finder"
    if num_results > 0 then title = title .. "  [" .. num_results .. "]" end
    draw.text(px + SLF_PAD + 2, py + 5, title, slf_col_accent, SLF_FONT)
    local hint = "drag ::"
    local hw = draw.get_text_size(hint, 10)
    draw.text(px + pw - hw - SLF_PAD, py + 6, hint, slf_col_dim, 10)
    local ly = py + SLF_TITLE_H + SLF_PAD
    if num_results == 0 then
        draw.text(px + SLF_PAD, ly, "No players in server", slf_col_none, SLF_FONT)
        return
    end
    for i, entry in ipairs(results) do
        local name_txt = "> " .. entry.name
        draw.text(px + SLF_PAD, ly, name_txt, slf_col_player, SLF_FONT)
        ly = ly + SLF_LINE_H
        for _, item in ipairs(entry.items) do
            local item_txt = "  - " .. item
            draw.text(px + SLF_PAD + 4, ly, item_txt, slf_col_item, SLF_FONT_S)
            ly = ly + SLF_LINE_H
        end
        if i < num_results then
            draw.line(px + SLF_PAD, ly + 1, px + pw - SLF_PAD, ly + 1, slf_col_sep, 1)
        end
        ly = ly + 6
    end
end

thread.create(update_mod_check, 1000)
update_mod_check()




-- Debug 


local WIN_TITLE_H = 24
local WIN_PAD     = 8
local WIN_LINE_H  = 16
local WIN_FONT    = 13
local WIN_FONT_S  = 11

local WIN_COL_BG      = { 0.09, 0.02, 0.09, 0.96 }
local WIN_COL_TITLE   = { 0.07, 0.07, 0.10, 1.00 }
local WIN_COL_BORDER  = { 0.58, 0.22, 0.84, 0.5 }
local WIN_COL_ACCENT  = { 0.78, 0.42, 0.84, 0.8 }
local WIN_COL_DIM     = { 0.25, 0.25, 0.30, 1.00 }
local WIN_COL_PLAYER  = { 1.00, 1.00, 1.00, 1.00 }
local WIN_COL_ITEM    = { 0.35, 0.90, 0.45, 1.00 }
local WIN_COL_NONE    = { 0.45, 0.45, 0.50, 1.00 }
local WIN_COL_SEP     = { 0.58, 0.22, 0.84, 0.5 }


function drawStyledWindow(win)
    if not win.visible then
        win.dragging = false
        return
    end

    local content = win.content or {}
    local num_sections = #content


    local panel_h = WIN_TITLE_H + WIN_PAD
    if num_sections == 0 then
        panel_h = panel_h + WIN_LINE_H + WIN_PAD
    else
        for _, sec in ipairs(content) do
            if type(sec) == "table" and sec.name then
                panel_h = panel_h + WIN_LINE_H
                if sec.items then
                    panel_h = panel_h + (#sec.items * WIN_LINE_H)
                end
                panel_h = panel_h + 6
            else
                panel_h = panel_h + WIN_LINE_H
            end
        end
        panel_h = panel_h + WIN_PAD
    end

    local px, py = win.x, win.y
    local pw = win.w


    local mx, my = utility.GetMousePos()
    local lmb = input.IsKeyDown(0x01)
    if lmb then
        if not win.dragging then
            if mx >= px and mx <= px + pw and my >= py and my <= py + WIN_TITLE_H then
                win.dragging = true
                win.drag_ox = mx - px
                win.drag_oy = my - py
            end
        else
            win.x = mx - win.drag_ox
            win.y = my - win.drag_oy
            px, py = win.x, win.y
        end
    else
        win.dragging = false
    end

    draw.Rect(px-1, py-1, pw+2, panel_h+2, STYLE.border, 5, 1)
    draw.RectFilled(px, py, pw, panel_h, STYLE.bg_main, 4)
    draw.RectFilled(px, py, pw, WIN_TITLE_H, STYLE.bg_title, 4)
    draw.RectFilled(px, py, 2, WIN_TITLE_H, STYLE.border, 0)
    draw.Line(px, py + WIN_TITLE_H, px + pw, py + WIN_TITLE_H, STYLE.border, 1)
    draw.Rect(px, py, pw, panel_h, STYLE.border, 4, 1)

    local title = win.title or "Window"
    if num_sections > 0 then
        title = title .. "  [" .. num_sections .. "]"
    end
    draw.Text(px + WIN_PAD + 2, py + 5, title, STYLE.accent, WIN_FONT)

    local hint = "drag ::"
    local hw, _ = draw.GetTextSize(hint, 10)
    draw.Text(px + pw - hw - WIN_PAD, py + 6, hint, WIN_COL_DIM, 10)

    local ly = py + WIN_TITLE_H + WIN_PAD
    if num_sections == 0 then
        draw.Text(px + WIN_PAD, ly, "No content", WIN_COL_NONE, WIN_FONT)
        return
    end

    for i, sec in ipairs(content) do
        if type(sec) == "table" and sec.name then
            local name_txt = "> " .. sec.name
            draw.Text(px + WIN_PAD, ly, name_txt, WIN_COL_PLAYER, WIN_FONT)
            ly = ly + WIN_LINE_H
            if sec.items then
                for _, item in ipairs(sec.items) do
                    local item_txt = "  - " .. tostring(item)
                    draw.Text(px + WIN_PAD + 4, ly, item_txt, WIN_COL_ITEM, WIN_FONT_S)
                    ly = ly + WIN_LINE_H
                end
            end
        else
            draw.Text(px + WIN_PAD, ly, tostring(sec), WIN_COL_PLAYER, WIN_FONT)
            ly = ly + WIN_LINE_H
        end
        if i < num_sections then
            draw.Line(px + WIN_PAD, ly + 1, px + pw - WIN_PAD, ly + 1, WIN_COL_SEP, 1)
        end
        ly = ly + 6
    end
end


local debugWin = {
    x = 20,
    y = 132,
    w = 280,
    title = "Debug Info",
    visible = true,
    dragging = false,
    drag_ox = 0,
    drag_oy = 0,
    content = {} 
}



function draw_Debug()
    if not menu.get("delta_debug_enabled") then return end

    local lp = entity.GetLocalPlayer()
    local myPos = lp and lp.Position or Vector3.New(0, 0, 0)
    local fps = utility.GetFPS()
    local count = entity.GetPlayerCount()

    local content = {}

    table.insert(content, {
        name = "Player",
        items = {
            string.format("Pos X:%.0f Y:%.0f Z:%.0f", myPos.X, myPos.Y, myPos.Z),
            string.format("FPS: %.0f", fps),
            string.format("Players: %d", count)
        }
    })

if norecoil_active or norecoil_status ~= "OFF" then
    local strength = menu.get("norecoil_strength") or 100
    local base = 10
    local current_value = base * (1 - strength / 100)
    local items = {
        "Status: " .. norecoil_status,
        "Nodes: " .. norecoil_nodes,
        "Message: " .. norecoil_message,
        "Current Value: " .. string.format("%.1f", current_value)
    }
    table.insert(content, {
        name = "No Recoil",
        items = items
    })
else
    table.insert(content, {
        name = "No Recoil",
        items = { "Status: OFF" }
    })
end

    local placeId = game.PlaceId or 0
    local gameId = game.GameId or 0
    table.insert(content, {
        name = "Game",
        items = {
            string.format("PlaceID: %d", placeId),
            string.format("GameID: %d", gameId)
        }
    })

    local thread_items = {}
    local now = utility.GetTime()
    for name, data in pairs(thread_status) do
        local last = (now - data.last_run)
        local last_str = (data.last_run == 0) and "?" or string.format("%.1fs ago", last)
        local status_text = data.status .. " (" .. data.count .. " R, L: " .. last_str .. ")"
        table.insert(thread_items, name .. ": " .. status_text)
    end
    table.insert(content, {
        name = "Threads",
        items = thread_items
    })

    debugWin.content = content
    drawStyledWindow(debugWin)
end



function drawCrosshair()
    if not menu.get("crosshair_enabled") then return end

    local sw, sh = draw.GetScreenSize()
    local cx, cy = sw / 2, sh / 2

    local style = menu.get("crosshair_style") or 0
    local dot_size = menu.get("crosshair_dot_size") or 4
    local line_len = menu.get("crosshair_line_len") or 10
    local line_gap = menu.get("crosshair_line_gap") or 4
    local line_thick = menu.get("crosshair_line_thick") or 2
    local color = menu.get_color("crosshair_color") or {0, 1, 0, 1}

    draw.CircleFilled(cx, cy, dot_size, color, 16)

    if style == 0 then
        return
    end

    local function drawLine(dx, dy, len, thick)
        local x1 = cx + dx
        local y1 = cy + dy
        local x2 = cx + dx + (len * (dx == 0 and 0 or (dx > 0 and 1 or -1)) or 0)
        local y2 = cy + dy + (len * (dy == 0 and 0 or (dy > 0 and 1 or -1)) or 0)
        draw.Line(x1, y1, x2, y2, color, thick)
    end

    local function drawDirection(dx, dy)
        local x1 = cx + dx
        local y1 = cy + dy
        local x2 = cx + dx + (dx == 0 and 0 or (dx > 0 and line_len or -line_len))
        local y2 = cy + dy + (dy == 0 and 0 or (dy > 0 and line_len or -line_len))
        draw.Line(x1, y1, x2, y2, color, line_thick)
    end

    local offset = dot_size + line_gap

    if style == 1 then
        drawDirection(0, -offset)
        drawDirection(0, offset)
        drawDirection(-offset, 0)
        drawDirection(offset, 0)
    elseif style == 2 then
        drawDirection(0, offset)
        drawDirection(-offset, 0)
        drawDirection(offset, 0)
    end
end

function on_frame()
    cache_settings()
    update_camera()
    draw_extracts()
    draw_items()
    draw_corpses()
    draw_containers()
    draw_quests()
    draw_npcs()
    draw_claymores()
    draw_players()
    draw_esp()
    draw_inv_panel()
    draw_boss_panel()
    run_aimbot()
    draw_mod_alert()
    draw_slf_panel()
	draw_Debug()
	draw_Notifications()
	drawRadarWindow()
	drawCrosshair()
	run_player_aimbot()
	run_silent_aim()
	drawConsoleWindow()
end

thread.create(update_world, 2000)
thread.create(update_npcs, 500)
thread.create(scan_players_inv, 100)
thread.create(slf_scan, 500)
thread.create(update_bosses, 500)

thread_status.update_world.status = "Started"
thread_status.update_npcs.status = "Started"
thread_status.scan_players_inv.status = "Started"
thread_status.slf_scan.status = "Started"
thread_status.update_bosses.status = "Started"

update_camera()
scan_players_inv()
slf_scan()
update_world()
update_npcs()
update_bosses()
load_config()





--addNotification("PDH - Project Delta v.1.5.1 executed", "success", 5)
--addConsoleMessage("[MAIN] PDH - Project Delta v1.5.1 loaded", {0.60, 0.50, 1, 1})
