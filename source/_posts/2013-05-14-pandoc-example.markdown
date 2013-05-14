---
layout: post
title: "pandoc example"
date: 2013-05-14 01:09
comments: true
categories: [pandoc, command]
---

from [pondoc ](http://johnmacfarlane.net/pandoc/demos.html)

* * * * *

 

To see the output created by each of the commands below, click on the
name of the output file:

1.  HTML fragment:

        pandoc README -o example1.html

2.  Standalone HTML file:

        pandoc -s README -o example2.html

3.  HTML with smart quotes, table of contents, CSS, and custom footer:

        pandoc -s -S --toc -c pandoc.css -A footer.html README -o example3.html

4.  LaTeX:

        pandoc -s README -o example4.tex

5.  From LaTeX to markdown:

        pandoc -s example4.tex -o example5.text

6.  reStructuredText:<!--more-->

        pandoc -s -w rst --toc README -o example6.text

7.  Rich text format (RTF):

        pandoc -s README -o example7.rtf

8.  Beamer slide show:

        pandoc -t beamer SLIDES -o example8.pdf

9.  DocBook XML:

        pandoc -s -S -w docbook README -o example9.db

    Chunked XHTML via DocBook
    and [xmlto](http://cyberelk.net/tim/xmlto/):

        xmlto xhtml -m config.xsl example9.db -o example9/

10. Man page:

        pandoc -s -w man pandoc.1.md -o example10.1

11. ConTeXt:

        pandoc -s -w context README -o example11.tex

    PDF via pandoc and ConTeXt’s `texexec`:

        texexec --pdf example11.tex # produces example11.pdf

12. Converting a web page to markdown:

        pandoc -s -r html http://www.gnu.org/software/make/ -o example12.text

13. From markdown to PDF:

        pandoc README -o example13.pdf

14. PDF with numbered sections and a custom LaTeX header:

        pandoc -N --template=mytemplate.tex --variable mainfont=Georgia --variable sansfont=Arial --variable monofont="Bitstream Vera Sans Mono" --variable fontsize=12pt --variable version=1.9 README --latex-engine=xelatex --toc -o example14.pdf

15. A wiki program using [Happstack](http://happstack.com/) and
    pandoc: [gitit](http://gitit.net/)

16. HTML slide shows:

        pandoc -s --mathml -i -t dzslides SLIDES -o example16a.html

        pandoc -s --webtex -i -t slidy SLIDES -o example16b.html

        pandoc -s --self-contained --webtex -i -t s5 SLIDES -o example16c.html

        pandoc -s --mathjax -i -t slideous SLIDES -o example16d.html

17. TeX math in HTML:

        pandoc math.text -s -o mathDefault.html

        pandoc math.text -s --mathml -o mathMathML.html

        pandoc math.text -s --webtex -o mathWebTeX.html

        pandoc math.text -s --mathjax -o mathMathJax.html

        pandoc math.text -s --latexmathml -o mathLaTeXMathML.html

18. Syntax highlighting of delimited code blocks:

        pandoc code.text -s --highlight-style pygments -o example18a.html

        pandoc code.text -s --highlight-style kate -o example18b.html

        pandoc code.text -s --highlight-style monochrome -o example18c.html

        pandoc code.text -s --highlight-style espresso -o example18d.html

        pandoc code.text -s --highlight-style haddock -o example18e.html

        pandoc code.text -s --highlight-style tango -o example18f.html

        pandoc code.text -s --highlight-style zenburn -o example18g.html

19. GNU Texinfo, converted to info, HTML, and PDF formats:

        pandoc README -s -o example19.texi

        makeinfo example19.texi -o example19.info

        makeinfo example19.texi --html -o example19

        texi2pdf example19.texi # produces example19.pdf

20. OpenDocument XML:

        pandoc README -s -w opendocument -o example20.xml

21. ODT (OpenDocument Text, readable by OpenOffice):

        pandoc README -o example21.odt

22. MediaWiki markup:

        pandoc -s -S -w mediawiki --toc README -o example22.wiki

23. EPUB ebook:

        pandoc -S README -o README.epub

24. Markdown citations:

        pandoc -s -S --biblio biblio.bib --csl chicago-author-date.csl CITATIONS -o example24a.html

        pandoc -s -S --biblio biblio.bib --csl mhra.csl CITATIONS -o example24b.html

        pandoc -s -S --biblio biblio.bib --csl ieee.csl CITATIONS -t man -o example24c.1

25. Textile writer:

        pandoc -s -S README -t textile -o example25.textile

26. Textile reader:

        pandoc -s -S example25.textile -f textile -t html -o example26.html

27. Org-mode:

        pandoc -s -S README -o example27.org

28. AsciiDoc:

        pandoc -s -S README -t asciidoc -o example28.txt

29. Word docx:

        pandoc -s -S README -o example29.docx

30. LaTeX math to docx:

        pandoc -s math.tex -o example30.docx

31. DocBook to markdown:

        pandoc -f docbook -t markdown -s howto.xml -o example31.text

