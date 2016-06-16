flexmark-java
=============

A rework of [commonmark-java] to generate AST which allows recreating the original source, full
source position references for all elements in the source and easier JetBrains Open API PsiTree
generation.

Motivation for this was the need to replace [pegdown] parser in [Markdown Navigator] plugin.
[pegdown] has a great feature set but its speed in general is not great and for pathological
input either hangs or practically hangs during parsing.

[commonmark-java] has an excellent parsing architecture that is easy to understand and extend.
The goal was to ensure that adding source position tracking in the AST would not change the ease
of parsing and generating the AST more than absolutely necessary.

Reasons for choosing [commonmark-java] as the parser are detailed in
[Pegdown - Achilles heel of the Markdown Navigator plugin]. Now that I have reworked the core
and added a few extensions I am extremely satisfied with my choice.

Another goal was to improve the ability of extensions to modify the behaviour of the parser so
that any dialect of markdown could be implemented through the extension mechanism. This included
adding an easily extensible options mechanism that would allow setting of all options in one
place and having the parser, renderer and extensions use these options to modify behavior,
including disabling some core block parsers.

This is a work in progress with many API changes.

No attempt is made to keep backward API compatibility to the original project.

Progress so far
---------------

- Wiki added [flexmark-java wiki]

- Unified options architecture to configure: parser, renderer and any custom extensions. This
  includes the list of extensions to use. Making a single argument configure the environment.
  These are also available during parsing and rendering phases for use by extensions.

- Test architecture based on original `spec.txt` augmented with:

    - expected AST so it is validated by tests

    - options can be specified for individual tests so that one file can validate all options
      available for the extension/core feature.  

    - full spec file generated with expected HTML and AST replaced with generated counterparts
      to make updating expected test results easier for new or modified tests.

    - section and example number added to each example opening line for cross referencing test
      results to test source.

- Rework `HtmlRenderer` to allow inserting rendered HTML into different parts of the generated
  HTML document. Now can generate HTML for top/bottom of document.

- Enhance `HtmlWriter` to make it easier to generate indented html and eliminate the need to
  implement attribute map and boiler plate render children method in custom node renderers.

- Add `BlockPreProcessor` interface to allow customizing of block processing of paragraph blocks
  on closing. Effectively, the mechanism of removing reference definitions from the start of the
  paragraph was generalized to be usable by any block and extensible.

- Add `LinkRefProcessor` interface to allow customizing parsing of link refs for custom nodes,
  such as footnotes `[^]` and wiki links `[[]]` that affect parsing which could not be done
  with a post processor extension.

- Parser options to be implemented:
    - GitHub Extensions
        - [x] Fenced code blocks
        - [ ] Anchor links for headers with auto id generation
        - [x] Table Spans option to be implemented for gfm-tables extension
        - [x] Wiki Links with GitHub and Creole syntax
        - [x] Emoji Shortcuts with use GitHub emoji URL option
    - GitHub Syntax
        - [x] Strikethrough
        - [ ] Task Lists
        - [ ] No Atx Header Space
        - [x] Hard Wraps (achieved with SOFT_BREAK option changed to `"<br />"`)
        - [ ] Relaxed HR Rules
        - [x] Wiki links
    - Publishing
        - [x] Abbreviations
        - [x] Footnotes
        - [ ] Definitions
        - [ ] Table of Contents
    - Typographic
        - [ ] Quotes
        - [ ] Smarts
    - Suppress
        - [x] inline HTML
        - [x] HTML blocks
    - Processor Extensions
        - [ ] Jekyll front matter
        - [ ] GitBook link URL encoding
        - [ ] HTML comment nodes

- AST is built based on Nodes in the source not nodes needed for HTML generation. New nodes:
    - `Reference`
    - `Image`
    - `LinkRef`
    - `ImageRef`
    - `AutoLink`
    - `MailLink`
    - `Emphasis`
    - `StrongEmphasis`
    - `HtmlEntity`

