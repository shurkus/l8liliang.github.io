---
layout: article
tags: Python
title: Bugzilla API 
mathjax: true
key: Linux
---

## install
```
pip install python-bugzilla
```

[github](https://github.com/python-bugzilla/python-bugzilla)

## API
```
# api address
BZ = 'https://bugzilla.xxx.com'
BZ_API = 'https://bugzilla.xxx.com/xmlrpc.cgi'

# create bugzilla
bzapi = bugzilla.Bugzilla(BZ_API,user=user,password=pwd)
#bzapi.login(user,pwd)

# query bug
query = bzapi.build_query(bug_id=bugs)
query['include_fields'] = ['id', 'status', 'qa_contact', 'summary', 'cf_verified']
query['include_fields'].extend(fields)
bzs = bzapi.query(query)

# get qa_contact
bzs[0].qa_contact

# get cmoments
bz = bzs[0]
comments=bz.getcomments()
for c in comments:
    print(c["text"])

```

## script
```
#!/bin/env python

import os
import re
import sys
import json
import yaml
import gitlab
import inspect
import argparse
import requests
import bugzilla
import threading
import subprocess
from colorama import Fore
from bs4 import BeautifulSoup
from datetime import datetime

BZ = 'https://bugzilla.xxx.com'
BZ_API = 'https://bugzilla.xxx.com/xmlrpc.cgi'

def printLog(*messages):
        print_log("log",*messages)

def getBugsInfo(bugs, fields=[]):
    printLog("getBugsInfo "+str(bugs))
    try:
        user=subprocess.check_output("cat ../cfg|grep BZUser|awk -F '=' '{print $NF}'",shell=True).rstrip("\n")
        pwd=subprocess.check_output("cat ../cfg|grep BZPWD|awk -F '=' '{print $NF}'",shell=True).rstrip("\n")
        bzapi = bugzilla.Bugzilla(BZ_API,user=user,password=pwd)
        #bzapi.login(user,pwd)
        query = bzapi.build_query(bug_id=bugs)
        query['include_fields'] = ['id', 'status', 'qa_contact', 'summary', 'cf_verified']
        query['include_fields'].extend(fields)
        bzs = bzapi.query(query)
        return bzs
    except Exception as e:
        printLog("Exception occur: "+str(e))
        return []

def getBugQA(bug):
    bzInfo = getBugsInfo([bug])
    if len(bzInfo):
        return bzInfo[0].qa_contact
    else:
        return None

def isNetworkQEBug(bug):
    QA=getBugQA(bug)
    if QA and QA in ["xxx","xxx"]:
        return True
    else:
        return False

def bugCommentMatchMrURL(bug,mrURL):
    bzInfo = getBugsInfo([bug])
    if len(bzInfo):
        bz = bzInfo[0]
        comments=bz.getcomments()
        for c in comments:
            #print(c["text"])
            if re.search(mrURL,c["text"],re.M|re.I):
                printLog("pipeline message " + mrURL + " has been added to bz " + bug)
                return True
        printLog("pipeline message " + mrURL + " has not been added to bz " + bug)
        return False
    else:
        return False

# If Bug's QA_Contact is Network-QE or it's member and pipeline result has been added to this Bug,
# the we say this Bug is testable for our.
def bugTestable(bug,mrURL):
    bzInfo = getBugsInfo([bug])
    if len(bzInfo):
        bz = bzInfo[0]
        QA=bz.qa_contact
        if QA and QA in ["xxx","xxx"]:
            return True
        return False
    else:
        return False

# If any bug is testable, return true
# else return false.
def anyBugTestable(bugs,mrURL):
    for bug in bugs:
        if bugTestable(bug,mrURL):
            return True
    return False
    
def main():
    pass

if __name__ == "__main__":
    main()

```
