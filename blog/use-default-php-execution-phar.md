# 怎么使用默认的 PHP 执行 phar 包？

最近在开发一个安装程序的时候，打包为了 phar 包，遇到了一个问题就是打包后的 phar 包不能省略 php 去执行。

```bash
# 正常运行
php install.phar

# 报错
./install.phar
```

省略 php 去运行时报错：

```bash
$ ./install.phar
./install.phar: line 1: ?php: No such file or directory
./install.phar: line 3: =: command not found
./install.phar: line 5: syntax error near unexpected token `'phar','
./install.phar: line 5: `if (in_array('phar', stream_get_wrappers()) && class_exists('Phar', 0)) {'
```

到这里就不知道具体原因了，因为按照正常 phar 的流程打包是没有问题的，但是确实是不能省略运行。

在网上搜索一圈也没有具体的答案，想到 `composer` 可以省略 php 去运行，于是乎去查看了一下 `composer` 的源码，

找到了一个 [compile](https://github.com/composer/composer/blob/cbc686c16a21f01ba351d74be4d1ef84c89fc8f2/src/Composer/Compiler.php#L43) 方法，发现代码中有一个`setStub`的操作，调用了`getStub()`：

```php
    private function getStub()
    {
        $stub = <<<'EOF'
#!/usr/bin/env php
<?php
/*
 * This file is part of Composer.
 *
 * (c) Nils Adermann <naderman@naderman.de>
 *     Jordi Boggiano <j.boggiano@seld.be>
 *
 * For the full copyright and license information, please view
 * the license that is located at the bottom of this file.
 */
// Avoid APC causing random fatal errors per https://github.com/composer/composer/issues/264
if (extension_loaded('apc') && filter_var(ini_get('apc.enable_cli'), FILTER_VALIDATE_BOOLEAN) && filter_var(ini_get('apc.cache_by_default'), FILTER_VALIDATE_BOOLEAN)) {
    if (version_compare(phpversion('apc'), '3.0.12', '>=')) {
        ini_set('apc.cache_by_default', 0);
    } else {
        fwrite(STDERR, 'Warning: APC <= 3.0.12 may cause fatal errors when running composer commands.'.PHP_EOL);
        fwrite(STDERR, 'Update APC, or set apc.enable_cli or apc.cache_by_default to 0 in your php.ini.'.PHP_EOL);
    }
}
if (!class_exists('Phar')) {
    echo 'PHP\'s phar extension is *missing. Composer requires it to run. Enable the extension or recompile php without --disable-phar then try again.' . PHP_EOL;
    exit(1);
}
Phar::mapPhar('composer.phar');
EOF;

        // add warning once the pha*r is older than 60 days
        if (preg_match('{^[a-f0-9]+$}', $this->version)) {
            $warningTime = ((int) $this->versionDate->format('U')) + 60 * 86400;
            $stub .= "define('COMPOSER_DEV_WARNING_TIME', $warningTime);\n";
        }

        return $stub . <<<'EOF'
require 'phar://composer.phar/bin/composer';
__HALT_COMPILER();
EOF;
    }
```

看到这里，就觉得可能是这里的问题，因为我是直接使用了`createDefaultStub`方法去创建的`stub`

```php
$phar->setStub($phar->createDefaultStub('install.php'));
```

参考 `composer` 的代码进行了一些修改：

```php
$dirname = dirname(__DIR__);
$pharFile = $dirname . '/install.phar';

if (file_exists($pharFile)) {
    unlink($pharFile);
}

function getStub()
{
    $stub = <<<'EOF'
#!/usr/bin/env php
<?php
if (!class_exists('Phar')) {
    echo 'PHP\'s phar extension is missing. Install requires it to run. Enable the extension or recompile php without --disable-phar then try again.' . PHP_EOL;
    exit(1);
}
Phar::mapPhar('install.phar');
EOF;

    return $stub . <<<'EOF'
require 'phar://install.phar/bin/install';
__HALT_COMPILER();
EOF;
}

$phar = new Phar($pharFile, 0, 'install.phar');
$phar->startBuffering();
$phar->buildFromDirectory($dirname);
$phar->delete('bin/build.php');
$phar->delete('.gitignore');
$phar->delete('install');
$content = file_get_contents($dirname . '/install');
$content = preg_replace('{^#!/usr/bin/env php\s*}', '', $content);
$phar->addFromString('bin/install', $content);
$phar->setStub(getStub());
$phar->stopBuffering();
```

再次尝试后，就可以省略 php 去运行了。

最后，提供了一个获取 php 信息的 phar 包，用于快速获取一些信息，如版本、ini 目录、是否为 zts 和 debug 版本等

[https://github.com/lufei/phpinfo](https://github.com/lufei/phpinfo)

下载 `phpinfo.phar`：

```bash
chmod +x phpinfo.phar

cp phpinfo.phar /usr/local/bin/phpinfo
```

执行 `phpinfo`：

```bash
$ phpinfo
PHP_SAPI：cli
PHP_VERSION：8.0.6
PHP_ZTS：false
PHP_DEBUG：false
PHP_OS：Darwin
PHP_BINARY：/Users/lufei/.phpbrew/php/php-8.0.6/bin/php
PHP_CONFIG_FILE_PATH：/Users/lufei/.phpbrew/php/php-8.0.6/etc
PHP_LOADED_CONFIG_FILE：/Users/lufei/.phpbrew/php/php-8.0.6/etc/php.ini
PHP_CONFIG_FILE_SCAN_DIR：/Users/lufei/.phpbrew/php/php-8.0.6/var/db
```
