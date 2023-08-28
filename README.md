# 4d-topic-php
Setup PHP for 4D

Clone or download [php-src](https://github.com/php/php-src).

By default, `re2c` is missing and `bison` is too old.

```
brew install re2c
brew install bison
echo 'export PATH="/opt/homebrew/opt/bison/bin:$PATH"' >> ~/.zshrc
```

Restart Terminal.
