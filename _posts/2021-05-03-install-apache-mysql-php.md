---
layout: article
tags: Web
title: Install Apache+PHP+MySql 
mathjax: true
key: Linux
---

## Installa Apache
```
#!/bin/bash

yum install -y httpd

export username=liliang
export groupname=liliang

# add apache user,group
groupadd liliang
useradd liliang -g liliang
sed -i "s/User apache/User ${username}/" /etc/httpd/conf/httpd.conf
sed -i "s/User apache/User ${groupname}/" /etc/httpd/conf/httpd.conf 

echo ">>>start copy web files"
[ -e /var/www/html ] || mkdir -p /var/www/html;
cp -r /mnt/tests/kernel/networking/tools/job_scheduler /var/www/html
mkdir /var/www/html/job_scheduler/logs
mkdir /var/www/html/job_scheduler/tests
mkdir /var/www/html/job_scheduler/ci/umb
mkdir /var/www/html/job_scheduler/ci/umb/messages
touch /var/www/html/job_scheduler/logs/SystemManager.log
touch /var/www/html/job_scheduler/logs/Scheduler.log
touch /var/www/html/job_scheduler/logs/TestManager.log
touch /var/www/html/job_scheduler/ci/log.txt
touch /var/www/html/job_scheduler/ci/umb/log
chown -R ${username}:${groupname} /var/www/html/job_scheduler
mkdir /var/www/html/job_scheduler/job_statistic/images
chmod 1777 /var/www/html/job_scheduler/job_statistic/images

# install bkr lib
if ! grep "sslverify=false" /etc/yum.conf;then
	sed -i '$a\sslverify=false' /etc/yum.conf
fi
bkr --version > /dev/null || {
	#wget --no-check-certificate -O /etc/yum.repos.d/beaker-client.repo http://download.lab.bos.redhat.com/beakerrepos/beaker-client-RedHatEnterpriseLinux.repo
	wget --no-check-certificate -O /etc/yum.repos.d/beaker-client.repo https://download.eng.bos.redhat.com/beakerrepos/beaker-client-RedHatEnterpriseLinux.repo
        yum install -y rhts-test-env beakerlib rhts-devel rhts-python beakerlib-redhat beaker-client beaker-redhat
}

# install kerberos client
yum install -y krb5-workstation
rm -f /etc/krb5.conf
cp krb5.conf /etc/

# install scrapy
echo ">>>start install scrapy"
source install_scrapy.sh

echo ">>>start install dependence"
yum install -y gcc
yum install -y redhat-rpm-config

# install googlesheet api
pip show google-api-python-client &> /dev/null || pip install google-api-python-client
pip show gspread                  &> /dev/null || pip install gspread
pip show oauth2client             &> /dev/null || pip install oauth2client

# disable selinux
setenforce 0

echo ">>>start install mysql"
source install_mysql.sh
echo ">>>start install php"
source install_php.sh

# install python package
pip install pyyaml
pip install python-gitlab
pip install colorama
pip install python-bugzilla

```

## Install PHP
```
#!/bin/bash

# install php
yum install -y php
php_version=$(yum info php|grep Version|awk '{print $NF}'|awk -F'.' '{print $1}')
if ! grep -q "/usr/lib64/httpd/modules/libphp${php_version}.so" /etc/httpd/conf/httpd.conf;then
	cat >> /etc/httpd/conf/httpd.conf <<-EOF
	LoadModule php${php_version}_module /usr/lib64/httpd/modules/libphp${php_version}.so
	<FilesMatch \.php$>
	    SetHandler application/x-httpd-php
	</FilesMatch>
EOF
fi

if ! grep -q "LoadModule mpm_prefork_module modules/mod_mpm_prefork.so" /etc/httpd/conf.modules.d/00-mpm.conf;then
	cat >> /etc/httpd/conf.modules.d/00-mpm.conf <<-EOF
	LoadModule mpm_prefork_module modules/mod_mpm_prefork.so
EOF
fi

# install php-mysql
yum install -y php-mysql.x86_64

# pChart depends on gd
yum install -y php-gd
if ! grep -q "extension=gd.so" /etc/php.ini;then
	line_num=$(grep -n "Dynamic Extensions" /etc/php.ini|awk -F":" '{print $1}')
	line_num=$((line_num+1))
	sed -i ${line_num}a\extension=gd.so /etc/php.ini
fi

# download pChart which will be used to draw picture
wget http://netqe-bj.usersys.redhat.com/share/liali/pChart.tar.gz -O /var/www/html/job_scheduler/job_statistic/pChart.tar.gz
pushd /var/www/html/job_scheduler/job_statistic
tar -zxv -f pChart.tar.gz
popd

# modify default session directory permission
chmod 777 /var/lib/php/session

systemctl restart httpd

```

## Install MySql
```
#!/bin/bash

if ! mysql --version;then
	wget http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
	rpm -ivh mysql-community-release-el7-5.noarch.rpm
	yum update -y
	yum install mysql-server -y
	
	chown mysql:mysql -R /var/lib/mysql
	
	mysqld --initialize
fi

systemctl start mysqld

# change root password
#mysqladmin -uroot -p password root
mysqladmin -uroot password root >/dev/null 2>&1

username="username"
password="yourpwd"
database=work
table1=jobs
table2=user
table3=system_status

# create table and user
mysql -uroot -proot <<-EOF
CREATE DATABASE ${database};
USE ${database};
CREATE TABLE \`${table1}\` (
  \`id\` varchar(20) NOT NULL,
  \`driver\` varchar(50) DEFAULT NULL,
  \`model\` varchar(100) DEFAULT NULL,
  \`speed\` int(11) DEFAULT NULL,
  \`cases\` varchar(100) DEFAULT NULL,
  \`testside\` varchar(20) DEFAULT NULL,
  \`status\` varchar(20) DEFAULT NULL,
  \`duration\` int(11) DEFAULT NULL,
  \`distro\` varchar(50) DEFAULT NULL,
  \`whiteboard\` varchar(100) DEFAULT NULL,
  \`arch\` varchar(20) DEFAULT NULL,
  PRIMARY KEY (\`id\`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
CREATE TABLE \`${table2}\` (
  \`id\` int(11) NOT NULL AUTO_INCREMENT,
  \`name\` varchar(30) NOT NULL,
  \`password\` varchar(200) NOT NULL,
  \`priority\` int(11) DEFAULT '0',
  PRIMARY KEY (\`id\`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
CREATE TABLE \`${table3}\` (
  \`id\` int(11) NOT NULL AUTO_INCREMENT,
  \`sysname\` varchar(100),
  \`status\` varchar(100),
  \`user\` varchar(200),
  \`testname\` varchar(200),
  \`time\` varchar(40),
  PRIMARY KEY (\`id\`)
) ENGINE=InnoDB AUTO_INCREMENT=4 DEFAULT CHARSET=utf8;
grant all on ${database}.${table1} to '${username}'@'%' identified by '${password}';
grant all on ${database}.${table2} to '${username}'@'%' identified by '${password}';
grant all on ${database}.${table3} to '${username}'@'%' identified by '${password}';
EOF

```
