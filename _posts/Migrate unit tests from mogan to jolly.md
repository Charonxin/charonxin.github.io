# Migrate unit tests from mogan to jolly
## Description
- [ ] string_test
- [x] analyze_test
- [x] tree_test

## Troubleshootings
### There may be naming conflicts between "enum" and "typedef".

**Problem**: There is a naming conflict between the enumeration types defined in the "wingdi.h" and "tree_label" files, which leads to redefinition errors.

```
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:981:20: error: 'typedef LOGBRUSH PATTERN' redeclared as different kind of symbol
   typedef LOGBRUSH PATTERN;
                    ^~~~~~~
In file included from Kernel\Abstractions/observer.hpp:16,
                 from Kernel\Types/tree.hpp:15,
                 from Kernel\Containers/list.hpp:14,
                 from Kernel\Containers/hashset.hpp:14,
                 from Kernel\Types/analyze.hpp:15,
                 from tests\Kernel\Types\analyze_test.cpp:3:
Kernel\Types/tree_label.hpp:293:3: note: previous declaration 'tree_label PATTERN'
   PATTERN,
   ^~~~~~~
In file included from C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/windows.h:71,
                 from D:\a\jolly\.xmake-global\.xmake\packages\d\doctest\2.4.11\1a60e4998b874d2e8f9168beada972b2\include/doctest/doctest.h:3214,
                 from tests\Kernel\Types\analyze_test.cpp:4:
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:982:11: error: 'PATTERN' does not name a type
   typedef PATTERN *PPATTERN;
           ^~~~~~~
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:983:11: error: 'PATTERN' does not name a type
   typedef PATTERN *NPPATTERN;
           ^~~~~~~
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:984:11: error: 'PATTERN' does not name a type
   typedef PATTERN *LPPATTERN;
           ^~~~~~~
```

**Solution:** Use enumeration classes, which implicitly limit the scope of enumeration values to the name of the enumeration.

For example, to define the following enumeration structure:

```
enum status{
  status_ok,
  status_error
};
```

   You can declare it inside a namespace:

```
namespace status{
       enum status{
         ok,
          error
  };
}
```

**Ref:**

