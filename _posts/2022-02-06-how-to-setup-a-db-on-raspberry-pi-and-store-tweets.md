---
title: How to setup a database on your Raspberry Pi and store tweets on it
updated: 2022-02-09 18:32
category: data
tags: data, python, mariadb, raspberry pi, mysql, database, pandas, tweets, twitter
---

This blog post covers the steps that I had to follow in order to set up a database on my Raspberry Pi, on which I could then store tweets from a daily Twitter search focusing on specific keywords. I installed a Mariadb database on my Raspberry Pi so that it could efficiently store the tweets in a tabular format and easily query subsets of it when needed, without having to store csvs on Github. 

This post will not cover the actual Tweet retrieval process, which I will cover in another blog post.
If you're wondering "why" I'm doing all this and what the purpose actually is, you can already read my [previous post](https://www.polegato.me/blog/data/2021/01/30/automating_retrieval_of_tweets.html) on the series where I explain the rationale of collecting tweets and analyzing online discourse.

## Choosing the database that you need
The first step of the project is to choose the right database for your pi. I'd recommend reading [this overview](https://chipwired.com/databases-for-raspberry-pi/) covering the pros and cons of multiple dialects including PostgreSQL, MySQL, MongoDB and SQLite, which are some usual suspects. You should probably ask yourself whether you have dependencies that would require you to use one specific setup over another, or if there are low hanging fruits in your configuration that you could seize by choosing a DB that offer out of the box compatibility with your needs.

In my case, I didn't properly research this topic and I first hesitated between SQLite and MongoDB. My first attempt focused on MongoDB, which I had used a long time ago while collaborating on the [inca project](https://github.com/uvacw/inca) of the University of Amsterdam. It quickly appeared cumbersome to set up the packages that I installed via Brew on my Raspberry and I wasn't sure this would be a good idea as I am already very familiar with the traditional SQL dialects. I stopped right there and decided I would try SQLite. 
SQLite has the advantage of being a minimal SQL database because it can live in a single file. I first installed it but quickly realized that this minimalism came with its set of limitations when used out of the box, and I didn't really understand how I was supposed to use it as a server for my pi. At the time of writing this however, I found a very nice project called [twitter-to-sqlite](https://github.com/dogsheep/twitter-to-sqlite) that may simply answer all my initial needs in one convenient package. To be investigated!

After these two missteps, I started looking in the direction of MySQL, as it seemed to be a "usual suspect" for small (but not only) projects. I tried to install it on my pi and was prompted that I needed MariaDB instead, the free fork of MySQL after it was acquired by Oracle.
In order to install MariaDB on your pi and create your very own SQL server, simply run the following:

```
sudo apt update
sudo apt upgrade
sudo apt install mariadb-server
```

### Configuring DB users
You get default access to your DB as "root" with all rights allowed on the currently non-existent DB.
It is good practice to differentiate between multiple roles 

In my case I called the user "tweet_getter" and defined the ip adress to which it should have access. You can then identify it with a password.
```
GRANT ALL PRIVILEGES ON *.* TO 'tweet_getter'@'192.xxx.x.%' 
  IDENTIFIED BY 'your_password' WITH GRANT OPTION;
```

### Authorizing external connections
By default, MySQL is listening only to local connections (127.0.0.1). If you want to be able to access your database from outside of the pi itself, you need to modify some settings by editing the MySQL configuration file located in the “/etc/mysql” folder:

```
sudo nano /etc/mysql/my.cnf
```

You will need to comment the line bind-address, as such:
 ```
#bind-address = 127.0.0.1
```

Note that this line could be anywhere in the following files depending on your setup:
1. "/etc/mysql/mariadb.cnf" (this file) to set global defaults,
2. "/etc/mysql/conf.d/*.cnf" to set global options.
3. "/etc/mysql/mariadb.conf.d/*.cnf" to set MariaDB-only options.
4. "~/.my.cnf" to set user-specific options.

To apply these changes, restart MySQL with the following line:

```
sudo service mysql restart
```

### Modifying your MySQL port
By default, MySQL is configured to listen on port 3306, choose a free port, on our side we chose port 8457, port not used by other software or major protocols.

### Configuring the db to accept tweets special characters
If you want to avoid spending a few hours wanting to die, I recommend to read this [Stackoverflow post](https://stackoverflow.com/questions/20411440/incorrect-string-value-xf0-x9f-x8e-xb6-xf0-x9f-mysql), I've quoted the main takeaways:

> Turns out MySQL’s utf8 charset only partially implements proper UTF-8 encoding. It can only store UTF-8-encoded symbols that consist of one to three bytes; encoded symbols that take up four bytes aren’t supported.

> In fact, MySQL’s utf8 only allows you to store 5.88% ((0x00FFFF + 1) / (0x10FFFF + 1)) of all possible Unicode code points. Proper UTF-8 can encode 100% of all Unicode code points.

I found it interesting to find that my initial error while trying to upload data to the DB mentioned the incapacity to deal with the `\xF0\x9F\x92\x99` character, which I later understood was the 💙. You can check emoji unicode references [here](https://apps.timwhitlock.info/emoji/tables/unicode) if you're interested.

if you want a bit of history of drama, this is the [one-line commit](https://github.com/mysql/mysql-server/commit/43a506c0ced0e6ea101d3ab8b4b423ce3fa327d0) that created chaos for so many:

Be careful that you need to make the changes at every level (database, schema and table). Follow the [instruction steps from Mathias Bynens](https://mathiasbynens.be/notes/mysql-utf8mb4):

```
# For each database:
ALTER DATABASE database_name CHARACTER SET = utf8mb4 COLLATE = utf8mb4_unicode_ci;
# For each table:
ALTER TABLE table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
# For each column:
ALTER TABLE table_name CHANGE column_name column_name VARCHAR(191) CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
# (Don’t blindly copy-paste this! The exact statement depends on the column type, maximum length, and other properties. The above line is just an example for a `VARCHAR` column.)
```

Make sure to verify your config:

```
SELECT   
  column_name,
  table_name,
  character_set_name,
  column_type,
  collation_name
FROM 
  information_schema.columns 
WHERE
  table_schema = 'twitter' 
  AND table_name = 'tweets';
```

### Restarting mysql 
Restart the service to make sure your latest changes are taken into account:

```
sudo service mysql restart
```

## Adding data to the DB
Now that you've set up a database that accepts the actual utf-8 encoding, you can start retrieving the actual tweets (not covered in this post) and send them to your db. In my case, I already have a df that contains the results, and I need to upload it to the MySQL (MariaDB) DB. 

### Creating a table inside the DB
You first need to create the skeleton of the table that will receive the data you plan on exporting/importing depending on how you want to think about it. The different data types that exist among the SQL dialects can be confusing, but make sure that you choose what's best fitted for you. In my case, I ended up with the following statement that I automatically generated using the very handy library `csvkit`, which enables you to create create statements based on a schema:

```
CREATE TABLE tweets (
	id INT PRIMARY KEY, 
	iso_language_code VARCHAR(20), 
	created_at TIMESTAMP, 
	screen_name VARCHAR(20), 
	text MEDIUMTEXT, 
	location MEDIUMTEXT, 
	favorite_count INT, 
	retweet_count INT, 
	queried_at TIMESTAMP, 
	company VARCHAR(20)
);
```
I only modified some of the size of the `VARCHAR` variables or replace them with `MEDIUMTEXT` when I suspected potential emoji inserts from users (either in their tweets or their location, which is often used as a description by users, such as "from space 👾👾👾")

### Importing data inside the DB
Luckily, the Pandas package of Python comes with a [built-in function](https://pandas.pydata.org/docs/reference/api/pandas.DataFrame.to_sql.html) `to_sql` that enables you to send a dataframe to a DB by specifying its connection settings using the SQLAlchemy engine you need, in this case [MySQL](https://docs.sqlalchemy.org/en/13/dialects/mysql.html). To be honest, I advice you to take some time to read the documentation and description of SQL Alchemy if you're not yet familiar with it as understanding the basics can help you save a lot of time the next time you need to deal with interoperability of DBs.

In order to configure the SQLAlchemy engine, I've used the following:
```
engine = create_engine('mysql+mysqldb://tweet_getter:your_password@192.xxx.x.:port/twitter?charset=utf8mb4&binary_prefix=true',
                      encoding='utf-8', 
                      pool_recycle=3600,
                      convert_unicode=True,
                      echo=True)
```
Carefully note the `?charset=utf8mb4&binary_prefix=true`, which was crucial to get it to work as intended. 

I could then export my Pandas dataframe:

```
tweets.to_sql(
        con=engine, 
        name='tweets', 
        if_exists='replace',
        index = False,
        method='multi', 
        chunksize=50)
```

And that's it folks, I hope that this blog post helped you understand the steps needed to store tweets inside a database set up on your Raspberry Pi. Make sure to let me know if you have any questions via Twitter @NosyOwl.

# Sources
* https://howtoraspberrypi.com/enable-mysql-remote-connection-raspberry-pi/
* https://mariadb.com/kb/en/configuring-mariadb-for-remote-client-access/
* https://dev.mysql.com/doc/connector-python/en/connector-python-example-cursor-transaction.html
* https://mariadb.com/kb/en/data-types/
* https://mariadb.com/kb/en/setting-character-sets-and-collations/


