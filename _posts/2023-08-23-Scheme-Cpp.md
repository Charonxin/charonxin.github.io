---
title: "How to interact with scheme using cpp code"
categories:
  - blog
tags:
  - scheme
  - lua
  - xmake
---

# 如何使用lua实现cpp与scheme逻辑交互

------------------

需求：scheme内调用cpp函数，返回Lolly版本。

## 梳理一下texmacs使用glue调用cpp代码的逻辑

先看`build_glue.lua`

```lua
local function translate_name(name)
    if name:endswith("*") then
        name = name:sub(1,-2) .. "_dot"
    end
    name = name:gsub("?", "P")
    name = name:gsub("!", "S")
    name = name:gsub("<", "F")
    name = name:gsub(">", "2")
    name = name:gsub("-", "_")
    name = name:gsub("=", "Q")
    name = "tmg_" .. name
    return name;
end

function main(glue, glue_name)
    local func_list = ""
    local reg_list = ""
    for _,glue_item in ipairs(glue.glues) do
        local scm_name = glue_item.scm_name
        local trans_name = translate_name(scm_name)
        local arg_count = 0;
        local arg_list = ""
        local in_list = ""
        local extraction = ""
        local assertion = ""

        if glue_item.arg_list ~= nil then
            arg_count = #glue_item.arg_list

            local iter,stat,init_pos = ipairs(glue_item.arg_list);
            local first_pos,first_arg = iter(stat,init_pos)
            if first_pos ~= nil then
                arg_list = "tmscm arg1"
                in_list = "in1"
                
                for pos,arg in iter,stat,first_pos do
                    arg_list = arg_list .. ", tmscm arg" .. pos
                    in_list = in_list .. ", in" .. pos
                end
            end

            for pos,arg in ipairs(glue_item.arg_list) do
                assertion = assertion .. "  TMSCM_ASSERT_" .. string.upper(arg) .. " (arg" .. pos .. ", TMSCM_ARG" .. pos .. ", \"" .. scm_name .. "\");\n"
                extraction = extraction .. "  " .. arg .. " in" .. pos .. "= " .. "tmscm_to_" .. arg .. " (arg" .. pos .. ");\n"
            end

            assertion = assertion .. "\n"
            extraction = extraction .. "\n"
        end

        local returned = "TMSCM_UNSPECIFIED"
        local invoke = glue.binding_object .. glue_item.cpp_name .. " (" .. in_list .. ");\n"
        if glue_item.ret_type ~= "void" then
            invoke = glue_item.ret_type .. " out= " .. invoke
            returned = glue_item.ret_type .. "_to_tmscm (out)"
        end

        func_list = func_list .. [[
tmscm
]] .. trans_name .. " (" .. arg_list .. [[) {
]] .. assertion  
.. extraction .. [[
  // TMSCM_DEFER_INTS;
  ]] .. invoke .. [[
  // TMSCM_ALLOW_INTS;

  return ]] .. returned ..[[;
}

]]
        reg_list = reg_list .. [[  tmscm_install_procedure ("]] .. scm_name .. [[",  ]] .. trans_name .. ", " .. arg_count .. [[, 0, 0);
]]
    end
    local res = [[

/******************************************************************************
*
* This file has been generated automatically using build-glue.scm
* from ]] .. glue_name .. [[.lua. Please do not edit its contents.
* Copyright (C) 2000 Joris van der Hoeven
*
* This software falls under the GNU general public license version 3 or later.
* It comes WITHOUT ANY WARRANTY WHATSOEVER. For details, see the file LICENSE
* in the root directory or <http://www.gnu.org/licenses/gpl-3.0.html>.
*
******************************************************************************/

]] .. func_list ..
[[void
]] .. glue.initializer_name .. [[ () {
]] .. reg_list
.. [[}
]]
    return res
end
```

