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

According to the [PHP modules support](https://doc.4d.com/4Dv20/4D/20/PHP-modules-support.300-6238471.en.html), `SQLite3` is enabled so the library must be statically linked. `Zip`, `Zlib`, `Iconv`, as well as XML-releated featured that depend on `Zlib` are all disabled.

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



---

Typical depenceies:

- [ ] aspell/0.60.8/lib/libpspell.15.dylib
- [ ] openldap/2.6.6/lib/liblber.2.dylib
- [x] libpq/15.4/lib/libpq.5.15.dylib
- [ ] gettext/0.21.1/lib/libintl.8.dylib
- [ ] tidy-html5/5.8.0/lib/libtidy.5.8.0.dylib
- [ ] aspell/0.60.8/lib/libaspell.15.dylib
- [ ] krb5/1.21.2/lib/libcom_err.3.0.dylib
- [ ] icu4c/73.2/lib/libicuio.73.2.dylib
- [ ] krb5/1.21.2/lib/libk5crypto.3.1.dylib
- [ ] freetds/1.3.20/lib/libsybdb.5.dylib
- [ ] openldap/2.6.6/lib/libldap.2.dylib
- [ ] krb5/1.21.2/lib/libgssapi_krb5.2.2.dylib
- [ ] argon2/20190702_1/lib/libargon2.1.dylib
- [ ] gmp/6.2.1_1/lib/libgmp.10.dylib
- [ ] krb5/1.21.2/lib/libkrb5support.1.1.dylib
- [ ] libzip/1.10.1/lib/libzip.5.5.dylib
- [ ] gd/2.3.3_5/lib/libgd.3.dylib
- [ ] krb5/1.21.2/lib/libkrb5.3.3.dylib
- [ ] openssl@3/3.1.2/lib/libssl.3.dylib
- [ ] pcre2/10.42/lib/libpcre2-8.0.dylib
- [ ] curl/8.2.1/lib/libcurl.4.dylib
- [ ] unixodbc/2.3.12/lib/libodbc.2.dylib
- [ ] webp/1.3.1/lib/libsharpyuv.0.0.1.dylib
- [ ] sqlite/3.43.0/lib/libsqlite3.0.dylib
- [ ] libsodium/1.0.18_1/lib/libsodium.23.dylib
- [ ] brotli/1.0.9/lib/libbrotlidec.1.0.9.dylib
- [ ] highway/1.0.6/lib/libhwy.1.0.6.dylib
- [ ] openssl@3/3.1.2/lib/libcrypto.3.dylib
- [ ] libnghttp2/1.55.1/lib/libnghttp2.14.dylib
- [ ] libidn2/2.3.4_1/lib/libidn2.0.dylib
- [ ] libtool/2.4.7/lib/libltdl.7.dylib
- [ ] oniguruma/6.9.8/lib/libonig.5.dylib
- [ ] rtmpdump/2.4+20151223_2/lib/librtmp.1.dylib
- [ ] libavif/0.11.1/lib/libavif.15.0.1.dylib
- [ ] libssh2/1.11.0_1/lib/libssh2.1.dylib
- [ ] brotli/1.0.9/lib/libbrotlicommon.1.0.9.dylib
- [ ] libpng/1.6.40/lib/libpng16.16.dylib
- [x] xz-5.4.4
- [ ] icu4c/73.2/lib/libicui18n.73.2.dylib
- [ ] icu4c/73.2/lib/libicuuc.73.2.dylib
- [x] zstd-1.5.5
- [ ] fontconfig/2.14.2/lib/libfontconfig.1.dylib
- [ ] freetype/2.13.1/lib/libfreetype.6.dylib
- [ ] jpeg-turbo/3.0.0/lib/libjpeg.8.3.2.dylib
- [ ] libunistring/1.1/lib/libunistring.5.dylib
- [ ] libtiff/4.5.1/lib/libtiff.6.dylib
- [ ] webp/1.3.1/lib/libwebp.7.1.7.dylib
- [ ] brotli/1.0.9/lib/libbrotlienc.1.0.9.dylib
- [ ] little-cms2/2.15/lib/liblcms2.2.dylib
- [x] jpeg-xl/0.8.2/lib/libjxl.0.8.2.dylib
- [ ] aom/3.6.1/lib/libaom.3.6.1.dylib
- [x] libvmaf-2.3.1
- [ ] icu4c/73.2/lib/libicudata.73.2.dylib

```
--without-iconv
```
