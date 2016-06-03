# TwoEfAy HTTP/2.0 Server

This server is based on the Twisted Python example from [hyper-h2 demo](https://github.com/python-hyper/hyper-h2/blob/master/examples/twisted/twisted-server.py)

### Pre-requisites: 
- python 2.7.9+
- pip
- mysql-server
- virtualenv (optional, but highly recommended!)

##### Note: 
Some of the Requirements listed below may need other libraries to be installed, otherwise you'll 
have issues when you try to install libraries with pip. 
Off the top of my head, you may need to run:
```sh
sudo apt-get install python-cffi libffi-dev build-essentials python-dev libmysqlclient-dev ...
```

### Requirements
  * h2
  * hyper
  * twisted
  * cffi
  * cryptography
  * service_identity
  * pyopenssl
  * typing
  * setuptools
  * pyasnl
  * pyasnl-modules
  * mysql-python
  * pycrypto
  * python-dateutil
  * pyotp
  * time
  * keyring
  * twilio

For example, to install the above in your virtualenv run:
```sh
pip install h2 twisted cryptography mysql-python etc...
```

#### Optional (helpful for testing)
- httpie
- httpie-http2

***
#### NOTE on testing with cURL
You may also need to rebuild cURL with the nghttp2 to be able to send HTTP/2.0 
requests.
If you need help, [click here for instructions](https://serversforhackers.com/video/curl-with-http2-support).
You may also want to look at hyper or httpie, httpie-http2 as additional 
testing tools to cURL.

Test whether you successfully rebuilt cURL with the following GET and POST:
```sh
curl --http2 https://httpbin.org/get
curl --http2 -d {"hello":"world"} -X POST https://httpbin.org/post
```
Testing with httpbin.org is awesome because it returns the contents of your request
so you can see immediately if you sent over what you thought you did.
***

#### NOTE on Keys & Certificates
You will have to generate your own self-signed certificate and key as they are 
required for HTTP/2.0, so lookup a tutorial on how to do that if you don't know. 
You'll see in the code, at the bottom, where to specify the location of your 
certificate and key (mine are in a certs dir: "../certs/server.key")
If you don't do this, the server won't run (not properly, at least). You *may* 
create a key without a pass phrase for your own testing but this is clearly not 
a secure option in real life.


#### NOTE on MySQL databases
For the server to work properly, it is assumed you have previously set up a MySQL
database. If you don't know how to do that, here's a quick and dirty intro to
what I did:

```sh
sudo apt-get install mysql-server
```
During the install you'll be asked to set a root password. **Do NOT forget it.**

Start it: 
```sh 
mysql -u root -p 
<when prompted, enter the root password you set up>
```

```mysql
CREATE DATABASE cs130;
GRANT ALL ON cs130.* TO '130user' IDENTIFIED BY '130Security!';

USE cs130;
CREATE TABLE Users(
	username VARCHAR(40) UNIQUE NOT NULL PRIMARY KEY, 
	email VARCHAR(40) NOT NULL,
	phone VARCHAR(12) NOT NULL,
	id_token VARCHAR(80) UNIQUE, 
	dev_token VARCHAR(80),
	status VARCHAR(10) DEFAULT '', 
); 

```
Note that *​*username**​ and *​*id_token**​ are the PRIMARY KEYS.
Also note that *​*username**​ and *​*email**​ are ​__UNIQUE__​ and ​__NOT NULL__​, 
(i.e. we can't create users with the same username or email **and** 
we can't create users with blank username or email).


Now, if you want to see what your table looks like, you run a query:
```mysql
SELECT * FROM Users;
```

On the droplet server this will return something like this:
```mysql
mysql> select * from Users;
+---------------+------------------------+------------+----------------------+---------------+-----------+
| username      | email                  | phone      | id_token             | dev_token     | status    |
+---------------+------------------------+------------+----------------------+---------------+-----------+
| bro           | bro@email.com          | NULL       | 76545246587232       | NULL          |           |
| isThisOn?     | blsssa@example.com     | NULL       | 3IYsBH6wDEyVCZfn     | NULL          |           |
| pastMyBedtime | INeedSleep@example.com | NULL       | oO1A92W7Yc4r3bEa     | MyDevTokenLOL | success   |
| ross          | rch******@gmail.com    | 619******* | vKrnc3nfFd7YQmTC     |               |           |
| sexy          | sexy@email.com         | NULL       | 23472                | NULL          |           |
| something     | bla@example.com        | NULL       | 234h38fKW0dje238E    | NULL          |           |
| tester        | tester@email.com       | NULL       | 4535343              | NULL          |           |
| testuser      | test@email.com         | NULL       | 23598745434943758438 | NULL          |           |
+---------------+------------------------+------------+----------------------+---------------+-----------+

```
The id_token here isn't very random/unique because these users were
created with an older version of the server. Also, the dev_token
is NULL because I created the column after the table already existed
and we haven't yet obtained the device tokens for these
users from their iPhones with the /verify POST request from the App.

You can run much more complex queries, of course. 
For example in the server I verify a certain user exists with the
following query:
```mysql
SELECT username, COUNT(*) FROM Users WHERE username='<username>'
```

I register new users into the database with the following:
```mysql
INSERT INTO Users (username, email, phone, id_token, dev_token, status) 
VALUES ('<username>', '<email>', '<phone>', '<random token>', '', '');
```

---

#### To run server:
```sh
python twisted-server.py localhost
```
You will have to enter your PEM key phrase after running the above command.


#### If deploying locally

> Assuming you've set up your database correctly, created the desired table 
> within your database, and provided the correct database info and credentials 
> in the **db_open()** function, you'll be able to /register a user **sexy** via
> a POST with the following request:

```sh
curl --http2 -H "Content-Type: application/json" -X POST -d '{"username":"sexy","email":"whatever@some.com", "phone":"1234567890"}' https://localhost:8080/register
```

**NOTE:** 
If you get an error stating the certificate is self-signed and cannot be verified, then run cURL with the **-k** flag. 
For example, to register user **dude** with email **animal@email.com** and IGNORE certificate security:
```sh
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"username":"dude", "email":"animal@email.com", "phone":"1234567890"}' https://localhost:8080/register
```

### GET request to your server:
```sh
curl --http2 -k https://localhost:8080/user/sexy
```
> Note: in the above GET request, we try to obtain info on a user **sexy**. 
> Of course, this user has to be in your database before you can get a return
> for this.

The return for our droplet server would be the following JSON:
```
{"token":"23472"}
```

===

### Deployed on the droplet
---
###### POST params available and their purpose:
__C__ = Client
__S__ = Server
__A__ = App
__U__ = User

| Path       | Purpose                                  | JSON data sent with request  | Server response/action                               |
| ----------:|-----------------------------------------:|-----------------------------:|-----------------------------------------------------:|
| /register  | C registers new U                        | username, email, phone       | responds to C's POST with JSON containing id_token   |
| /verify    | A verifies U and device token            | id_token, dev_token          | sends POST to APN with dev_token                     |
| /login     | C reports that U attempts to log in      | id_token                     | POSTs to APN with dev_token (or SMS OTP to U)        |
| /auth      | C polls for U's successful/failed login  | id_token                     | responds to request with success/failure/wait/''     |
| /backup    | C provides OTP entered by unverified U   | id_token, otp                | verify OTP the U entered at C                        |
| /success   | A reports U authenticated successfully   | id_token                     | POSTs /login to C with JSON with id_token, success   |
| /failure   | A reports U authenticated unsuccessfully | id_token                     | POSTS /login to C with JSON with id_token, failure   |


The database has been setup on Chris's droplet, so you can test POST and GET 
requests in the following way:

#### Client -> Server /register POST:
```sh
curl --http2 -kv -H "Content-Type: application/json" -X POST -d '{"username":"crazy","email":"crazy@email.com", "phone":"1234567890"}' https://45.55.160.135:8080/register
```
Server responds with:
```
{'token':'<the user's token>'}
```

#### GET:
```sh
curl --http2 -kv https://45.55.160.135:8080/user/crazy
```
Server responds with:
```
{'token':'<the user's token>'}
```

#### App -> Server /verify POST:
```sh
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"id_token":"3IYsBH6wDEyVCZfn","dev_token":"34f4rhg5424h234f83hf"}' https://localhost:8080/verify
```
This will set the dev_token for user identified by id_token who happens to currently be user "**isThisOn?**".
Server responds with:
{'login':'verified'}

### Client -> Server /login POST:
```sh
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"3IYsBH6wDEyVCZfn"}' https://localhost:8080/login
```
Server responds with:
{'login':'<success or failure>'}

### Client -> Server /backup POST (for unverified users):
```sh
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"3IYsBH6wDEyVCZfn", "otp":"234234"}' https://localhost:8080/backup
```
Server responds with:
{'login':'<success or failure>'}

### App -> Server /success and /failure POSTs:
```sh
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"3IYsBH6wDEyVCZfn"}' https://localhost:8080/success
curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"3IYsBH6wDEyVCZfn"}' https://localhost:8080/failure
```
Server sends the following to Client:
{'token':'<user's token>','login':'<success or failure>'}


### Example:
```sh
ross@ubuntu:~$ curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"username":"pastMyBedtime","email":"INeedSleep@example.com","phone":"1234567890"}' https://45.55.160.135:8080/register
{"token": "oO1A92W7Yc4r3bEa"}
ross@ubuntu:~$ curl --http2 -k https://45.55.160.135:8080/user/pastMyBedtime
{"token": "oO1A92W7Yc4r3bEa"}

ross@ubuntu:~$ curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"id_token":"o01A92W7Yc4r3bEa","dev_token":"MyDevTokenLOL"}' https://45.55.160.135:8080/verify (**THIS FAILED BECAUSE I MISTYPED THE id_token, LOL!)
ross@ubuntu:~$ curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"id_token":"oO1A92W7Yc4r3bEa","dev_token":"MyDevTokenLOL"}' https://45.55.160.135:8080/verify

ross@ubuntu:~$ curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"oO1A92W7Yc4r3bEa"}' https://45.55.160.135:8080/success
ross@ubuntu:~$ curl --http2 -k -H "Content-Type: application/json" -X POST -d '{"token":"oO1A92W7Yc4r3bEa"}' https://45.55.160.135:8080/failure
```
All the POST requests above trigger POST requests on the server that should be 
sent to various places but I'm currently sending all to https://httpbin.org/post,
so I can verify that the Server POST requests contain the correct info. Thus far, they do. :)

=====

### How to run the server:
Chris set up a systemd to make everyone's life easier, here are the instructions:
You can control the server as follows:
1. sudo systemctl start twoefay-server
2. sudo systemctl stop twoefay-server
3. sudo systemctl restart twoefay-server

If anything goes wrong, you can see logs with:
1. sudo systemctl status twoefay-server
2. sudo journalctl -xe

-----
-----
#### Note on _encrypted string_:
The **crypto.py** module does the encryption and decryption. I call it within
the server to decrypt the encrypted string sent with the request but if you,
like me, test with cURL, you'll have to manually generate the encrypted string
as follows.
  * Open up your text editor and save the following in a generator.py (or whatever).
```python
import crypt
from datetime import datetime
from dateuril.parser import parse

# this is my super-secret key
do_crypto = crypt.AEScipher('f456a56b567f4094ccd45745f063a465')
date = datetime.now()
date_format = '%Y/%m/%d %H:%M:%S'

secret = str(date)
msg = do_crypto.encrypt(secret)
print msg
```
Run it and you'll get the string. :)

**Currently, this functionality is disabled, so you don't have to generate
this string when testing!**
