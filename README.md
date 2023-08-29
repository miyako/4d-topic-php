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

To create universal binary, restart Terminal using Rosetta, repeat, then `lipo -create`.

## Build static PHP with embedded `libz` and `libsqlite3`

Download libraries, then `lipo -create`:

```
brew fetch --bottle-tag=arm64_big_sur sqlite
brew fetch --bottle-tag=x86_64_big_sur sqlite
brew fetch --bottle-tag=arm64_big_sur zlib
brew fetch --bottle-tag=x86_64_big_sur zlib
```

Direct path to static library directory:

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

We could link to the homebrew header file and also add `-liconv` to `LDFLAGS`:

```
export CFLAGS="-I{path-to-user-header-dir}" 
export LDFLAGS="-L{path-to-static-library-dir} -liconv"
export LIBS="-lz -liconv"
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

**Solution**: Add `--enable-mbstring` to `configure`. In fact, enable all the extensions available in 4D.

|Modules or Extension|Configure Option|4D v20|This Repository|
|-|-|:-:|:-:|
|[BCMath](https://www.php.net/manual/en/book.bc.php)|`--enable-bcmath`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Calendar](https://www.php.net/manual/en/book.calendar.php)|`--enable-calendar`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Character type checking](https://www.php.net/manual/en/book.ctype.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Date and Time](https://www.php.net/manual/en/book.datetime.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Document Object Model](https://www.php.net/manual/en/book.dom.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Exchangeable image information](https://www.php.net/manual/en/book.exif.php)|`--enable-exif`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[File Information](https://www.php.net/manual/en/book.fileinfo.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Data Filtering](https://www.php.net/manual/en/book.filter.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[FTP](https://www.php.net/manual/en/book.ftp.php)|`--enable-ftp`|<ul><li>- [x] </li></ul>||
|[Image Processing and GD](https://www.php.net/manual/en/book.image.php)|`--enable-gd --with-avif --with-webp --with-jpeg --enable-gd-jis-conv`|*disabled*|<ul><li>- [x] </li></ul>|
|[HASH Message Digest Framework](https://www.php.net/manual/en/book.hash.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[iconv](https://www.php.net/manual/en/book.iconv.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[JavaScript Object Notation](https://www.php.net/manual/en/book.json.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[libxml](https://www.php.net/manual/en/book.libxml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Lightweight Directory Access Protocol](https://www.php.net/manual/en/book.ldap.php)|`--with-ldap`|<ul><li>- [x] </li></ul>||
|[Multibyte String](https://www.php.net/manual/en/book.mbstring.php)|`--enable-mbstring`|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Regular Expressions (Perl-Compatible)](https://www.php.net/manual/en/book.pcre.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[PHP Data Objects](https://www.php.net/manual/en/book.pdo.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[SQLite Functions (PDO_SQLITE)](https://www.php.net/manual/en/ref.pdo-sqlite.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Phar](https://www.php.net/manual/en/book.phar.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[POSIX](https://www.php.net/manual/en/book.posix.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Random Number Generators and Functions Related to Randomness](https://www.php.net/manual/en/book.random.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Reflection](https://www.php.net/manual/en/book.reflection.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Sessions](https://www.php.net/manual/en/features.sessions.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[SimpleXML](https://www.php.net/manual/en/book.simplexml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[SQLite3](https://www.php.net/manual/en/book.sqlite3.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Standard PHP Library (SPL)](https://www.php.net/manual/en/book.spl.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[Tokenizer](https://www.php.net/manual/en/book.tokenizer.php)|(default)|<ul><li>- [x] </li></ul>|<ul><li>- [x] </li></ul>|
|[XML Parser](https://www.php.net/manual/en/book.xml.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[XMLReader](https://www.php.net/manual/en/book.xmlreader.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[XMLWriter](https://www.php.net/manual/en/book.xmlwriter.php)|(default)|*disabled*|<ul><li>- [x] </li></ul>|
|[Zip](https://www.php.net/manual/en/book.zip.php)|`--with-zip`|*disabled*|<ul><li>- [x] </li></ul>|
|[Zlib Compression](https://www.php.net/manual/en/book.zlib.php)|`--with-zlib`|*disabled*|<ul><li>- [x] </li></ul>|

### Configure Options

```
./configure
 --with-tidy
 --with-zip
 --with-zlib
 --enable-static
 --enable-bcmath
 --enable-calendar
 --enable-exif
 --enable-gd --with-avif --with-webp --with-jpeg --enable-gd-jis-conv
 --enable-mbstring
