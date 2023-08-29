# 4d-topic-php
Setup PHP for 4D

## Build static PHP with minimal dependencies

Clone or download [php-src](https://github.com/php/php-src).

By default, `re2c` is missing and `bison` is too old. 

```
brew install re2c
brew install bison
echo 'export PATH="/opt/homebrew/opt/bison/bin:$PATH"' >> ~/.zshrc
```

Restart Terminal.

```
cd {php-src-master}
autoreconf
export LIBS="-lz"
/configure --enable-static --without-iconv --prefix={path-to-install-dir}
make
make install
```

This will install static `php-cgi` with only the following system dependencies:
 
```
/usr/lib/libresolv.9.dylib
/usr/lib/libnetwork.dylib
/usr/lib/libSystem.B.dylib
/usr/lib/libz.1.dylib
/usr/lib/libsqlite3.dylib
```

For comparision, the v20 `php-fcgi-4d` file looks like this:

```
/usr/lib/libresolv.9.dylib
/usr/lib/libnetwork.dylib
/usr/lib/libSystem.B.dylib
```

According to the [PHP modules support](https://doc.4d.com/4Dv20/4D/20/PHP-modules-support.300-6238471.en.html) documentation, `SQLite3` is enabled so the library must be statically linked. `Zip`, `Zlib`, `Iconv`, as well as XML-releated featured that depend on `Zlib` are all disabled.

To create universal binary, restart Terminal using Rosetta, `make distclean`, repeat, then `lipo -create`.

## Build static PHP with embedded `libz` and `libsqlite3`

Download libraries, then `lipo -create`:

```
brew fetch --bottle-tag=arm64_big_sur sqlite
brew fetch --bottle-tag=x86_64_big_sur sqlite
brew fetch --bottle-tag=arm64_big_sur zlib
brew fetch --bottle-tag=x86_64_big_sur zlib
```

Indicate static library location:

```
export LDFLAGS="-L{path-to-static-library-dir}"
export LIBS="-lz -lsqlite3"
```

**Error**: `sqlite3.c:446:6: error: call to undeclared function 'sqlite3_load_extension'`

c.f. https://bugs.python.org/issue44997

We could edit */ext/sqlite3/sqlite3.stub.php* like so:

```c
#ifndef SQLITE_OMIT_LOAD_EXTENSION
    /** @tentative-return-type */
#    public function loadExtension(string $name): bool {}
#define SQLITE_OMIT_LOAD_EXTENSION 1
#endif
```
…but this won't update the source files.

**Solution**: Remove from *ext/sqlite3/sqlite3_arginfo.h*

```c
#if !defined(SQLITE_OMIT_LOAD_EXTENSION)
ZEND_BEGIN_ARG_WITH_TENTATIVE_RETURN_TYPE_INFO_EX(arginfo_class_SQLite3_loadExtension, 0, 1, _IS_BOOL, 0)
	ZEND_ARG_TYPE_INFO(0, name, IS_STRING, 0)
ZEND_END_ARG_INFO()
#endif
```

…and add to *ext/sqlite3/sqlite3.c*

```c
#define SQLITE_OMIT_LOAD_EXTENSION 1
```

This will install static `php-cgi` with the same 3 dependencies as `php-fcgi-4d`.

## Build static PHP with embedded `libiconv`

Download libraries, then `lipo -create`:

```
brew fetch --bottle-tag=arm64_big_sur libiconv
brew fetch --bottle-tag=x86_64_big_sur libiconv
```

**Error**: `configure: error: Please reinstall the iconv library.`

Indicate static library location:

We could indicate static library location. Also add `-liconv` to `LDFLAGS`:

```
export CFLAGS="-I{path-to-user-header-dir}" 
export LDFLAGS="-L{path-to-static-library-dir} -liconv"
export LIBS="-lz  -lsqlite3 -liconv"
```

…but this won't eliminate compiler errors.

**Solution**: Remove `#undef` *ext/iconv/iconv.c*

```c
#ifdef HAVE_LIBICONV
//#undef iconv
#endif
```

## Does it actually work?

Rename the program as `php-fcgi-4d` and replace the one inside 4D.app.

Test call:

```4d
var $returnValue : Text

$success:=PHP Execute(""; "phpversion"; $returnValue)

var $response : Text
ARRAY TEXT($labels; 0)
ARRAY TEXT($values; 0)

PHP GET FULL RESPONSE($response; $labels; $values)
```

**Result**: `$success` is `False`. Activity Monitor shows multiple instances of `php-fcgi-4d`. `X-4DPHP-Error-php-interpreter` contains the message:

```
PHP Fatal error:  Uncaught Error: Call to undefined function mb_convert_encoding() in _4D_Execute_PHP.php:131
```

**Solution**: Add `--enable-mbstring` to `configure`. 

## Build static PHP with embedded `libzip`

The homebrew distribution of [libzip](https://formulae.brew.sh/formula/libzip) is dynamic-only.

**Solution**: Build from [source](https://github.com/nih-at/libzip). Uncheck `BUILD_SHARED_LIBS` in CMake.

## Build static PHP with modules and extensions

|Modules or Extension|Configure Option|4D v20|This Repository|
|-|-|:-:|:-:|
|[BCMath](https://www.php.net/manual/en/book.bc.php)|`--enable-bcmath`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Calendar](https://www.php.net/manual/en/book.calendar.php)|`--enable-calendar`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Character type checking](https://www.php.net/manual/en/book.ctype.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Client URL Library](https://www.php.net/manual/en/book.curl.php)|`--with-curl=DIR`|*disabled*|*disabled*|
|[Date and Time](https://www.php.net/manual/en/book.datetime.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Document Object Model](https://www.php.net/manual/en/book.dom.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Exchangeable image information](https://www.php.net/manual/en/book.exif.php)|`--enable-exif`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[File Information](https://www.php.net/manual/en/book.fileinfo.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Data Filtering](https://www.php.net/manual/en/book.filter.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[FTP](https://www.php.net/manual/en/book.ftp.php)|`--enable-ftp`|<ul><li>- [x] </li></ul>|*disabled*||
|[GNU Multiple Precision](https://www.php.net/manual/en/book.gmp.php)|`--with-gmp`|*disabled*|<ul><li>- [x] </li></ul>|
|[HASH Message Digest Framework](https://www.php.net/manual/en/book.hash.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[iconv](https://www.php.net/manual/en/book.iconv.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Image Processing and GD](https://www.php.net/manual/en/book.image.php)|`--enable-gd --with-avif --with-webp --with-jpeg --enable-gd-jis-conv`|*disabled*|<ul><li>- [x] </li></ul>|
|[JavaScript Object Notation](https://www.php.net/manual/en/book.json.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[libxml](https://www.php.net/manual/en/book.libxml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Lightweight Directory Access Protocol](https://www.php.net/manual/en/book.ldap.php)|`--with-ldap`|<ul><li>- [x] </li></ul>|*disabled*||
|[Multibyte String](https://www.php.net/manual/en/book.mbstring.php)|`--enable-mbstring`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[OpenSSL](https://www.php.net/manual/en/book.openssl.php)|`--with-openssl`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Regular Expressions (Perl-Compatible)](https://www.php.net/manual/en/book.pcre.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[PHP Data Objects](https://www.php.net/manual/en/book.pdo.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[SQLite Functions (PDO_SQLITE)](https://www.php.net/manual/en/ref.pdo-sqlite.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Phar](https://www.php.net/manual/en/book.phar.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[POSIX](https://www.php.net/manual/en/book.posix.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Random Number Generators and Functions Related to Randomness](https://www.php.net/manual/en/book.random.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Reflection](https://www.php.net/manual/en/book.reflection.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Sessions](https://www.php.net/manual/en/features.sessions.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[SimpleXML](https://www.php.net/manual/en/book.simplexml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Sockets](https://www.php.net/manual/en/book.sockets.php)|`--enable-sockets`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[SQLite3](https://www.php.net/manual/en/book.sqlite3.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Standard PHP Library (SPL)](https://www.php.net/manual/en/book.spl.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Tidy](https://www.php.net/manual/en/book.tidy.php)|`--with-tidy=DIR`|*disabled*|<ul><li>- [x] </li></ul>|
|[Tokenizer](https://www.php.net/manual/en/book.tokenizer.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[XML Parser](https://www.php.net/manual/en/book.xml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[XMLReader](https://www.php.net/manual/en/book.xmlreader.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[XMLWriter](https://www.php.net/manual/en/book.xmlwriter.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Zip](https://www.php.net/manual/en/book.zip.php)|`--with-zip`|*disabled*|<ul><li>- [x] </li></ul>|
|[Zlib Compression](https://www.php.net/manual/en/book.zlib.php)|`--with-zlib`|*disabled*|<ul><li>- [x] </li></ul>|

### Configure Options

```
./configure
 --with-tidy=DIR
 --with-zlib
 --with-gmp
 --with-zip
 --with-openssl
 --enable-static
 --enable-bcmath
 --enable-calendar
 --enable-exif
 --enable-ftp
 --enable-gd --with-avif --with-webp --with-jpeg --enable-gd-jis-conv
 --enable-sockets
 --enable-mbstring
```

### Static Libraries

```
export LIBS="
 -lz
 -lsqlite3
 -liconv
 -lbz2
 -lzip
 -lzstd
 -lonig
 -llzma
 -lgd -lwebp -lavif -ltiff -lpng16 -lsharpyuv
 -ltidy
 -lgmp
 -lcrypto -lssl"
```

## Build static PHP with embedded `libcurl`

Add `--with-curl`.

**Error**: `configure: error: There is something wrong. Please check config.log for more information.`

**Solution (kind of)**: Edit *configure* to skip the hard testing of `curl_easy_perform`.

**Error**: Undefined symbols: `libunistring` expects a non-plug version of `libiconv`. 

**Solution**: Compile and use a hybrid version that contains both names (`_iconv` and `_libiconv`).

Just throw more libraries and see what sticks?

```
export LIBS="
 -lz
 -lsqlite3
 -liconv
 -lbz2
 -lzip
 -lzstd
 -lonig
 -llzma
 -lgd
 -lwebp -lavif -ltiff -lpng16 -lsharpyuv
 -ltidy
 -lgmp
 -lcrypto -lssl
 -lcurl -lnghttp2 -lidn2 -lssh2 -lldap -llber
 -lbrotlidec -lbrotlienc -lbrotlicommon
 -lsasl2
 -lunistring
 -lrtmp
 -framework GSS
 -framework SystemConfiguration
 -framework Cocoa
 -framework Security
 -framework LDAP
 -framework Kerberos"
```

Also specify subdirectories for headers because some symbols are constants.

```
export CFLAGS="
 -I{path-to-user-header-dir}
 -I{path-to-user-header-dir}/brotli
 -I{path-to-user-header-dir}/curl"
```

## Build static PHP with embedded `libldap`

**Error**: `configure: error: LDAP build check failed. Please check config.log for more information.`

No workaround. Yet.
