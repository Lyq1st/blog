# template fork from Yous

[![Build Status](https://travis-ci.org/yous/yous.github.io.svg?branch=source)](https://travis-ci.org/yous/yous.github.io)

Blog for hackers.

## License

All articles are licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).

## Blog

[Lyq's blog](http://blog.droid-sec.com/)

## change code style

install pygments:

```bash
pip install pygments
gem install pygments.rb
```
config _config.yml:
```bash
highlighter: pygments
mardown: redcarpet
```

enum style:

```python
>>> from pygments.styles import STYLE_MAP
>>> STYLE_MAP.keys()
['monokai', 'manni', 'rrt', 'perldoc', 'borland', 'colorful', 'default', 'murphy', 'vs', 'trac', 'tango', 'fruity', 'autumn', 'bw', 'emacs', 'vim', 'pastie', 'friendly', 'native']
```python

```bash
pygmentize -S perldoc -f html -a .highlight > you/path/pygments.css
```
convert pygments.css pygments.css _syntax-highlighting.scss on http://sebastianpontow.de/css2compass/

or change .css to .scss