```

### Static Libraries

```
export LIBS="-lz -liconv -lonig -llzma -lgd -lwebp -lavif -ltiff -lpng16-lsharpyuv -ltidy"
```

---

Typical depenceies:

- [ ] aspell/0.60.8/lib/libpspell.15.dylib
- [x] libpq-15.4
- [ ] gettext/0.21.1/lib/libintl.8.dylib
- [x] tidy-html5-5.8.0
- [ ] aspell/0.60.8/lib/libaspell.15.dylib
- [ ] krb5/1.21.2/lib/libcom_err.3.0.dylib
- [ ] icu4c/73.2/lib/libicuio.73.2.dylib
- [ ] krb5/1.21.2/lib/libk5crypto.3.1.dylib
- [ ] freetds/1.3.20/lib/libsybdb.5.dylib
- [x] openldap-2.6.6
- [x] openldap-2.6.6-liblber
- [ ] krb5/1.21.2/lib/libgssapi_krb5.2.2.dylib
- [ ] argon2/20190702_1/lib/libargon2.1.dylib
- [ ] gmp/6.2.1_1/lib/libgmp.10.dylib
- [ ] krb5/1.21.2/lib/libkrb5support.1.1.dylib
- [x] libzip-1.10.1
- [x] gd-2.3.3_5
- [ ] krb5/1.21.2/lib/libkrb5.3.3.dylib
- [ ] openssl@3/3.1.2/lib/libssl.3.dylib
- [ ] pcre2/10.42/lib/libpcre2-8.0.dylib
- [ ] curl/8.2.1/lib/libcurl.4.dylib
- [ ] unixodbc/2.3.12/lib/libodbc.2.dylib
- [ ] sqlite/3.43.0/lib/libsqlite3.0.dylib
- [ ] libsodium/1.0.18_1/lib/libsodium.23.dylib
- [ ] brotli/1.0.9/lib/libbrotlidec.1.0.9.dylib
- [ ] brotli/1.0.9/lib/libbrotlienc.1.0.9.dylib
- [ ] highway/1.0.6/lib/libhwy.1.0.6.dylib
- [ ] openssl@3/3.1.2/lib/libcrypto.3.dylib
- [ ] libnghttp2/1.55.1/lib/libnghttp2.14.dylib
- [ ] libidn2/2.3.4_1/lib/libidn2.0.dylib
- [ ] libtool/2.4.7/lib/libltdl.7.dylib
- [x] oniguruma-6.9.8
- [ ] rtmpdump/2.4+20151223_2/lib/librtmp.1.dylib
- [x] libavif-0.11.1
- [ ] libssh2/1.11.0_1/lib/libssh2.1.dylib
- [ ] brotli/1.0.9/lib/libbrotlicommon.1.0.9.dylib
- [x] libpng-1.6.40
- [x] xz-5.4.4
- [ ] icu4c/73.2/lib/libicui18n.73.2.dylib
- [ ] icu4c/73.2/lib/libicuuc.73.2.dylib
- [x] zstd-1.5.5
- [ ] fontconfig/2.14.2/lib/libfontconfig.1.dylib
- [ ] freetype/2.13.1/lib/libfreetype.6.dylib
- [x] jpeg-turbo-3.0.0
- [ ] libunistring/1.1/lib/libunistring.5.dylib
- [x] libtiff/4.5.1/lib/libtiff.6.dylib
- [x] webp-1.3.1-libwebp
- [x] webp-1.3.1-libsharpyuv
- [ ] little-cms2/2.15/lib/liblcms2.2.dylib
- [x] jpeg-xl-0.8.2
- [ ] aom/3.6.1/lib/libaom.3.6.1.dylib
- [x] libvmaf-2.3.1
- [ ] icu4c/73.2/lib/libicudata.73.2.dylib

```
--without-iconv
```
