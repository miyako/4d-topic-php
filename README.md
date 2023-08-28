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

```
cd {php-src-master}
autoreconf
./configure
make
```

Typical depenceies:

- [ ] aspell/0.60.8/lib/libpspell.15.dylib
- [ ] openldap/2.6.6/lib/liblber.2.dylib
- [ ] libpq/15.4/lib/libpq.5.15.dylib
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
- [ ] xz/5.4.4/lib/liblzma.5.dylib
- [ ] icu4c/73.2/lib/libicui18n.73.2.dylib
- [ ] icu4c/73.2/lib/libicuuc.73.2.dylib
- [ ] zstd/1.5.5/lib/libzstd.1.5.5.dylib
- [ ] fontconfig/2.14.2/lib/libfontconfig.1.dylib
- [ ] freetype/2.13.1/lib/libfreetype.6.dylib
- [ ] jpeg-turbo/3.0.0/lib/libjpeg.8.3.2.dylib
- [ ] libunistring/1.1/lib/libunistring.5.dylib
- [ ] libtiff/4.5.1/lib/libtiff.6.dylib
- [ ] webp/1.3.1/lib/libwebp.7.1.7.dylib
- [ ] brotli/1.0.9/lib/libbrotlienc.1.0.9.dylib
- [ ] little-cms2/2.15/lib/liblcms2.2.dylib
- [ ] jpeg-xl/0.8.2/lib/libjxl.0.8.2.dylib
- [ ] aom/3.6.1/lib/libaom.3.6.1.dylib
- [x] libvmaf-2.3.1
- [ ] icu4c/73.2/lib/libicudata.73.2.dylib

```
--without-iconv
```