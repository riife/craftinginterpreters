## 写在前面
本项目是对 《[Crafting Interpreters][en reading online]》 一文的翻译。[📖在线阅读][zh reading online]

这本书在 Github 上已有完整翻译 [craftinginterpreters_zh][]，但是排版和原文有较大差异，原文的阅读体验更好。故本项目做了一次搬运，将两个项目结合，在不破坏原文排版的基础上进行了搬运和查漏补缺。
相比与[原项目][crafting interpreters github]，改动的内容主要有:
- 对原书的翻译，翻译后的文本保存在 `/book_zh` 目录下
- 对网页相关位置的翻译，主要涉及 `/asset` 目录下的静态网页
- 为了适应翻译和部署，对原项目中的部分 Dart 代码进行了修改。主要是将 `/book` 替换为了 `/book_zh`；将`/site` 替换为了 `/docs`

由于本人英语水平有限，大部分翻译内容取自 [craftinginterpreters_zh][] 或 ChatGPT 或者翻译网站，若发现有任何出错的地方，欢迎提交 PR 或者 [联系我][zhihu]。

项目尚未开发完毕，正在持续更新中。

[zh reading online]: https://zaslee.github.io/craftinginterpreters/
[en reading online]: http://craftinginterpreters.com
[craftinginterpreters_zh]: https://github.com/GuoYaxiang/craftinginterpreters_zh
[zhihu]: https://www.zhihu.com/people/an-you-wo

## Crafting Interpreters
本项目包含该书的 Markdown 文本、两个解释器的完整实现，以及将两者编织成最终网站的构建系统。原仓库地址: [Crafting Interpreters][crafting interpreters github]

[crafting interpreters github]: https://github.com/munificent/craftinginterpreters

## 构建

### 前提条件

原项目在 OS X 机器上开发的，但任何 POSIX 系统都可以。

大部分工作都由 make 协调完成。构建脚本、测试运行程序和其他工具都是用 [Dart][] 编写的。[这里][install]有安装 Dart 的说明。安装好 Dart 并放在路径上后，运行

```sh
$ make get
```

[dart]: https://dart.dev/
[install]: https://dart.dev/get-dart

这会下载编译和测试脚本使用的所有软件包。

为了编译这两个解释器，还需要一个 C 编译器和 `javac`。

### 构建

完成上述设置后，请尝试：

```sh
$ make
```

如果一切正常，就会生成本书的网站，并编译两个解释器 clox 和 jlox。你可以直接从 repo 的根目录运行这两个解释器：

```sh
$ ./clox
$ ./jlox
```

### 本书的黑科技

Markdown和源代码片段使用手工编写的静态站点生成器编织成最终的HTML，该生成器最初是作者[第一本书][gpp]的一个小[Python脚本][py] ，后来以某种方式发展成了一个接近真实程序的东西。

[py]: https://github.com/munificent/game-programming-patterns/blob/master/script/format.py
[gpp]: http://gameprogrammingpatterns.com/

生成的 HTML 保存在`docs/`下的软件仓库中。它由 Markdown 和代码片段组合而成，前者放在 `book/` 目录中，后者放在 `java/` 和 `c/` 目录中。

执行所有魔法的脚本是 `tool/bin/build.dart`。你可以直接运行它，或者运行：

```sh
$ make book
```

这样就能一次性生成整个网站。如果你是渐进式开发，则需要运行开发服务器：

```sh
$ make serve
```

它在本地主机上运行一个小小的 HTTP 服务器，根植于 `docs/` 目录。只要你请求一个页面，它就会重新生成源文件已更改的任何文件，包括 Markdown 文件、解释器源文件、模板和资产。只需让它继续运行，在本地编辑文件，然后刷新浏览器即可看到更改。

### 构建解释器

你可以这样构建各个解释器：

```sh
$ make clox
$ make jlox
```

这将构建每个解释器的最终版本，该版本将出现在本书各部分的结尾处。

你还可以在每章末尾看到解释器的样子（我用它来确保解释器在书的中间部分也能正常工作）。这是由脚本 `tool/bin/split_chapters.dart` 驱动的，它使用代码片段的相同注释标记来确定每章中的代码块。该脚本只提取每章末尾的代码片段，并在 `gen/` 目录中生成一份新的源代码副本，每章的代码放在一个目录中。(这也是一种更简单的查看源代码的方法，因为它们去除了所有分散注意力的标记注释)。

这样，就可以分别编译这些代码了。运行：

```sh
$ make c_chapters
```

在 `build/` 目录中，每一章都会有一个可执行文件，如 `chap14_chunks` 等。同样

```sh
$ make java_chapters
```

这将把 Java 代码编译到每章子目录下 `build/gen/` 中的类文件中。

## 测试

我有一套完整的 Lox 测试套件，用来确保书中的解释器能完成它们应该做的事情。测试用例位于 `test/`。Dart 程序 `tool/bin/test.dart` 是一个测试运行器，它可以在 Lox 解释器上运行每个测试文件，解析结果，并验证测试是否完成了预期的工作。

你可以使用多种解释器来运行测试：


```sh
$ make test       # 测试 clox 和 jlox 的最终版本
$ make test_clox  # 测试 clox 的最终版本
$ make test_jlox  # 测试 jlox 的最终版本
$ make test_c     # 测试每一章的 clox
$ make test_java  # 测试每一章的 jlox
$ make test_all   # 测试以上所有的选项
```

### 测试你的实现

欢迎使用测试套件和测试运行程序来测试你自己的 Lox 实现。测试运行器位于 `tool/bin/test.dart` 中，可以使用 `--interpreter` 给出一个自定义解释器可执行文件来运行。例如，如果你有一个位于 `my_code/boblox` 的解释器可执行文件，你可以像下面这样测试它：

```sh
$ dart tool/bin/test.dart clox --interpreter my_code/boblox
```

你仍然需要告诉它要运行哪个测试套件，因为这决定了测试的预期。如果解释器的行为类似 jlox，则使用 "jlox "作为测试套件名称。如果解释器的行为类似于 clox，则使用 "clox"。如果你的解释器只完成到书中某一章的结尾，你可以使用该章作为套件，如 "chap10_functions"。有关所有章节的名称，请参阅 Makefile。

如果解释器需要其他命令行参数，请使用 `--arguments` 将参数传递给测试运行器，测试运行器将转发给解释器。

## 仓库布局

*   `asset/` – 用于生成网站的 Sass 文件和 jinja2 模板。
*   `book/` - 每章文本的 Markdown 文件。
*   `book_zh/` - 每章翻译的 Markdown 文件。
*   `build/` - 中间文件和其他构建输出（网站本身除外）放在这里。将不会提交到 Git。
*   `c/` –  用 C 编写的解释器 clox 的源代码。如果你喜欢，这里还包含一个 XCode 项目。
*   `gen/` –  由 GenerateAst.java 生成的 Java 源文件。将不会提交。
*   `java/` – 用 Java 编写的解释器 jlox 的源代码。
*   `note/` – 各种研究、笔记、TODO 和其他杂项。
*   `note/answers` – 习题的答案示例。不准作弊！
*   `docs/` – 最终生成的网站。这里的大部分内容由 build.py 生成，但是字体、图片和 JS 只存在于此文件夹。所有内容都将提交。
*   `test/` – Lox 的测试用例。
*   `tool/` – 包含联编、测试和其他脚本的 Dart 包。