- [jolly](https://github.com/LiiiLabs/jolly/pull/13)
- [avoid using `using namespace std`](https://www.zhihu.com/question/26911239/answer/1259552756)
- [folly](https://github.com/facebook/folly/blob/03fbccdc93e90a135d074f87e1433b4dcff988f0/folly/io/Cursor.h#L52)

### Using `using namespace jolly` in a .hpp file can lead to namespace pollution and potential naming conflicts.

**Problem**: When a header file contains numerous function and variable declarations and definitions, using `using namespace` may create naming conflicts with other libraries or code, potentially resulting in compilation errors or runtime errors.

```
// Kernel/Abstractions/observer.hpp
#include "string.hpp"
#include "tree_label.hpp"

using namespace jolly;

class tree;
class hard_link_rep;
class observer;
```

**Solution**: Be specific with the structures within the namespace to avoid naming conflicts.

```
// Kernel/Abstractions/observer.hpp
#include "string.hpp"
#include "tree_label.hpp"

using jolly::tree_label;

class tree;
class hard_link_rep;
```

### During mingw cross-compilation, `std::mutex` type is undefined.

**Problem**: During mingw cross-compilation, the use of different mingw compilers with different suffixes can cause issues, such as the inability of a mingw compiler with the "-win32" suffix to compile `std::mutex` in `doctest.h`.

```
error: @programdir/modules/private/async/runjobs.lua:256: @programdir/modules/private/action/build/object.lua:91: @programdir/modules/core/tools/gcc.lua:752: In file included from tests/Kernel/Types/string_test.cpp:5:
/home/runner/work/jolly/.xmake-global/.xmake/packages/d/doctest/2.4.11/1a60e4998b874d2e8f9168beada972b2/include/doctest/doctest.h:5418:9: error: ‘mutex’ in namespace ‘std’ does not name a type
 5418 |         DOCTEST_DECLARE_MUTEX(mutex)
      |         ^~~~~~~~~~~~~~~~~~~~~
In file included from tests/Kernel/Types/string_test.cpp:5:
/home/runner/work/jolly/.xmake-global/.xmake/packages/d/doctest/2.4.11/1a60e4998b874d2e8f9168beada972b2/include/doctest/doctest.h:3217:1: note: ‘std::mutex’ is defined in header ‘<mutex>’; did you forget to ‘#include <mutex>’?
 3216 | #include <io.h>
  +++ |+#include <mutex>
 3217 | 
```

**Solution**: Use a mingw compiler with the "-posix" suffix to resolve this issue.

```
sudo update-alternatives --set x86_64-w64-mingw32-g++ /usr/bin/x86_64-w64-mingw32-g++-posix
```

**Ref:**

- [Cross-Compiling with mingw: "mingw mutex is not a member of std"](https://thomas.trocha.com/blog/cross-compiling-with-mingw---mingw-mutex-is-not-a-member-of-std-/)
- [pr](https://github.com/LiiiLabs/jolly/pull/6)

------

## 问题与解决

### enum和typedef的命名冲突

**问题**：wingdi.h文件和tree_label文件中的枚举类型发生冲突，导致重定义。

```
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:981:20: error: 'typedef LOGBRUSH PATTERN' redeclared as different kind of symbol
   typedef LOGBRUSH PATTERN;
                    ^~~~~~~
In file included from Kernel\Abstractions/observer.hpp:16,
                 from Kernel\Types/tree.hpp:15,
                 from Kernel\Containers/list.hpp:14,
                 from Kernel\Containers/hashset.hpp:14,
                 from Kernel\Types/analyze.hpp:15,
                 from tests\Kernel\Types\analyze_test.cpp:3:
Kernel\Types/tree_label.hpp:293:3: note: previous declaration 'tree_label PATTERN'
   PATTERN,
   ^~~~~~~
In file included from C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/windows.h:71,
                 from D:\a\jolly\.xmake-global\.xmake\packages\d\doctest\2.4.11\1a60e4998b874d2e8f9168beada972b2\include/doctest/doctest.h:3214,
                 from tests\Kernel\Types\analyze_test.cpp:4:
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:982:11: error: 'PATTERN' does not name a type
   typedef PATTERN *PPATTERN;
           ^~~~~~~
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:983:11: error: 'PATTERN' does not name a type
   typedef PATTERN *NPPATTERN;
           ^~~~~~~
C:/ProgramData/chocolatey/lib/mingw/tools/install/mingw64/x86_64-w64-mingw32/include/wingdi.h:984:11: error: 'PATTERN' does not name a type
   typedef PATTERN *LPPATTERN;
           ^~~~~~~
```

**解决**：使用枚举类，隐式地在枚举名称内限定枚举值的范围。

比如要定义如下枚举结构：

```
enum status{
  status_ok,
  status_error
};
```

可以将其声明在命名空间内：

```
namespace status{
       enum status{
         ok,
          error
  };
}
```

**参考**：

- [jolly](https://github.com/LiiiLabs/jolly/pull/13)
- [尽量不要使用using namespace std](https://www.zhihu.com/question/26911239/answer/1259552756)
- [folly](https://github.com/facebook/folly/blob/03fbccdc93e90a135d074f87e1433b4dcff988f0/folly/io/Cursor.h#L52)

### .hpp文件中使用了`using namespace jolly`，会导致污染

**问题**：头文件中包含了很多函数和变量的声明和定义，如果使用了`using namespace`，可能会与其他库或代码中的名称冲突，导致编译错误或者运行时错误。

```
// Kernel/Abstractions/observer.hpp
#include "string.hpp"
#include "tree_label.hpp"

using namespace jolly;

class tree;
class hard_link_rep;
class observer;
```

**解决**：具体到命名空间内具体的结构。

```
// Kernel/Abstractions/observer.hpp
#include "string.hpp"
#include "tree_label.hpp"

using jolly::tree_label;

class tree;
class hard_link_rep;
```

### mingw交叉编译报错`‘mutex’ in namespace ‘std’ does not name a type`

**问题**：在使用mingw交叉编译时，由于不同的mingw编译器采用不同的后缀，带有-win32的mingw编译器似乎不能编译doctest.h中的std::mutex。

```
error: @programdir/modules/private/async/runjobs.lua:256: @programdir/modules/private/action/build/object.lua:91: @programdir/modules/core/tools/gcc.lua:752: In file included from tests/Kernel/Types/string_test.cpp:5:
/home/runner/work/jolly/.xmake-global/.xmake/packages/d/doctest/2.4.11/1a60e4998b874d2e8f9168beada972b2/include/doctest/doctest.h:5418:9: error: ‘mutex’ in namespace ‘std’ does not name a type
 5418 |         DOCTEST_DECLARE_MUTEX(mutex)
      |         ^~~~~~~~~~~~~~~~~~~~~
In file included from tests/Kernel/Types/string_test.cpp:5:
/home/runner/work/jolly/.xmake-global/.xmake/packages/d/doctest/2.4.11/1a60e4998b874d2e8f9168beada972b2/include/doctest/doctest.h:3217:1: note: ‘std::mutex’ is defined in header ‘<mutex>’; did you forget to ‘#include <mutex>’?
 3216 | #include <io.h>
  +++ |+#include <mutex>
 3217 | 
```

**解决**：采用带有-posix后缀的mingw编译器。

```
sudo update-alternatives --set x86_64-w64-mingw32-g++ /usr/bin/x86_64-w64-mingw32-g++-posix
```

**参考**：

- [Cross-Compiling with mingw: "mingw mutex is not a member of std"](https://thomas.trocha.com/blog/cross-compiling-with-mingw---mingw-mutex-is-not-a-member-of-std-/)
- [pr](https://github.com/LiiiLabs/jolly/pull/6)

