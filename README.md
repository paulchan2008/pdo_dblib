# pdo_dblib
linux版本：64位CentOS 6.4<br>
Nginx版本：nginx1.8.0<br>
php版本：php5.6<br>
Sqlserver版本：2008<br>
FreeTDS版本：0.95<br>
 
1.首先需要编译安装FreeTDS<br>
说明：一定要从官网下载最新的版本FreeTDS-0.95(最新版1.0了) ftp://ftp.freetds.org/pub/freetds/stable/freetds-patched.tar.gz<br>
 
$wget ftp://ftp.freetds.org/pub/freetds/stable/freetds-patched.tar.gz<br>
$tar -zxvf freetds-patched.tar.gz<br>
$cd freetds-0.95（解压之后的目录我的是1.0）<br>
需要注意的就是这里的--with-tdsver=7.3，这个非常重要，你需要根据你的数据库版本选择正确的配置项，由于现在大多是SQLserve2008所以需要选择7.3.<br>
关于这个问题网上有的说是7.1，也有的说是7.2，甚至有的说是8.0，可以看文末参考帖子，不过那些说的都有问题。<br>
造成这个配置项混乱的根源是很多人用的是FreeTDS-0.91，经过我的测试FreeTDS-0.91只支持7.1，如果是7.2以上配置那么通通会变为5.0。<br><br>

其实参考官网的文档就知道这个问题了，不过由于很多人下载了旧版FreeTDS-0.91，即使设置为--with-tdsver=7.2以上也没有用。<br><br>

总结：FreeTDS-0.91只支持7.1，其余都会默认为5.0。只有最新的FreeTDS-0.95，也就是对Sqlserver2008的最佳配置。<br><br>

$ ./configure --prefix=/usr/local/freetds --with-tdsver=7.3 --enable-msdblib<br>
$ make && make install<br><br><br>
 
 
配置FreeTDS<br>
$ cd ../<br>
$ echo "/usr/local/freetds/lib/" > /etc/ld.so.conf.d/freetds.conf<br>
$ ldconfig<br><br>
 
验证FreeTDS版本<br>
这一步非常重要，通过才可以继续，不然后面的步骤都是无意义的。<br>
首先看看版本信息<br>
$ /usr/local/freetds/bin/tsql -C<br><br><br>


测试数据库是否联通<br>
$ /usr/local/freetds/bin/tsql -H 数据库服务器IP  -p 端口号 -U 用户名 -P 密码<br><br><br><br>

 
关于freetds/etc/freetds.conf配置项（非必要）<br>
很多其他帖子写了需要配置/usr/local/freetds/etc/freetds.conf，其实这个不需要配置。如果配置也可以，配置了PHP就可以调用这个配置项，否则需要PHP代码里指定数据库服务器信息即可。<br>
另外需要注意的是/usr/local/freetds/etc/下的freetds.conf不同于前面/usr/local/freetds/lib/那个freetds.conf。<br>
如果配置了这里，那么PHP页面就可以使用这里的配置，不然PHP页面指定一样可以。<br><br>

如果你想使用配置项，只要修改[egServer70]即可：<br>
 A typical Microsoft server  <br>
[egServer70]  <br>
    host = ntmachine.domain.com  <br>
    port = 1433  <br>
    tds version = 7.0  <br><br>
	
 [cpp] view plain copy<br>
[egServer70]  <br>
    host = 192.168.1.235 这个是数据库服务器IP  <br>
    port = 1433  <br>
    tds version = 7.1  <br>
其他都不用动，关于[egServer70]的名字也是随意的，这个就是给PHP调用的，和PHP代码里一致即可。<br><br>
 
3.添加PHP扩展pdo_dblib<br>
$tar -zxf PDO_DBLIB-1.0.tgz && cd PDO_DBLIB-1.0<br>
$/usr/local/php/bin/phpize<br>
$./configure --with-php-config=/usr/local/php/bin/php-config --with-pdo-dblib=/usr/local/freetds<br><br>

##注意：如果编译不成功，比如有错误：Cannot find FreeTDS in known installation directories<br>
其实是PHP检测其安装目录的时候有些问题，检查依据是两个已经不用的文件，创建两个空文件就OK<br>
$touch /usr/local/freetds/include/tds.h<br>
$touch /usr/local/freetds/lib/libtds.a<br>
然后在重新编译一次PDO_DBLIB<br><br>

注：make时候出现一次错误：/usr/src/PDO_DBLIB-1.0/pdo_dblib.c:37:1: error: unknown type name ‘function_entry’<br>
将‘function_entry‘改为'zend_function_entry'可以解决。可能是由于php版本不同。<br><br>
 
(3).在php.ini配置文件中增加.so<br>
$ cd /usr/local/php/lib下的php.ini<br>
增加：<br>
extension ="pdo_dblib.so"  <br><br>

如果你只需要上述2种扩展之一，自然只要新增其中一个的.so扩展到php.ini即可。<br>
(4).重启PHP FastCGI<br>
$ killall php-fpm<br>
$ /etc/init.d/php-fpm<br><br>
 
如果没有正确生成扩展是不能重启php-fpm的。<br>
这时候在phpinfo里就可以看到扩展添加成功的信息了。<br><br><br>


4.使用PHP调用SQLserver<br>
<?php  <br>
  try {  <br>
    $hostname = "192.168.23.12";  <br>
    $port = 1433;  <br>
    $dbname = "test";  <br>
    $username = "sa";  <br>
    $pw = "123456";  <br>
    $dbh = new PDO ("dblib:host=$hostname:$port;dbname=$dbname","$username","$pw");  <br>
  } catch (PDOException $e) {  <br>
    echo "Failed to get DB handle: " . $e->getMessage() . "\n";  <br>
    exit;  <br>
  }  <br>
  $stmt = $dbh->prepare("SELECT top 5 * FROM test");  <br>
  $stmt->execute();  <br>
  while ($row = $stmt->fetch()) {  <br>
    print_r($row);  <br>
  }  <br>
