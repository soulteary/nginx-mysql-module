# Nginx MySQL Module

- (c) 2012 Arutyunyan Roman (arut@qip.ru)
- (c) 2012 DenoFiend zhao (denofiend@gmail.com) (add transaction support in mysql)
- (c) 2015 ideal (https://github.com/ideal) (bugfix for close mysql and ngx_http_mysql_cleanup)


== MySQL support in NGINX ==

* this module is asynchronous (non-blocking)
* this module support transaction in mysql
* this module response is generated in rds format, so it's compatible with ngx_rds_json.
* requires nginx-mtask-module (https://github.com/arut/nginx-mtask-module)
* requires standard MySQL client shared library libmysqlclient.so
* requires ngx_rds_json module(https://github.com/denofiend/rds-json-nginx-module)


## NGINX configure

```bash
./configure --add-module=/path/to/nginx-mysql-module/ --add-module=/path/to/nginx-mtask-module/ --add-module=/path/to/rds-json-nginx-module/
```

## Usage examples

MySQL:

```mysql	
USE test;

CREATE TABLE `users` (
  `name` varchar(128) DEFAULT NULL,
  `id` int(11) NOT NULL auto_increment,  PRIMARY KEY (`id`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8;

CREATE TABLE `a` (
  `id` int(11) NOT NULL DEFAULT '0',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8; 

CREATE TABLE `b` (
  `id` bigint(20) NOT NULL,
  `name` varchar(128) NOT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

nginx.conf:

```nginx
http {

	...

	server {

		...


		# TCP/IP access
		#mysql_host 127.0.0.1;

		# unix socket access (default)
		#mysql_host localhost;

		#mysql_user theuser;
		#mysql_password thepass;
		#mysql_port theport;

		mysql_database test;
		mysql_connections 32;

		mysql_charset utf8;

		# this turns on multiple statements support
		# and is REQUIRED when calling stored procedures
		mysql_multi on;

		mtask_stack 65536;

		
		#transaction location conf
		location  /test/select {

			mysql_query "SELECT a.`id`, b.`name` from  a,  b where a.`id` = b.`id` limit 3";

			rds_json on;
			rds_json_root ids;

		}
		location /test/insert {

			mysql_transaction "BEGIN" "INSERT INTO `a` values('$arg_id')"  "INSERT INTO `b` values('$arg_id', '$arg_val')" "COMMIT";
			rds_json on;
		}


		location ~ /all {

			mysql_query "SELECT name FROM users WHERE id='$arg_id'";
		}

		location /multi {

			mysql_query "SELECT id, name FROM users ORDER BY id DESC LIMIT 10; SELECT id, name FROM users ORDER BY id ASC LIMIT 10";
		}

		location ~ /select.* {

			if ( $arg_id ~* ^\d+$ ) {

				mysql_query "SELECT name FROM users WHERE id='$arg_id'";

			}
		}

		location ~ /insert.* {

			mysql_escape $escaped_name $arg_name;

			mysql_query "INSERT INTO users(name) VALUES('$escaped_name')";
		}

		location ~ /update.* {

			mysql_query "UPDATE users SET name='$arg_name' WHERE id=$arg_id";
		}

		location ~ /delete.* {

			mysql_query "DELETE FROM users WHERE id=$arg_id";
		}

		# store number of page shows per user
		location ~ /register-by-cookie-show.* {

			# register show in database
			# need a location with this query:
			# mysql_query "INSERT INTO shows(id) VALUES('$id') ON DUPLICATE KEY UPDATE count=count+1";
			mysql_subrequest /update_shows_in_mysql?id=$cookie_abcdef;

			# output content
			rewrite ^(.*)$ /content last;
		}

		# spammers use big ids in picture refs to see who opened the message
		location ~/spammer-style-image-show-register.* {

			# need a location with this query:
			# mysql_query "INSERT INTO shows(id) VALUES('$arg_id')";
			mysql_subrequest /insert_show?id=$arg_id;

			rewrite ^(.*)$ /content last;
		}

		location ~ /session.* {

			# get user session
			# need a location with this query:
			# mysql_query "SELECT username, lastip FROM sessions WHERE id='$id'";
			mysql_subrequest /select_from_mysql?id=$cookie_abcdef $username $lastip;

			# go to content generator
			rewrite ^(.*)$ /content?username=$username&lastip=$lastip last;
		}

		...
	}

	...

}
```

access:

```bash
curl 'http://localhost:8080/test/insert?id=1&val=test_case_1'
{"errcode":0}

curl 'http://localhost:8080/test/insert?id=2&val=test_case_2'
{"errcode":0}

curl 'http://localhost:8080/test/select'
{"ids":[{"id":1,"name":"test_case_1"},{"id":2,"name":"test_case_2"}]}

curl 'localhost:8080/insert?name=foo'
11

curl 'localhost:8080/select?id=11'
foo

curl 'localhost:8080/update?id=11&name=bar'
<nothing>

curl 'localhost:8080/select?id=11'
bar

curl 'localhost:8080/delete?id=11'
<nothing>

curl 'localhost:8080/select?id=11'
<nothing>
```