- `spec.txt` now `ast_spec_txt` with an added section to each example that contains the expected
  AST so that the generated AST can be validated.

        ```````````````````````````````` example Links: 35
        [foo *bar](baz*)
        .
        <p><a href="baz*">foo *bar</a></p>
        .
        Document[0, 17]
          Paragraph[0, 17]
            Link[0, 15] textOpen:[0, 1, "["] text:[1, 9, "foo *bar"] textClose:[9, 10, "]"] linkOpen:[0, 0] urlOpen:[0, 0] url:[11, 15, "baz*"] urlClose:[0, 0] titleOpen:[0, 0] title:[0, 0] titleClose:[0, 0] linkClose:[0, 0]
              Text[1, 9] chars:[1, 9, "foo *bar"]
        ````````````````````````````````
    
Whitespace is left out. So all spans of text not in a node are implicitly white space.

I am very pleased with the decision to switch to [commonmark-java] based parser. Even though I
had to do major surgery on its innards to get full source position tracking and AST that matches
source elements, it is a pleasure to work with and is now a pleasure to extend a parser based ot
its original design.

Benchmarks
----------

Here are some basic benchmarking results:

| File             | commonmark-java | flexmark-java | intellij-markdown |    pegdown |
|:-----------------|----------------:|--------------:|------------------:|-----------:|
| README-SLOW      |         0.715ms |       1.000ms |           1.696ms |   16.823ms |
| VERSION          |         1.242ms |       1.618ms |           3.467ms |   44.950ms |
| commonMarkSpec   |        42.357ms |      71.312ms |         583.142ms |  616.430ms |
| markdown_example |        19.522ms |      25.025ms |         207.381ms | 1046.346ms |
| spec             |         8.642ms |      11.801ms |          33.733ms |  317.981ms |
| table            |         0.143ms |       0.302ms |           0.644ms |    3.964ms |
| table-format     |         1.417ms |       1.755ms |           3.795ms |   24.867ms |
| wrap             |         4.605ms |       8.767ms |          14.768ms |   91.417ms |

Ratios of above:

| File             | commonmark-java | flexmark-java | intellij-markdown |   pegdown |
|:-----------------|----------------:|--------------:|------------------:|----------:|
| README-SLOW      |            1.00 |          1.40 |              2.37 |     23.54 |
| VERSION          |            1.00 |          1.30 |              2.79 |     36.19 |
| commonMarkSpec   |            1.00 |          1.68 |             13.77 |     14.55 |
| markdown_example |            1.00 |          1.28 |             10.62 |     53.60 |
| spec             |            1.00 |          1.37 |              3.90 |     36.79 |
| table            |            1.00 |          2.11 |              4.51 |     27.76 |
| table-format     |            1.00 |          1.24 |              2.68 |     17.55 |
| wrap             |            1.00 |          1.90 |              3.21 |     19.85 |
| -----------      |       --------- |     --------- |         --------- | --------- |
| overall          |            1.00 |          1.55 |             10.79 |     27.50 |

| File             | commonmark-java | flexmark-java | intellij-markdown |   pegdown |
|:-----------------|----------------:|--------------:|------------------:|----------:|
| README-SLOW      |            0.71 |          1.00 |              1.70 |     16.83 |
| VERSION          |            0.77 |          1.00 |              2.14 |     27.78 |
| commonMarkSpec   |            0.59 |          1.00 |              8.18 |      8.64 |
| markdown_example |            0.78 |          1.00 |              8.29 |     41.81 |
| spec             |            0.73 |          1.00 |              2.86 |     26.95 |
| table            |            0.47 |          1.00 |              2.13 |     13.14 |
| table-format     |            0.81 |          1.00 |              2.16 |     14.17 |
| wrap             |            0.53 |          1.00 |              1.68 |     10.43 |
| -----------      |       --------- |     --------- |         --------- | --------- |
| overall          |            0.65 |          1.00 |              6.98 |     17.79 |

* [VERSION.md] is the version log file I use for Markdown Navigator
* [commonMarkSpec.md] is a 33k line file used in [intellij-markdown] test suite for performance
  evaluation.
* [spec.txt] commonmark spec markdown file in the [commonmark-java] project
* [hang-pegdown.md] is a file containing a single line of 17 characters `[[[[[[[[[[[[[[[[[`
  which causes pegdown to go into a hyper-exponential parse time.
