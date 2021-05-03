---
layout: article
tags: Python
title: Install Scrapy
mathjax: true
key: Linux
---

## Install Scrapy
```
#!/bin/bash

yum install -y python3 || yum install -y platform-python || yum install -y python2

if [ ! -e /usr/bin/python ];then
         if [ -e "/usr/bin/python3" ];then
                 ln -s /usr/bin/python3 /usr/bin/python
         elif [ -e "/usr/libexec/platform-python" ];then
                 ln -s /usr/libexec/platform-python /usr/bin/python
         elif [ -e "/usr/bin/python2" ];then
                 ln -s /usr/bin/python2 /usr/bin/python
         fi
fi

python -V >  python_version.txt 2>&1
python_version=$(cat python_version.txt|grep "Python [0-9]"|awk '{print $2}'|awk -F'.' '{print $1}')

if [ $python_version -eq 3 ];then
    yum install -y python3-devel
    yum install -y platform-python-devel
    yum install -y python3-pip
    ln -s /usr/bin/pip3 /usr/bin/pip

    yum install -y python3-PyMySQL
elif [ $python_version -eq 2 ];then
    if [ ! -e /usr/bin/pip ];then
    	yum install -y python2-devel
    	wget https://bootstrap.pypa.io/get-pip.py
    	python get-pip.py
    	ln -s /usr/bin/pip2 /usr/bin/pip

	yum install -y MySQL-python
    fi
fi

# check again because sometimes /usr/bin/python is deleted
if [ ! -e /usr/bin/python ];then
         if [ -e "/usr/bin/python3" ];then
                 ln -s /usr/bin/python3 /usr/bin/python
         elif [ -e "/usr/libexec/platform-python" ];then
                 ln -s /usr/libexec/platform-python /usr/bin/python
         elif [ -e "/usr/bin/python2" ];then
                 ln -s /usr/bin/python2 /usr/bin/python
         fi
fi

# install virtualenv,virtualenvwrapper
pip install virtualenv virtualenvwrapper
s_dir=$(find / -name virtualenvwrapper.sh)
if ! grep WORKON_HOME ~/.bashrc;then
cat >> ~/.bashrc <<-EOF
export WORKON_HOME=$HOME/.virtualenvs
export PROJECT_HOME=$HOME/virtualprj
source $s_dir
EOF
fi
mkdir $HOME/virtualprj
source ~/.bashrc

# create virtualenv 'VENV'
mkvirtualenv VENV
# install scrapy in 'VENV'
pip install twisted
pip install pyopenssl
pip install scrapy
deactivate

```
