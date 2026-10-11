---
aliases: ["/blog/mongodb-the-nosql-db-2018/"]
title: "MongoDB: The NoSQL Database"
date: 2018-11-16T22:28:42Z
slug: ""
description: "A Database with no SQL"
keywords: ["mongodb", "nosql", "db", "database"]
draft: false
tags: ["tutorials"]
math: false
toc: true
---

### Created to lessen your drop table nightmares.
> Does the name refer to mangoes?

No, the ‘Mongo’ is short for ‘humongous’ and ‘DB’ is pretty self-explanatory that it stands for ‘Database’.
> What’s NoSQL?

NoSQL is a type of database which generally uses JSON Objects, instead of traditional SQL table system.
> Which language it’s written in?

It’s written in C++.
> What type of Database Management System does MongoDB follow?

It follows the Relational Database Management System (RDBMS).

## Why MongoDB?

### MongoDB is:

* Schema-less.

* It’s customizable.

* Deep query-ability. MongoDB supports dynamic queries on documents using a document-based query language that’s nearly as powerful as SQL.

* Scalable.

### Name someone whose already bearing it?

* [Shutterfly](http://shutterfly.com)

* [Aadhar](http://uidai.gov.in)

* [Metlife](http://metlife.com)

* [eBay](http://ebay.com)
> Seems interesting, how do I start using it?

## Installation:

### Windows

* Go [here](https://www.mongodb.org/downloads) and download the required setup for your system.

* Now extract your downloaded file to C:\ drive or any other location. Make sure the name of the extracted folder is mongodb-win32-i386-[version] or mongodb-win32-x86_64-[version]. Here [version] is the version of MongoDB download.

* Next, open the command prompt and type the following command in

	move mongodb-win64-* mongodb

then,

```
    C:\>md data
    C:\md data\db
```

* Navigate to the bin directory found in the installation folder, set the default DB folder.

```
    D:\set up\mongodb\bin>mongod.exe --dbpath "d:\set up\mongodb\data"
```

It’s done!

### Ubuntu/Linux

* To get the MongoDB public GPG key,

```
    sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 7F0CEB10
```
* Create a mongodb.list,

```
    echo 'deb http://downloads-distro.mongodb.org/repo/ubuntu-upstart dist 10gen' 
       | sudo tee /etc/apt/sources.list.d/mongodb.list
```

* Next, to update the repositories use. sudo apt-get update

* Install MongoDB via apt-get intall mongo-10gen = 2.2.3

### Fingertip commands:

Start: `sudo service mongodb start`

Stop: `sudo service mongodb stop`

Restart: `sudo service mongodb restart`

Help: `db.help()`

## Basic Operations

### Creating a database:

	use DATABASE_NAME

* The command is used to create or use an existing database.

### Check the current database

	db

* The command is used to know with which database we’re currently interacting.

### List Databases

	show dbs

* The command is used to list databases.

Note: Empty Databases aren't shown in the list.

### Insert data in the database

	db.insert([Raw JSON/JSON file])

* Suppose JSON file is named as ‘dt.json’, then the function will be used as, db.insert(dt.json)

### Query Data

	db.COLLECTION.find()

* The method is used to find documents in a database, it shows the document in an unstructured way. You may go [here](https://www.tutorialspoint.com/mongodb/mongodb_query_document.htm) to study more about queries in MongoDB

### Formatted Data

	db.COLLECTION.find().pretty()

* The command is used to show data in a formatted way.

### Update Data

	db.COLLECTION_NAME.update(SELECTION_CRITERIA, UPDATED_DATA)

* The command is used to Update the original data with another data or any addition to it.

### Drop database

	db.dropDatabase()

* The command is used to drop the currently selected database.
