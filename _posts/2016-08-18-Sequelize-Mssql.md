---
layout: post
title: Getting starting with Sequelize and MSSQL
keywords: node, sequelize, mssql, sqlexpress
---

## A little background
Sequelize is an ORM which provides for interaction between a number of databases, among which is MS SQL.  To manage the MS SQL communication, it leverages another library, `tedious`.

# JavaScript Setup
You should specify the connection properties with the sequelize parameters -- sorry, but the MSSQL Connection String won't work here.  
Due to the other issues I ran into, I gave up trying to get the URI paramater to work, too, though in theory in should. 

Below is a basic configuration with some extra parameters defined that you'll probably want.
``` JS
// sequelize.js
import Sequelize from 'sequelize'

// Note that you must use a SQL Server login -- Windows credentials will not work.
const sequelize = new Sequelize('MyDatabase', 'login', 'password', {
    dialect: 'mssql',
    host: 'localhost',
    port: 1433, // Default port
    dialectOptions: {
        instanceName: 'SQLEXPRESS',
        requestTimeout: 30000
    },
    pool: {
        max: 50,
        min: 0,
        idle: 10000
    }
})

export default sequelize
```

An easy way to get testing this is below, taken from the [sequelize website](http://docs.sequelizejs.com/en/latest/docs/getting-started/) and wrapped in a test

``` JS
// sequelize.test.js
import sequelize from './sequelize'

describe('sequelize', () => {
    it('should connect to the database', () => {
        sequelize
            .authenticate()
            .then(function (err) {
                console.log('Database connection has been established successfully.');
            })
            .catch(function (err) {
                console.log('Unable to connect to the database:', err);
            });
    })
})
```


# Machine / Database Configuration
*This is where it gets really fun...*

**1) You must enable TCP/IP in SQL Server Configuration Manager**

<img src="/images/sql enable tcp.png" alt="enable tcp screenshot" />
    
#### Help: I can't find SQL Server Configuration Manager!

Note that if you're on Windows 10 or a similar environment, this may be a little harder to find.  Microsoft has even published a [page](https://msdn.microsoft.com/en-us/library/ms174212.aspx) specifically to help you with this task.

The long and short of it is that you must **fully** type out the name of the snap-in in the start menu:

ex: `SQLServerManager12.msc`  You may swap out the "12" to match your version of SQL Server: 
* 2008 = 10
* 2012 = 11
* 2014 = 12
* 2016 = 13

**2) SQL Server must be set up to allow SQL Server Authentication as a login option**

**3) You must have `Sql Server Browser` running in Services.  This allows `tedious` to find the connection.**

*(Error: `SequelizeConnectionError: Failed to connect to localhost:undefined in 15000ms`)*

<img src="/images/sql services.png" alt="sql services screenshot" />

    If it's not running, don't forget to setup `Automatic` start for the future! 

**4) The login you're using should be mapped to the database you're trying to access.**

1) Right click the Database Server and click `Properties`
1) Go to `Security` page
1) Under `Server Authentication`, choose the `SQL Server and Windows Authentication mode` radio button
1) Click `OK`
1) Restart your server