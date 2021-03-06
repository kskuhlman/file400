# file400

IBM i record level access for Python 3. (beta)

## Background
We have a lot of python modules written for iseries python 2.7 that uses the file400 module.

When I planned to migrate to Python 3.6 pase for i, my biggest concern was the database access, and the only option was the ibm-db-dbi module.

We have an Ubuntu box with postgres, that runs the same python programs.  
It has a layer on top of psycopg2 that emulates file400.  
Db2 database changes is copied to the Ubuntu postgres database using journals, and when the IBM i is unavailable during full backup a few hours every week, this Ubunto box takes over.
This has been running for several years without any performance problems with the file400 layer on the Ubuntu box. 

I decided to use this same layer on top of ibm-db-dbi on the IBM i, but the performance hit was to big.

I then converted the \_db2 module from iseries Python 2.7 and used that instead of ibm-db-dbi, but it didn't make big difference.

Now I have rewritten the file400 module, and with this the performance is at least as good as on Python 2.7.

## Installation
You will have to install an ile service program called reclevacc and a python c extention module named file400.so  
Download file400.so.zip and reclevacc.savf.zip  
Unpack the zip files on your pc.  
Transfer file400.so to site-packages (/QOpenSys/pkgs/lib/python3.6/site-packages)  
Create a lib called python3  
Transfer reclevacc.savf to python3 (/QSYS\.LIB/PYTHON3.LIB)  
Run RSTOBJ OBJ(RECLEVACC) SAVLIB(PYTHON3) DEV(\*SAVF) SAVF(PYTHON3/RECLEVACC)  
  
Use it as in python2.7 (a few of the functions has been removed)  
```
from file400 import File400  
f = File400('myfile', 'r')  
f.posb((first_key, second_key))  
while f.readne():  
    print(f._first1)  
```

The db2 module is also available, though it's probably better to use ibm-db-dbi.  
Install as with file400  
Example on how to use it.  
```  
import _db2 as db2  
con = db2.connect(servermode=False, autocommit=True, sysnaming=True)  
for row in con.execute("select col1, col2 from myfile"):  
    col1, col2 = row  
    print(col1, col2)  
```
  
## In case you would like to do the compilation your self.
\_db2.c  
Compile  
gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -O2 -g -maix64 -I/QOpenSys/pkgs/include -I/python3/source/include -I/QOpenSys/pkgs/include/python3.6m -c \_db2.c -o build/temp.os400-powerpc64-3.6/\_db2.o  
link  
gcc -pthread -shared /QOpenSys/pkgs/lib/python3.6/config-3.6m/python.exp /QOpenSys/lib/libdb400.a build/temp.os400-powerpc64-3.6/\_db2.o -o build/lib.os400-powerpc64-3.6/\_db2.so  

file400.c  
Compile  
gcc -pthread -Wno-unused-result -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -O2 -g -maix64 -I/QOpenSys/pkgs/include -I/python3/source/include -I/QOpenSys/pkgs/include/python3.6m -c file400.c -o build/temp.os400-powerpc64-3.6/file400.o  
link  
gcc -pthread -shared /QOpenSys/pkgs/lib/python3.6/config-3.6m/python.exp /QOpenSys/lib/libiconv.a build/temp.os400-powerpc64-3.6/file400.o -o build/lib.os400-powerpc64-3.6/file400.so  
  
RECLEVACC ILE program  
CRTCMOD MODULE(PYTHON3/RECLEVACC) SRCSTMF('/python3/source/reclevacc.c')
SYSIFCOPT(*IFS64IO) LOCALETYPE(*LOCALEUTF)
TERASPACE(*YES *TSIFC) STGMDL(*TERASPACE) DTAMDL(*P128)  

CRTSRVPGM SRVPGM(PYTHON3/RECLEVACC) EXPORT(*ALL) STGMDL(*TERASPACE)  