main函数内部对每一个glue，先拿到它的`scm_name`，然后根据不同类型生成scheme中用到的`translate_name`，然后就是通过某种手段获取参数，解析参数，，解析声明，，定义，，调用最后生成对应的cpp文件。。传进去的这个glue是什么呢？？

比方说这个L3 kernel目录下的`string_glue.lua`

```lua
function main()
    return {
        binding_object = "",
        initializer_name = "initialize_glue_string",
        glues = {
            {
                scm_name = "cpp-string-number?",
                cpp_name = "is_double",
                ret_type = "bool",
                arg_list = {
                    "string"
                }
            },
            {
                scm_name = "encode-base64",
                cpp_name = "encode_base64",
                ret_type = "string",
                arg_list = {
                    "string"
                }
            },
            {
                scm_name = "decode-base64",
                cpp_name = "decode_base64",
                ret_type = "string",
                arg_list = {
                    "string"
                }
            },
        }
    }
end
```

非常简洁，比刚刚的生成cpp代码的lua脚本简洁不少。。这里的string_glue其实就是上文的一个glue，然后里面的glues是对应的列表，用来做**函数和参数**绑定。。这个`initializer_name`是啥呢？这个玩意是`initialize_glue_l3.cpp`内的一个函数名：`initialize_glue_string ();`所以**这个lua脚本其实完成了scm函数和cpp函数的绑定**。

这个函数声明有了，那么定义在哪呢？函数具体实现就放到了上文`build_glue.lua`生成的文件内，在`void initialize_glue_l3 `函数上面把生成的文件名通过宏引用的方式引入，就可以绕过编译器的`defined but not declared`报错。

## 梳理一下我们的需求

我们需要在scheme中输入类似`lolly-version`一行命令给我们返回当前正在用的Lolly的版本，参考`PDFHUMMUS_VERSION`宏

在`package.lua`的`function add_requires_of_mogan()`内部，先定义一个局部变量，然后走xmake的编译前预处理`set_configvar`接口，添加一些编译前需要预处理的模板配置变量。

```lua
local PDFHUMMUS_VERSION = "4.5.10"
set_configvar("PDFHUMMUS_VERSION", PDFHUMMUS_VERSION)
```

可以仿照写Lolly version的宏：

```lua
local LOLLY_VERSION = "1.1.3"
set_configvar("LOLLY_VERSION", LOLLY_VERSION)
```

**然后看`xmake.lua`文件** L325

```lua
add_configfiles(
    "src/System/tm_configure.hpp.xmake", {
        filename = "tm_configure.hpp",
        pattern = "@(.-)@",
        variables = {
            XMACS_VERSION = XMACS_VERSION,
            CONFIG_USER = CONFIG_USER,
            CONFIG_STD_SETENV = "#define STD_SETENV",
            tm_devel = "Texmacs-" .. DEVEL_VERSION,
            tm_devel_release = "Texmacs-" .. DEVEL_VERSION .. "-" .. DEVEL_RELEASE,
            tm_stable = "Texmacs-" .. STABLE_VERSION,
            tm_stable_release = "Texmacs-" .. STABLE_VERSION .. "-" .. STABLE_RELEASE,
            }})
```

大概是用`add_configfiles`对`tm_configure.hpp.xmake`进行编译前预处理，但是写`LOLLY_VERSION`宏又有点不一样，因为打算把这个写在L3的kernel里面，而不是Plugins里面。

所以找到与L3有关的配置文件：libkernel_l3 和 libmogan 两个target用到了"src/Scheme/L3"

所以`src/System/config_l3.h.xmake`和`src/System/tm_configure.hpp.xmake`里面都需要添加

```
#define LOLLY_VERSION "@LOLLY_VERSION@"
```

这里的@字符对应`pattern = "@(.-)@",`



## Ref

- https://xmake.io/#/manual/project_target?id=set-template-configuration-variables
- https://xmake.io/#/manual/project_target?id=add-template-configuration-files

