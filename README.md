# GAW.SH URL Shortener

---------------------------------------

*Eric O'Callaghan*

*Started: November 11th, 2011*

http://gaw.sh/ is a simple URL shortener which I created mostly over a two week vacation using PHP with a MySQL database containing two tables ("urls" and "visits")

My design skills suck, but it works. Thanks to [Jico](https://github.com/jico) for the alternating table row background colors tip! (see "tr:nth-child..." in [gawsh.css](https://github.com/ericoc/gaw.sh/blob/master/gawsh.css#L89-97))

---------------------------------------

## Adding URLs

### This section describes most of the main/index page of http://gaw.sh/

Check that the IP address adding the URL has not hit the rate limit defined within the isRude function in functions.php:

	// Create an interval => limit array that defines rudeness for an IP address
	$rudeness = array(1 => 1, 60 => 5, 3600 => 10, 86400 => 30);

To prevent abuse or spam, per the above array - a single IP address cannot add more than:
 * 1 URL per second
 * 5 URLs per minute (60 seconds)
 * 10 URLs per hour (3,600 seconds)
 * 30 URLs per day (86,400 seconds)

Check that URL actually exists and does not return 404
* http://php.net/manual/en/book.curl.php

If they gave an (optional) custom alias:
* Make sure it is alpha-numeric and not taken

Checks URL domain name against:
* [Dumb domain list](https://github.com/ericoc/gaw.sh/blob/master/admin/dumb.txt.example)
	* Mostly contains other URL shorteners, localhost, gaw.sh itself, etc...
* Spamhaus Domain Block List (DBL)
	* http://www.spamhaus.org/dbl/
* SURBL
	* http://www.surbl.org/
* URIBL
	* http://www.uribl.com/
* Google Safe Browsing API
	* https://developers.google.com/safe-browsing/
* PhishTank API
	* https://www.phishtank.com/api_info.php

Checks domain names IP address(es) against:
* Spamhaus ZEN (SBL, SBLCSS, XBL and PBL)
	* http://www.spamhaus.org/zen/

If everything goes well and checks out, we add a URL to the "urls" table of the MySQL database, then...

	mysql> show columns from urls;
	+--------+--------------------+------+-----+-------------------+-----------------------------+
	| Field  | Type               | Null | Key | Default           | Extra                       |
	+--------+--------------------+------+-----+-------------------+-----------------------------+
	| id     | int(10) unsigned   | NO   | PRI | NULL              | auto_increment              |
	| alias  | varchar(50)        | NO   | UNI | NULL              |                             |
	| url    | text               | NO   |     | NULL              |                             |
	| ip     | varchar(128)       | NO   |     | NULL              |                             |
	| time   | timestamp          | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
	| status | enum('-1','0','1') | NO   |     | 1                 |                             |
	+--------+--------------------+------+-----+-------------------+-----------------------------+
	6 rows in set (0.01 sec)

If they did not give an (optional) custom alias:
* Generate one using the base36 of auto-incremented numeric database ID (from MySQL "urls" table)
	* $shorturl = base_convert($id, 10, 36);
* If the generated one is also taken:
	* Keep generating new ones using base36 of random generated number (0-10), followed by numeric database ID, and UNIX timestamp until we find something that is not taken
		* $shorturl = base_convert(rand(0,10).$id.time(), 10, 36);

Example data:

	mysql> select * from urls order by id desc limit 1;
	+------+---------+---------------------+---------------+---------------------+--------+
	| id   | alias   | url                 | ip            | time                | status |
	+------+---------+---------------------+---------------+---------------------+--------+
	| 1244 | 8static | http://8static.com/ | 98.221.114.54 | 2011-12-11 14:49:55 | 1      |
	+------+---------+---------------------+---------------+---------------------+--------+
	1 row in set (0.00 sec)

---------------------------------------

## Visiting URLs

### This section describes what happens when someone tries to go to a shortened URL by visiting http://gaw.sh/whatever

.htaccess rewrites any requests for gaw.sh/whatever that are not for a real file or directory to /go.php?x=whatever like so:

	RewriteEngine On
	RewriteCond %{DOCUMENT_ROOT}%{REQUEST_URI} !-d
	RewriteCond %{DOCUMENT_ROOT}%{REQUEST_URI} !-f
	RewriteRule ^(.*?)/?$ /go.php?x=$1 [L]

A row is inserted in to the MySQL "visits" table with the database ID of the URL from the "urls" table (relational, woo!) as well as the IP address, browser, and referring URL of the visitor along with the timestamp of the visit:

	mysql> show columns from visits;
	+----------+------------------+------+-----+-------------------+-----------------------------+
	| Field    | Type             | Null | Key | Default           | Extra                       |
	+----------+------------------+------+-----+-------------------+-----------------------------+
	| id       | int(10) unsigned | NO   | MUL | NULL              |                             |
	| ip       | varchar(128)     | NO   |     | NULL              |                             |
	| browser  | text             | YES  |     | NULL              |                             |
	| referrer | text             | YES  |     | NULL              |                             |
	| time     | timestamp        | NO   |     | CURRENT_TIMESTAMP | on update CURRENT_TIMESTAMP |
	+----------+------------------+------+-----+-------------------+-----------------------------+
	5 rows in set (0.00 sec)

Example data:

	mysql> select * from visits order by id desc limit 1;
	+------+---------------+-----------------------------------------------------------------------------------------------------------------------+----------------+---------------------+
	| id   | ip            | browser                                                                                                               | referrer       | time                |
	+------+---------------+-----------------------------------------------------------------------------------------------------------------------+----------------+---------------------+
	| 1244 | 98.221.114.54 | Mozilla/5.0 (Macintosh; Intel Mac OS X 10_7_2) AppleWebKit/535.2 (KHTML, like Gecko) Chrome/15.0.874.121 Safari/535.2 | http://gaw.sh/ | 2011-12-11 14:50:08 |
	+------+---------------+-----------------------------------------------------------------------------------------------------------------------+----------------+---------------------+
	1 row in set (0.00 sec)

HTTP 301 redirect ("Moved Permanently") to the long/actual URL if it is active (status "1"):

	$ curl -I http://gaw.sh/8static
	HTTP/1.1 301 Moved Permanently
	Date: Mon, 12 Dec 2011 00:21:54 GMT
	Server: Apache
	Location: http://8static.com/
	Content-Type: text/html

HTTP 410 ("Gone") error message if the short URL is determined to be disabled (status "0"):

	$ curl -I http://gaw.sh/nope
	HTTP/1.1 410 Gone
	Date: Mon, 12 Dec 2011 01:00:34 GMT
	Server: Apache
	Content-Type: text/html

HTTP 404 ("Not Found") error message otherwise (if the short URL never existed or has a funky status like "-1" for "Hidden") :

	$ curl -I http://gaw.sh/404
	HTTP/1.1 404 Not Found
	Date: Mon, 12 Dec 2011 01:00:54 GMT
	Server: Apache
	Content-Type: text/html

---------------------------------------

## Public URL Statistics

There is also a publicly usable function by which to view the number of visits to a short URL.
The number of visits to a specific alias can be seen by prepending the alias with "/i/", such as http://gaw.sh/i/whatever

## Administering URLs

I built a simple administration panel to view/manage URLs at /admin/ which just uses .htaccess/.htpasswd for authentication.
The dumb domain name list is stored in /admin/ as well at /admin/dumb.txt

Allows an administrator to search and sort URLs by all of their different characteristics/fields pretty easily
* Stores search/sort methods in cookies so refreshes of the page do not wipe out the search/sort values

Gives the ability to quickly/easily disable a spam/malicious/bad/whatever URL
* Clicking the status ("Active" or "Disabled") of a URL allows to you quickly enable/disable a URL
* Clicking the IP address of the creator of a bad URL allows you to quickly disable any and all URLs created by that IP address

Lets an administrator edit the details of any specific/individual URL
* No method by which to delete a URL from the table or database entirely, only disable/edit/hide it
* Statuses are:
	* 0 - disabled
	* 1 - active (default)
	* -1 - "hidden" (returns 404; can only be set by admin)

Can view the number of visits to a URL; my first time using a JOIN in MySQL (thanks [Stan](https://github.com/Stantheman)!):

	mysql> SELECT urls.*, count(visits.id) AS visitors FROM urls INNER JOIN visits ON visits.id = urls.id GROUP BY visits.id ORDER BY visitors desc limit 1;
	+----+----------------+-------------------------------------------------------------------+---------------+---------------------+--------+----------+
	| id | alias          | url                                                               | ip            | time                | status | visitors |
	+----+----------------+-------------------------------------------------------------------+---------------+---------------------+--------+----------+
	|  3 | linodereferral | http://www.linode.com/?r=f9976de410ab39ee8e022d5bcc9ad24c6df18536 | 71.197.190.30 | 2011-12-04 19:20:15 | 1      |       52 |
	+----+----------------+-------------------------------------------------------------------+---------------+---------------------+--------+----------+
	1 row in set (0.00 sec)

---------------------------------------

## Double-checking URLs

[/admin/blcheck.php](https://github.com/ericoc/gaw.sh/blob/master/admin/blcheck.php) can be run as a cron job to perform all checks against any URLs added in the past week. You could modify a Linux servers crontab to include a line such as the following to run the script at 6:30 each morning :

	30 6 * * *	curl -su user:password http://gaw.sh/admin/blcheck.php | mail -r 'badurls@gaw.sh' -s 'gaw.sh Bad URLs Report' admin@gaw.sh

Furthermore, this script will also check local aliased URLs (i.e. gaw.sh/sketchyurl) from the past week for formatting as well as against the PhishTank and Google Safe Browsing APIs to get a head start on any abuse in the case where a local alias has been blacklisted and is redirecting to a phishing website that has not yet been placed on a blacklist.


### ENJOY!
