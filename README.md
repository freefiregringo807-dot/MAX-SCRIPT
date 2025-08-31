--carregar biblioteca
local Fluent = loadstring(game:HttpGet("https://github.com/dawid-scripts/Fluent/releases/latest/download/main.lua"))()

local Window = Fluent:CreateWindow({
    Title = "Max Script" .. Fluent.Version,
    TabWidth = 160, Size = UDim2.fromOffset(580, 460), Theme = "Dark"
})

local Tabs = {
    Main = Window:AddTab({ Title = "Max" }),
    Settings = Window:AddTab({ Title = "Settings", Icon = "settings" })
}
--parágrafos
Tabs.Main:AddParagraph({ Title = "Max", Content = "Max Script" })

--botões
Tabs.Main:AddButton({ Title = "fly", Callback = function() 
 loadstring(game:HttpGet("https://github.com/freefiregringo807-dot/MAX-SCRIPT.git"))()
 end })
