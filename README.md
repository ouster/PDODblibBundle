PDODblibBundle
--------------

I created this repo and instructions based on my experience of getting this working for a production environment 
With Silex/Doctrine 2 PHP framework connecting to an MS SQL Server 2012 DB on Ubuntu. 

Doctrine 2 does support any method of connecting to SQL Server on a Linux box. 

This git repo and instructions are for using the Microsoft Linux native driver and are based on the instructions for freeTDS driver
here: [https://github.com/LeaseWeb/LswDoctrinePdoDblib] thanks to Maurits van der Schee!

This bundle requires the following accessible from PHP extension directory:

You will need a sane build environment, sudo access and your wits about you!

* pdo.so # built from php sources
* pdo_odbc.so # built from php sources
* libmsodbcsql.so # created from Microsoft Download
* libsqlncli.so #optional for isql cli SQL Server access
* unixODBC-2.3.2
* PHP 5.x source for your PHP version (you need it to build pdo_odbc.so)

To get the microsoft drivers:
=============================
*msodbcsql-11.0.2270.0.tar.gz [http://www.microsoft.com/en-au/download/details.aspx?id=36437]
*sqlncli-11.0.1790.0.tar.gz [http://www.microsoft.com/en-au/download/details.aspx?id=28160]

To get unixODBC-2.3.2 sources:
==============================
http://www.unixodbc.org/ - this site is extremely slow/broken so grab them from here: 
[http://www.linuxfromscratch.org/blfs/view/svn/general/unixodbc.html]

To get PHP sources and build pdo.so & pdo_odbc.so for use with unixODBC:
===============================================================
git clone https://github.com/php/php-src

use git checkout to switch to the correct tag for your php version, in my case it was:
```
*pdo.so:
git checkout php-5.4.19
cd ext/pdo
phpize
./configure
make
sudo make install #installs into your php extensions dir

*pdo_odbc.so:
cd ext/pdo_odbc
phpize
./configure --with-pdo-odbc=unixODBC
make
sudo make install #installs into your php extensions dir
```

Useful blog entry here:http://blog.afoolishmanifesto.com/archives/1855

Building the UnixODBC 2.3.2
===========================
```
cd unixodbc-2.3.2

vi configure and change LB_VERSION from 2.0.0. to 1.0.0

./configure --disable-gui --with-pdo-odbc=unixODBC --enable-stats=no --enable-iconv --with-iconv-char-enc=UTF8 --with-iconv-ucode-enc=UTF16LE --prefix=/usr --sysconfdir=/etc/

make

sudo make install
```

Create symlinks as these drivers are designed for RH 5/6
==================================================
```
sudo ln -s /lib/x86_64-linux-gnu/libssl.so.1.0.0 ./libssl.so.10
sudo ln -s /lib/x86_64-linux-gnu/libcrypto.so.1.0.0 ./libcrypto.so.10
```

MS Native Command Line Client driver 
=====================================

Extract the sqlncli package.

```
cd sqlncli*

sudo bash ./install.sh install --force   --accept-license

```

MS ODBC Native driver (not required for access from PHP but needed for isql access)
===================================================================================

Extract the msodbcsql package.

```
cd msodbcsql*

sudo bash ./install.sh install --force   --accept-license
```

Phew now setup your odbc.ini & odbcinst.ini files:

MS Linux Driver ODBC configuration
==================================

PDO ODBC requires an ODBC driver, and you need to refer to MS ODBC driver in the connection string

odbc.ini
```
[SQLServerNCLI]
Description = SQL Server NCLI
Driver = {SQLServerNativeClient11.0}
Database = YourDatabaseName
ServerName = YourServerHost
TDS_Version = 8.0


[SQLServerMSODBC]
Description = MS ODBC SQL Server Linux
#Driver = SQL Server Native Client 11.0
Driver = {ODBCDriver11forSQLServer}
Database = YourDatabaseName
ServerName = YourServerHost
TDS_Version = 8.0
```

odbcinst.ini
```
[ODBC]
Trace                   = yes
TraceFile               = /tmp/odbctracefile.log

[SQLServerNativeClient11.0]
Description             = Microsoft SQL Server ODBC Driver V1.0 for Linux
Driver                  = /opt/microsoft/sqlncli/lib64/libsqlncli-11.0.so.1790.0
Threading               = 1
UsageCount              = 3

[ODBCDriver11forSQLServer]
Description             = Microsoft ODBC Driver 11 for SQL Server
#Driver                 = /opt/microsoft/msodbcsql/lib64/libmsodbcsql-11.0.so.2270.0
Driver                  = /usr/lib/php5/20100525/libmsodbcsql.so
Threading               = 1
UsageCount              = 3
```

Check unixodbc version
======================
odbcinst -j

Check locate unixODBC is looking for config files
=================================================
odbc_config --libs --longodbcversion --odbcini

You may have to symlink to your actual odbc.ini and odbcinst.ini files!

Test iSQL
=========
isql -v SQLServerNCLI <User Id> <password>


Setup PHP php.ini
=================

add the following entries:
```
extension=pdo.so
extension=pdo_odbc.so
```

Make sure you have the following files in the active php extensions dir:
```
pdo_odbc.so
libmsodbcsql.so
pdo.so
```

You may have to do this twice once for apache php.ini and once for CLI php.ini

Type php -i | grep php.ini to find your CLI php.ini

To find out apache php configuration you need to add
an 'info.php' file to your root web project and goto http://<your web server url>/info.php to find out useful information on php configuration under
apache/web server. Google is your friend!


Silex Doctrine setup
====================
TODO




FYI - Doctrine Test Suite
=========================

Doctrine2's test suite does not allow you to add database drivers on the fly. If you want to test this package, modify `Doctrine/DBAL/Driver/DriverManager::$_driverMap` as follows:

```php
final class DriverManager
{
    private static $_driverMap = array(
		/* ... snip ... */
        'pdo_dblib' => 'Doctrine\DBAL\Driver\PDODblib\Driver',
    );
}
```

FYI - Generating Entities from database
=======================================

It's possible, but not easy. Here's what I did:

- Map any non-compatible column types to string
- Hack the Doctrine core to skip any tables without primary keys

