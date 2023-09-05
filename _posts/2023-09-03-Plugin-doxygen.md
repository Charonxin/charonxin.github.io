---
title: "Analyze the implement of doxygen Plugin in xmake"
categories:
  - blog
tags:
  - xmake
  - lua
---

# Analyze the implement of doxygen Plugin in xmake

---------

## 1 问题复现

在大型项目中，注释和文档都是不可或缺的，为了提高效率，我们在lolly基础库中使用doxygen来进行文档生成。

我们希望对每个正式发布的版本\<tag>都有对应的文档能够生成并发布到github静态页面。

bug如下

```
generating ..🍺
/home/runner/work/lolly/xmake-global/.xmake/packages/d/doxygen/1.9.6/c2fa104f6aa247f594ffe5e82a9ade16/bin/doxygen /home/runner/work/lolly/lolly/doxyfile
Doxygen version used: 1.9.6
Searching for include files...
Searching for example files...
Searching for images...
Searching for dot files...
Searching for msc files...
Searching for dia files...
Searching for files to exclude
Searching INPUT for files to process...
Searching for files in directory /home/runner/work/lolly/lolly/build
Searching for files in directory /home/runner/work/lolly/lolly/build/L1
Reading and parsing tag files
Parsing files
Preprocessing /home/runner/work/lolly/lolly/build/L1/config.h...
Parsing file /home/runner/work/lolly/lolly/build/L1/config.h...
```

可以看到并没有扫描所有需要生成文档的目录。

## 2 问题排查

原因是我们在doxygen文档生成时使用了这样的命令：

```yml
    - name: BUILD
      run: xmake doxygen -y -v build
```

正确的方法应该是：

```bash
xmake doxygen -y -v
```

遵循usage：

```bash
Usage: $xmake [task] [options] [target]
```

## 3 原理探索

`doxygen`在xmake中是用插件的形式实现的，在使用`xmake doxygen`前应该先安装：

```bash
xrepo install doxygen -y
```

下面我们来分析一下这个插件的实现原理：

xmake.lua文件：

```lua
-- define task
task("doxygen")

    -- set category
    set_category("plugin")

    -- on run
    on_run("main")

    -- set menu
    set_menu {
                -- usage
                usage = "xmake doxygen [options] [arguments]"

                -- description
            ,   description = "Generate the doxygen document."

                -- options
            ,   options =
                {
                    {'o', "outputdir",  "kv", nil,      "Set the output directory."         }
                ,   {}
                ,   {nil, "srcdir",     "v",  "src",    "Set the source code directory."    }
                }
            }
```

可以看到在脚本域调用了main函数，我们接下来分析一下main.lua脚本：

```lua
-- imports
import("core.base.option")
import("core.project.config")
import("core.project.project")
import("lib.detect.find_tool")
import("private.action.require.impl.packagenv")
import("private.action.require.impl.install_packages")

-- generate doxyfile
function _generate_doxyfile(doxygen)

    -- generate the default doxyfile
    local doxyfile = path.join(project.directory(), "doxyfile")
    os.vrunv(doxygen.program, {"-g", doxyfile})

    -- enable recursive
    --
    -- RECURSIVE = YES
    --
    io.gsub(doxyfile, "RECURSIVE%s-=%s-NO", "RECURSIVE = YES")

    -- set the source directory
    --
    -- INPUT = xxx
    --
    local srcdir = option.get("srcdir")
    if srcdir and os.isdir(srcdir) then
        io.gsub(doxyfile, "INPUT%s-=.-\n", format("INPUT = %s\n", srcdir))
    end

    -- set the output directory
    --
    -- OUTPUT_DIRECTORY =
    --
    local outputdir = option.get("outputdir") or config.buildir()
    if outputdir then
        io.gsub(doxyfile, "OUTPUT_DIRECTORY%s-=.-\n", format("OUTPUT_DIRECTORY = %s\n", outputdir))
        os.mkdir(outputdir)
    end

    -- set the project name
    --
    -- PROJECT_NAME =
    --
    local name = project.name()
    if name then
        io.gsub(doxyfile, "PROJECT_NAME%s-=.-\n", format("PROJECT_NAME = %s\n", name))
    end
    return doxyfile
end

function main()

    -- load configuration
    config.load()

    -- enter the environments of doxygen
    local oldenvs = packagenv.enter("doxygen")

    -- find doxygen
    local packages = {}
    local doxygen = find_tool("doxygen")
    if not doxygen then
        table.join2(packages, install_packages("doxygen"))
    end

    -- enter the environments of installed packages
    for _, instance in ipairs(packages) do
        instance:envs_enter()
    end

    -- we need to force detect and flush detect cache after loading all environments
    if not doxygen then
        doxygen = find_tool("doxygen", {force = true})
    end
    assert(doxygen, "doxygen not found!")

    -- get doxyfile first
    local doxyfile = "doxyfile"
    if not os.isfile(doxyfile) then
        doxyfile = _generate_doxyfile(doxygen)
    end
    assert(os.isfile(doxyfile), "%s not found!", doxyfile)

    -- set the project version
    --
    -- PROJECT_NUMBER =
    --
    local version = project.version()
    if version then
        io.gsub(doxyfile, "PROJECT_NUMBER%s-=.-\n", format("PROJECT_NUMBER = %s\n", version))
    end

    -- generate document
    cprint("generating ..${beer}")
    os.vrunv(doxygen.program, {doxyfile}, {curdir = project.directory()})

    -- done
    cprint("${bright green}result: ${default green}%s/html/index.html", outputdir)
    cprint("${color.success}doxygen ok!")
    os.setenvs(oldenvs)
end
```

先看`_generate_doxyfile`函数：

1. 使用`doxygen -g doxyfile`生成对应的doxyfile文件
2. 分别将“RECURSIVE%s-=%s-NO”、“INPUT%s-=.-\n”和“PROJECT_NAME%s-=.-\n”替换为option中传参。

然后在option里解析相关的config参数。

**但是`xmake doxygen`**并未提供所有的参数，仅仅提供了output dir。。。
