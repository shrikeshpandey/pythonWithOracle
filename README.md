# pythonWithOracle
Using Python with Oracle
This page discusses using Python with Oracle.

The page is based on the cx_oracle Python extension module. It was developed on a VM running Oracle Enterprise Linux 6U4 runnng Oracle 11.2.0.4 and Python 2.6.6.

Installation and Configuration
Assuming Python 2.6 has already been installed, the first job is to install the cx_oracle module. At the time of writing the current version was still 5.1.3 for which binaries are available for both Windows and Linux. For other platforms the sources can be downloaded and compiled.

I downloaded version 5.1.2 from http://sourceforge.net/projects/cx-oracle/files where the following RPM is available. This was the closest match to my environment

cx_Oracle-5.1.2-11g-py26-1.x86_64.rpm
Installation is standard. As the root user:

rpm -ivh cx_Oracle-5.1.2-11g-py26-1.x86_64.rpm
My VM already contained Oracle 11.2.0.4 Enterprise Edition with a database called "TARGET". I set the following environment variables in .bash_profile

export ORACLE_HOME=/u01/app/oracle/product/11.2.0/db_1
export LD_LIBRARY_PATH=$ORACLE_HOME/lib
export ORACLE_SID=TARGET
I rarely need to set $LD_LIBRARY_PATH, but in this case it was necessary to avoid raising the following error when the cx_Oracle is imported:

ImportError: libclntsh.so.11.1: cannot open shared object file: No such file or directory
Preparation
cx_oracle supports various connection methods. If possible I like to use a TNS alias with the Oracle client as the alias can be configured to use connection load balancing, failover etc. For this post my VM was called vm16, so the entry in $ORACLE_HOME/network/admin/tnsnames.ora was:

TARGET=
(DESCRIPTION=
  (ADDRESS=(HOST=vm16)(PROTOCOL=TCP)(PORT=1521))
  (CONNECT_DATA=
    (SERVER=DEDICATED)
    (SERVICE_NAME=TARGET)
  )
)
Example Script
The following is a complete script that connects to the database, executes a simple SELECT aggregate statement and returns the result

#!/usr/bin/python
# Example of fetchone

import sys
import cx_Oracle

def printf (format,*args):
  sys.stdout.write (format % args)

def printException (exception):
  error, = exception.args
  printf ("Error code = %s\n",error.code);
  printf ("Error message = %s\n",error.message);

username = 'scott'
password = 'tiger'
databaseName = "TARGET"

try:
  connection = cx_Oracle.connect (username,password,databaseName)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to connect to %s\n',databaseName)
  printException (exception)
  exit (1)

cursor = connection.cursor ()

try:
  cursor.execute ('SELECT COUNT(*) FROM emp')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to select from EMP\n')
  printException (exception)
  exit (1)

count = cursor.fetchone ()[0]
printf ('Count = %d\n',count)

cursor.close ()

connection.close ()

exit (0)
The above code is discussed in the following sections

printf statement
Python does not include a printf statement. However the following library function provides similar functionality.

import sys

def printf (format,*args):
  sys.stdout.write (format % args)
This can be called using the familiar C syntax. For example:

printf ("Str %s Int %d\n",s,i)
printException function
I have also created a function to print details of any exceptions raised when accessing the database:

def printException (exception):
  error, = exception.args
  printf ("Error code = %s\n",error.code);
  printf ("Error message = %s\n",error.message);
Connection
The connection requires a username, password and TNS alias:

username = 'scott'
password = 'tiger'
databaseName = "TARGET"
Experience has demonstrated that it is good practice to execute cx_oracle methods that access the database within a try..except structure in order to catch and report any exceptions that they might throw. The printException function prints more detail

For the connection I have used the following code:

try:
  connection = cx_Oracle.connect (username,password,databaseName)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to connect to %s\n',databaseName)
  printException (exception)
  exit (1)
The connection should be closed when no longer required using:

connection.close ()
It is a good idea to develop the exception handling code at the outset, then most can be reused for other cx_oracle method calls. Remember to rollback any uncommitted transactions, close cursors and close any connections when the exception is raised.

Alternatively consider using a finally clause in the try statement. The finally clause of a try statement is executed irrespective of any exceptions that may be raised.

Cursor Management
In the previous section we created a connection. We now need to create a cursor as follows:

cursor = connection.cursor ()
The cursor should be closed when no longer required using:

cursor.close ()
The Cursor class has a number of powerful methods that allow statements and PL/SQL blocks to be prepared and executed

Execute method
The cursor.execute method is the easiest way to execute simple statements including DDL.

The simple SELECT statement is an aggregate returning a single column in a single row:

try:
  cursor.execute ('SELECT COUNT(*) FROM emp')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to select from EMP\n')
  printException (exception)
  exit (1)
We can check the result as follows:

count = cursor.fetchone ()[0]
printf ('Count = %d\n',count)
In this case the fetchone method is more appropriate when we are expecting a single row.

Alternatively we could have used the Cursor.fetchall method which fetches all rows returned by the statement:

count = cursor.fetchall ()[0][0]
printf ("Count = %d\n",count)
SELECT Statements
This section examines more complex examples of the SELECT statement. The next example returns multiple rows and columns:

sql = """\\
  SELECT empno,ename,sal FROM emp
  WHERE sal > 2000
  ORDER BY sal DESC"""

