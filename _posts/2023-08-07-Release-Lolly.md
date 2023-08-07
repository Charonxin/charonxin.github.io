---
title: "Release Lolly"
categories:
  - blog
tags:
  - lolly
  - xmake-xrepo
---

#  记录一下Lolly库发布和xmake-xrepo制作
-----------

链接：[#2410](https://github.com/xmake-io/xmake-repo/pull/2410)

## 背景

因为mogan编辑器基于的Texmacs源码实在太过庞大复杂，所以决定拆下一部分基础代码构建Lolly库方便维护，另外也方便升级与维护，另外mogan采用xmake构建，用**xrepo**作为包管理工具，所以需要将Lolly打包推到xrepo上游。



## 开始动手

```lua
package("lolly")
--- 设置主页地址和包描述
    set_homepage("https://github.com/XmacsLabs/lolly")
    set_description("Lolly is an alternative to the C++ Standard Library.")
--- 设置仓库地址、版本号和哈希校验码
    add_urls("https://github.com/XmacsLabs/lolly.git")
    add_urls("https://gitee.com/XmacsLabs/lolly.git")

    add_versions("v1.0.1", "69ebde6df3e5b4b9473f018d105f48f4abb179ff")

    add_configs("nowide_standalone", {description = "nowide", default = true, type = "boolean"})

    on_load(function (package)
        if package:is_plat("mingw", "windows") and package:config("nowide_standalone") then
            package:add("deps", "nowide_standalone")
        end
    end)

    on_install("linux", "macosx", "mingw", "wasm", function (package)
        local configs = {}
        if package:config("shared") then
            configs.kind = "shared"
        end
        import("package.tools.xmake").install(package, configs)
    end)

    on_test(function (package)
        assert(package:check_cxxsnippets({test = [[
            #include "string.hpp"
            void test() {
                string s("hello");
            }
        ]]}, {configs = {languages = "c++11"}}))
    end)
```



### 坑点

- lolly库中实现了tm_ostream类其中由于需要支持unicode编码引入了nowide库，虽然只有4行……，但是暂时在mingw和windows平台还没有很好的替代方案，所以需要在mingw和windows平台引入nowide_standalone包。

```c++
// lolly库中部分代码如下
#if (defined OS_MINGW || defined OS_WIN32)
  if (file == fstdout) {
    nowide::cout << s;
    nowide::cout.flush ();
  }
  else if (file == fstderr) {
    nowide::cerr << s;
    nowide::cerr.flush ();
  }
  else
#endif
```

```lua
--- 引入nowide包的代码片段
	add_configs("nowide_standalone", {description = "nowide", default = true, type = "boolean"})

    on_load(function (package)
        if package:is_plat("mingw", "windows") and package:config("nowide_standalone") then
            package:add("deps", "nowide_standalone")
        end
    end)
```

当然在linux平台不需要nowide时，可以通过命令行执行指令，就可以少下一个包。

```bash
xmake config nowide_standalone=n
```

此外，代码规范方面，区分脚本域和描述域。。。