* [hang-pegdown2.md] a file containing a single line of 18 characters `[[[[[[[[[[[[[[[[[[` which
  causes pegdown to go into a hyper-exponential parse time.
* [wrap.md] is a file I was using to test wrap on typing performance only to discover that it
  has nothing to do with the wrap on typing code when 0.1 seconds is taken by pegdown to parse
  the file. In the plugin the parsing may happen more than once: syntax highlighter pass, psi
  tree building pass, external annotator.
* markdown_example.md a file with 10,000+ lines containing 500kB+ of text.

Contributing
------------

Pull requests, issues and comments welcome :smile:. For pull requests:

* Add tests for new features and bug fixes, preferably in the ast_spec.txt format
* Follow the existing style to make merging easier, as much as possible: 4 space indent.  

* * * 

License
-------

Copyright (c) 2015-2016 Atlassian and others.

Copyright (c) 2016, Vladimir Schneider,

BSD (2-clause) licensed, see LICENSE.txt file.

[.gitignore]: http://hsz.mobi
[Android Studio]: http://developer.android.com/sdk/installing/studio.html
[AppCode]: http://www.jetbrains.com/objc
[autolink-java]: https://github.com/robinst/autolink-java
[CLion]: https://www.jetbrains.com/clion
[commonmark-java]: https://github.com/atlassian/commonmark-java
[commonmark.js]: https://github.com/jgm/commonmark.js
[CommonMark]: http://commonmark.org/
[commonMarkSpec.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/commonMarkSpec.md
[Craig's List]: http://montreal.en.craigslist.ca/
[DataGrip]: https://www.jetbrains.com/datagrip
[flexmark-java]: https://github.com/vsch/flexmark-java
[gfm-tables]: https://help.github.com/articles/organizing-information-with-tables/
[GitHub Flavoured Markdown]: https://help.github.com/articles/basic-writing-and-formatting-syntax/
[GitHub Issues page]: ../../issues
[GitHub]: https://github.com/vsch/laravel-translation-manager
[hang-pegdown.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/hang-pegdown.md
[hang-pegdown2.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/hang-pegdown2.md
[idea-markdown]: https://github.com/nicoulaj/idea-markdown
[IntelliJ IDEA]: http://www.jetbrains.com/idea
[intellij-markdown]: https://github.com/valich/intellij-markdown 
[JetBrains plugin comment and rate page]: https://plugins.jetbrains.com/plugin/writeComment?pr=&pluginId=7896
[JetBrains plugin page]: https://plugins.jetbrains.com/plugin?pr=&pluginId=7896
[Kotlin]: http://kotlinlang.org
[Kramdown]: http://kramdown.gettalong.org/
[Markdown Navigator]: http://vladsch.com/product/markdown-navigator
[Markdown]: https://daringfireball.net/projects/markdown/
[Maven Central]: https://search.maven.org/#search|ga|1|g%3A%22com.atlassian.commonmark%22
[MultiMarkdown]: http://fletcherpenney.net/multimarkdown/
[nicoulaj/idea-markdown plugin]: https://github.com/nicoulaj/idea-markdown
[nicoulaj]: https://github.com/nicoulaj
[pegdown]: http://pegdown.org
[PhpExtra]: https://michelf.ca/projects/php-markdown/extra/
[PhpStorm]: http://www.jetbrains.com/phpstorm
[Pipe Table Formatter]: https://github.com/anton-dev-ua/PipeTableFormatter
[PyCharm]: http://www.jetbrains.com/pycharm
[RubyMine]: http://www.jetbrains.com/ruby
[Semantic Versioning]: http://semver.org/
[sirthias]: https://github.com/sirthias
[spec.txt]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/spec.md
[table.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/table.md
[VERSION.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/VERSION.md
[vsch/pegdown]: https://github.com/vsch/pegdown/tree/develop
[WebStorm]: http://www.jetbrains.com/webstorm
[wrap.md]: https://github.com/vsch/idea-multimarkdown/blob/master/test/data/performance/wrap.md
[Pegdown - Achilles heel of the Markdown Navigator plugin]: http://vladsch.com/blog/15
[flexmark-java wiki]: ../../wiki