try:
  cursor.execute (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to select from EMP\n')
  printException (exception)

result = cursor.fetchall ()
for row in result:
  printf (" %s;%s;%d\n",row[0],row[1],row[2])
I have separated the SQL statement from the call to cursor.execute to improve readability. The triple quotes at the start and end of the SQL statement allow it to be written across multiple lines. This is particularly important for longer SQL statements where indentation is necessary. The backslash after the opening triple quote suppresses a newline character from being appended before the first line. A newline character will be appended to the remaining lines until the closing triple quote.

The results are returned using the Cursor.fetchall�method which returns a 2-dimensional array. All results are returned by the single call to fetchall and are stored in process memory. Obviously this works when there are only 14 employees - it would not be appropriate for larger result sets. In this case I am iterating through the results, printing them one line at a time. This works but is not particularly readable.

An alternative version of the code to handle the call to cursor.fetchall() follows:

for empno, ename, sal in cursor.fetchall ():
  printf (" %s;%s;%d\n",empno, ename, sal)
This version is more readable and requires less process memory

Bind Variables
So far our statements have only used literal values. These are not particularly scalable in high volume workloads so most applications use bind variables to reduce the parsing overhead.

The following example shows a statement with a couple of bind variables in the predicates

sql = """\\
  SELECT empno,ename,sal FROM emp
  WHERE job = :job
  AND deptno = :dept
  ORDER BY sal DESC"""

try:
  cursor.execute (sql,job = 'SALESMAN',dept = 30)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to select from EMP\n')
  printException (exception)

for empno, ename, sal in cursor.fetchall ():
  printf (" %s;%s;%d\n",empno, ename, sal)
In the above example the call to cursor.execute includes the sql statement and values for the bind variables 'job' and 'dept'. Note from the second bind variable that the bind variable name does not have to be the same as the column value.

Prepare Statements
Whilst bind variables reduce parsing to a limited extent, they cannot eliminate it completely. To minimize parsing it is best to assign a prepared statement to a dedicated cursor. It is not then necessary to parse the statement again until the cursor is closed.

In the following code, the bind variable example from the previous section has been modified to use a separate prepare call.

sql = """\\
  SELECT empno,ename,sal FROM emp
  WHERE job = :job
  AND deptno = :dept
  ORDER BY sal DESC"""

try:
  cursor.prepare (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to prepare cursor')
  printException (exception)
  exit (1)

try:
  cursor.execute (None,job = 'SALESMAN',dept = 30)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor')
  printException (exception)
  exit (1)

for empno, ename, sal in cursor.fetchall ():
  printf (" %s;%s;%d\n",empno, ename, sal)
In the above example the statement is prepared using the cursor.prepare call and is then executed by the cursor.execute call.

Note that the first parameter of the cursor.execute call is "None". This specifies that the cursor should use the existing prepared statement instead of parsing a new SQL statement.

Alternatively the cursor.execute statement can take cursor.statement as the first parameter. For example:

try:
  cursor.execute (cursor.statement,job = 'SALESMAN',dept = 30)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor')
  printException (exception)
  exit (1)
Cursor.statement returns the last statement parsed by the cursor.

INSERT Statements
This section discusses INSERT statements. The first example uses bind variables to insert a new row into the DEPT table.

sql = "INSERT INTO dept (deptno, dname, loc) VALUES (:deptno, :dname, :loc)"

try:
  cursor.execute (sql,deptno=50, dname='MARKETING', loc='LONDON')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to insert row')
  printException (exception)
  exit (1)

cursor.close ()

connection.commit ()
In this example the values are supplied as parameters for the cursor.execute() method.

Note the commit statement at the end of the transaction

The following example uses a dictionary instead of the parameter list:

sql = "INSERT INTO dept (deptno, dname, loc) VALUES (:deptno, :dname, :loc)"

try:
  cursor.execute (sql,{ 'deptno':50, 'dname': 'MARKETING', 'loc': 'LONDON'})
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to insert row')
  printException (exception)
  exit (1)
In the above example a dictionary is passed to the cursor.execute () method. The dictionary contains name-value pairs for the column values to be inserted into the new row.

The next example demonstrates the use of a prepared INSERT statement:

sql = "INSERT INTO dept (deptno, dname, loc) VALUES (:deptno, :dname, :loc)"

try:
  cursor.prepare (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to prepare cursor')
  printException (exception)
  exit (1)

try:
  cursor.execute (None,deptno=50, dname='MARKETING', loc='LONDON')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to insert row')
  printException (exception)
  exit (1)
As with the SELECT statement example, the cursor.execute method can use cursor.statement() instead of None, as shown below

  cursor.execute (cursor.statement,deptno=50, dname='MARKETING', loc='LONDON')
UPDATE Statement
This section discusses UPDATE statements. The first example uses bind variables.

sql = "UPDATE dept SET loc = :loc WHERE deptno = :deptno"

try:
  cursor.execute (sql,deptno = 50,loc = 'LONDON')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor\n')
  printException (exception)
  exit (1)
In the above example, bind variable values are specified in the arguments of the call to cursor.execute() The next example uses a prepare statement

sql = "UPDATE dept SET loc = :loc WHERE deptno = :deptno"

try:
  cursor.prepare (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to prepare cursor\n')
  printException (exception)
  exit (1)

try:
  cursor.execute (None,deptno = 50,loc = 'LONDON')
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor\n')
  printException (exception)
  exit (1)
As with the INSERT statement example, the Cursor.execute method can use Cursor.statement instead of None, as shown below:

  cursor.execute (cursor.statement,deptno = 50,loc = 'LONDON')
DELETE Statement
This section discusses DELETE statements. The following is an example of a delete that uses a bind variable:

sql = "DELETE FROM dept WHERE deptno = :deptno"

try:
  cursor.prepare (sql)
except:
  printf ('Failed to prepare cursor\n')
  exit (1)

try:
  cursor.execute (None,deptno = 50)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor\n')
  printException (exception)
  exit (1)
As with previous examples, Cursor.execute can use Cursor.statement instead of None, as shown below:

  cursor.execute (cursor.statement,deptno = 50)
Array Operations
So far we have concentrated on statements which only make a single visit to the database for each execution. However, if we want our application to scale, at some stage we will need to consider batch operations to reduce the number of round trips from the client to the server. In this section we will examine batching techniques for SELECT statements and also INSERT, UPDATE and DELETE statements.

SELECT statements
For SELECT statements we can control the number of statements retrieved for each fetch using Cursor.arraysize.

The following statement sets the array size to 4 and then queries the EMP table:

cursor = connection.cursor ()

sql = """\\
  SELECT empno, ename, job
  FROM emp
  ORDER BY ename"""

# set array size to 4
cursor.arraysize = 4

try:
  cursor.execute (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to select from EMP\n')
  printException (exception)
  exit (1)

while True:
   rows = cursor.fetchmany()
   if rows == []:
        break;
   printf ("Fetched %d rows\n", len(rows))
   for empno,ename,job in rows:
     printf (" %s;%s;%s\n",empno,ename,job)
The above example includes the creation of the cursor using connection.cursor() as the cursor must exist before the array size can be modified.

In this example we are using the Cursor.fetchmany method to fetch the rows. Every fetch with the exception of the last one will return four rows to the application.

The output of the above script is:

Fetched 4 rows
 7876;ADAMS;CLERK
 7499;ALLEN;SALESMAN
 7698;BLAKE;MANAGER
 7782;CLARK;MANAGER
Fetched 4 rows
 7902;FORD;ANALYST
 7900;JAMES;CLERK
 7566;JONES;MANAGER
 7839;KING;PRESIDENT
Fetched 4 rows
 7654;MARTIN;SALESMAN
 7934;MILLER;CLERK
 7788;SCOTT;ANALYST
 7369;SMITH;CLERK
Fetched 2 rows
 7844;TURNER;SALESMAN
 7521;WARD;SALESMAN
There are 14 rows in the EMP table.

The advantage of specifying an array size is that the client has more control over memory usage. Only the rows retrieved by one call to fetchmany, in this case four, will be stored in memory on the client. If we had used Cursor.fetchall instead of fetchmany, then all rows would be returned and would need to be stored in client memory.

The impact on the database server depends on the nature of the query. If the query was a set operation then all result rows would need to be stored in local memory on the database server until they had been fetched by the application or the cursor was closed. If the query was a row operation then it may only be necessary to store sufficient results to satisfy the latest fetch request.

As the above query includes an ORDER by clause, it is almost certainly executing as a set operation as there is not index on the ENAME column.

Setting the array size controls the number of network messages exchanged between the client and the server.

The DB API recommends that the default array size is 1. However, cx_oracle ignores this recommendation and sets the default to 50. We can check this using:

  printf ("Array Size = %d\n",cursor.arraysize);
If cursor.arraysize is not set, then the query will fetch all rows. Output will be:

Fetched 14 rows
 7876;ADAMS;CLERK
 7499;ALLEN;SALESMAN
 7698;BLAKE;MANAGER
 7782;CLARK;MANAGER
 7902;FORD;ANALYST
 7900;JAMES;CLERK
 7566;JONES;MANAGER
 7839;KING;PRESIDENT
 7654;MARTIN;SALESMAN
 7934;MILLER;CLERK
 7788;SCOTT;ANALYST
 7369;SMITH;CLERK
 7844;TURNER;SALESMAN
 7521;WARD;SALESMAN
If cursor.arraysize is set to 1, then the query will fetch one row at a time. Output will be:

Fetched 1 rows
 7876;ADAMS;CLERK
Fetched 1 rows
 7499;ALLEN;SALESMAN
Fetched 1 rows
 7698;BLAKE;MANAGER
Fetched 1 rows
 7782;CLARK;MANAGER
Fetched 1 rows
 7902;FORD;ANALYST
Fetched 1 rows
 7900;JAMES;CLERK
Fetched 1 rows
 7566;JONES;MANAGER
Fetched 1 rows
 7839;KING;PRESIDENT
Fetched 1 rows
 7654;MARTIN;SALESMAN
Fetched 1 rows
 7934;MILLER;CLERK
Fetched 1 rows
 7788;SCOTT;ANALYST
Fetched 1 rows
 7369;SMITH;CLERK
Fetched 1 rows
 7844;TURNER;SALESMAN
Fetched 1 rows
 7521;WARD;SALESMAN
Array INSERT Statement
The previous example investigated use of array size to manage fetches from SELECT statements. In the following sections we will discuss use of arrays with INSERT, UPDATE and DELETE statements. In this section we will cover array INSERT statements.

The following is an example of an array INSERT of three rows into the DEPT table:

sql = "INSERT INTO dept (deptno, dname, loc) VALUES (:deptno, :dname, :loc)"

try:
  cursor.prepare (sql)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to prepare cursor')
  printException (exception)
  exit (1)

array=[]
array.append ((50,'MARKETING','LONDON'))
array.append ((60,'HR','PARIS'))
array.append ((70,'DEVELOPMENT','MADRID'))

try:
  cursor.executemany (None,array)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to insert rows')
  printException (exception)
  exit (1)
Note that we are using the Cursor.executemany () method instead of Cursor.execute ()

If we enable trace for the above statement we can verify that an array insert was performed on the database server:

PARSING IN CURSOR #140238572911728 len=68 dep=0 uid=83 oct=2 lid=83 tim=1419870619434242 hv=3863288051 ad='91cf3288' sqlid='9fxqrt3m4a67m'
INSERT INTO dept (deptno, dname, loc) VALUES (:deptno, :dname, :loc)
END OF STMT
PARSE #140238572911728:c=0,e=267,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=0,tim=1419870619434241
EXEC #140238572911728:c=1000,e=516,p=0,cr=1,cu=7,mis=1,r=3,dep=0,og=1,plh=0,tim=1419870619434770
STAT #140238572911728 id=1 cnt=0 pid=0 pos=1 obj=0 op='LOAD TABLE CONVENTIONAL  (cr=1 pr=0 pw=0 time=258 us)'
WAIT #140238572911728: nam='SQL*Net message to client' ela= 17 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419870619435236
WAIT #140238572911728: nam='SQL*Net message from client' ela= 174 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419870619435451
CLOSE #140238572911728:c=0,e=5,dep=0,type=0,tim=1419870619435486
XCTEND rlbk=0, rd_only=0, tim=1419870619435523
In the above, the EXEC line includes r=3 which indicates that three rows were inserted.

We could also call executemany using cursor.statement as the first parameter:

  cursor.executemany (cursor.statement,array)
Array UPDATE Statement
We can also use arrays with UPDATE statements. In the following example, we are using a dictionary to pass the bind variable values to the cursor.executemany method:

sql = "UPDATE dept SET loc = :loc WHERE deptno = :dept"

try:
  cursor.prepare (sql)
except:
  printf ('Failed to prepare cursor')
  exit (1)

array=[]
array.append ({'dept':50,'loc':'SHANGHAI'})
array.append ({'dept':60,'loc':'MOSCOW'})
array.append ({'dept':70,'loc':'MUMBAI'})

try:
  cursor.executemany(None,array)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor')
  printException (exception)
  exit (1)
Again the trace shows that all three rows were updated using a single execution of the UPDATE statement:

PARSING IN CURSOR #140267303715936 len=47 dep=0 uid=83 oct=6 lid=83 tim=1419873983403707 hv=2665682896 ad='8bda6038' sqlid='gja8j7agf65yh'
UPDATE dept SET loc = :loc WHERE deptno = :dept
END OF STMT
PARSE #140267303715936:c=0,e=450,p=0,cr=0,cu=0,mis=0,r=0,dep=0,og=1,plh=267198286,tim=1419873983403669
EXEC #140267303715936:c=0,e=317,p=0,cr=3,cu=5,mis=0,r=3,dep=0,og=1,plh=267198286,tim=1419873983404140
STAT #140267303715936 id=1 cnt=0 pid=0 pos=1 obj=0 op='UPDATE  DEPT (cr=3 pr=0 pw=0 time=212 us)'
STAT #140267303715936 id=2 cnt=3 pid=1 pos=1 obj=87107 op='INDEX UNIQUE SCAN PK_DEPT (cr=3 pr=0 pw=0 time=70 us cost=1 size=21 card=1)'
WAIT #140267303715936: nam='SQL*Net message to client' ela= 1 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419873983404265
WAIT #140267303715936: nam='SQL*Net message from client' ela= 133 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419873983404418
CLOSE #140267303715936:c=0,e=4,dep=0,type=0,tim=1419873983404447
XCTEND rlbk=0, rd_only=0, tim=1419873983404492
As with the previous example, the EXEC line includes r=3 which indicates that three rows were updated.

Array DELETE Statement
Arrays can also be used with DELETE statements. As with UPDATE statements this example uses a dictionary to specify bind variables

sql = "DELETE FROM dept WHERE deptno = :dept"

try:
  cursor.prepare (sql)
except:
  printf ('Failed to prepare cursor')
  exit (1)

array=[]
array.append ({'dept':50})
array.append ({'dept':60})
array.append ({'dept':70})

try:
  cursor.executemany(None,array)
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to execute cursor')
  printException (exception)
  exit (1)
In the above example, the array is used to specify the deptno for each of the three rows to be deleted

As in the previous DML examples, the EXEC line in the trace file includes r=3 indicating that three rows were deleted

PARSING IN CURSOR #140404237827184 len=37 dep=0 uid=83 oct=7 lid=83 tim=1419881960556726 hv=1258566750 ad='8be19350' sqlid='3mpppmd5h8d2y'
DELETE FROM dept WHERE deptno = :dept
END OF STMT
PARSE #140404237827184:c=0,e=234,p=0,cr=0,cu=0,mis=1,r=0,dep=0,og=1,plh=0,tim=1419881960556725
EXEC #140404237827184:c=11999,e=12902,p=0,cr=49,cu=18,mis=1,r=3,dep=0,og=1,plh=2529563076,tim=1419881960569679
STAT #140404237827184 id=1 cnt=0 pid=0 pos=1 obj=0 op='DELETE  DEPT (cr=49 pr=0 pw=0 time=12065 us)'
STAT #140404237827184 id=2 cnt=3 pid=1 pos=1 obj=87107 op='INDEX UNIQUE SCAN PK_DEPT (cr=3 pr=0 pw=0 time=58 us cost=1 size=13 card=1)'
WAIT #140404237827184: nam='SQL*Net message to client' ela= 15 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419881960570446
WAIT #140404237827184: nam='SQL*Net message from client' ela= 200 driver id=1413697536 #bytes=1 p3=0 obj#=-1 tim=1419881960570681
CLOSE #140404237827184:c=0,e=22,dep=0,type=0,tim=1419881960570758
XCTEND rlbk=0, rd_only=0, tim=1419881960570869
Calling PL/SQL Subroutines
PL/SQL procedures and functions can be called using the Cursor.callproc and Cursor.callfunc methods

PL/SQL Procedures
Assume we have created the following PL/SQL procedure

CREATE OR REPLACE PROCEDURE proc1
(
  p_deptno NUMBER,
  p_dname VARCHAR2,
  p_loc VARCHAR2
)
IS
BEGIN
  INSERT INTO dept (deptno, dname, loc) VALUES (p_deptno, p_dname, p_loc);
END;
/
We can call the procedure using the following code:

deptno = 50
dname = 'MARKETING'
loc   = 'LONDON'

try:
  cursor.callproc('PROC1',(deptno,dname,loc))
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to call procedure')
  printException (exception)
  exit (1)
The cursor.callproc method specifies the name of the procedure and also a list of bind variable parameters.

Note that in this example the bind variables are input parameters.

PL/SQL Functions
Assume we have created the following PL/SQL function which returns the number of employees with a specific job in a specific department:

CREATE OR REPLACE FUNCTION func1
(
  p_job VARCHAR2,
  p_deptno NUMBER
)
RETURN NUMBER
IS
  l_result NUMBER;
BEGIN
  SELECT COUNT(*) INTO l_result 
  FROM emp 
  WHERE job = p_job 
  AND deptno = p_deptno;
  RETURN l_result;
END;
/
To call the function we use the cursor.callfunc method

job = 'SALESMAN'
deptno = 30

result = cursor.var (cx_Oracle.NUMBER)

try:
  cursor.callfunc('FUNC1',result,(job,deptno))
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to call function')
  printException (exception)
  exit (1)

printf ("Result = %d\n",result.getvalue ())
The above code creates a result variable using the cursor.var method. It then calls the PL/SQL function FUNC1 passing the job and deptno as bind variables. The result is returned in a Variable object and is then converted to a number using the Variable.getvalue method.

Procedures with OUT parameters
If a PL/SQL subroutine only returns one value it will typically be written as a function. However if the PL/SQL subroutine needs to return more than one value it will usually return them as OUT parameters.

The following procedure returns the employee name and job for a specific employee number.

CREATE OR REPLACE PROCEDURE proc2
(
  p_empno NUMBER,
  p_ename OUT VARCHAR2,
  p_job OUT VARCHAR2
)
IS
BEGIN
  SELECT ename,job INTO p_ename,p_job FROM emp WHERE empno = p_empno;
END;
/
The procedure has one input parameter (p_empno) and two output parameters (p_ename and p_job)

The following code calls PROC2:

empno = 7839
ename = cursor.var (cx_Oracle.STRING)
job   = cursor.var (cx_Oracle.STRING)

try:
  cursor.callproc('PROC2',(empno,ename,job))
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to call procedure\n')
  printException (exception)
  exit (1)

printf ("Ename = %s\n",ename.getvalue ())
printf ("Job = %s\n",job.getvalue ())
The above code creates an input parameter for empno and two output parameters to received the results for ename and job. The procedure is called using the Cursor.callproc method specifying the name of the procedure (PROC2) and a list of parameters.

The OUT parameters are returned as Variable objects. The Variable.getvalue method must be called to extract the value from the object.

Procedures with IN-OUT parameters
It is debatable whether IN-OUT parameters for simple data types are ever justified and it is therefore difficult to devise a credible example for the scott/tiger database. However, it is often necessary to call subroutines developed by others in which case they may follow different standards.

The following PL/SQL procedure adds the specified values for SAL and COMM to the existing values in the table for the specified EMPNO. The PL/SQL procedure returns the new values in the same parameters.

CREATE OR REPLACE PROCEDURE proc3
(
  p_empno NUMBER,
  p_sal   IN OUT NUMBER,
  p_comm  IN OUT NUMBER
)
IS
BEGIN
  UPDATE emp SET sal = sal + p_sal, comm = comm + p_comm
  WHERE empno = p_empno
  RETURNING sal, comm INTO p_sal,p_comm;
END;
/
The following code calls the PROC3 procedure for EMPNO 7499

empno = 7499

sal = cursor.var (cx_Oracle.NUMBER)
sal.setvalue (0,400)

comm = cursor.var (cx_Oracle.NUMBER)
comm.setvalue (0,200)

try:
  cursor.callproc('PROC3',(empno,sal,comm))
except cx_Oracle.DatabaseError, exception:
  printf ('Failed to call procedure')
  printException (exception)
  exit (1)

printf ("sal = %d\n",sal.getvalue ())
printf ("comm = %d\n",comm.getvalue ())
The above code sets a value for EMPNO which is an input parameter. It then creates Variable objects for the two output parameters and sets initial values for these parameters using the Variable.setvalue method. The first parameter in the Cursor.var call will normally be 0.

The procedure then calls PROC3 using the Cursor.callproc method. When the procedure returns, the values of the SAL and COMM parameters are extracted from the Variable objects using the Variable.getvalue method
