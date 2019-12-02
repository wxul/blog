---
title: nodejs写爬虫
tags: ['javascript','node']
date: 2016-06-28 17:31:26
categories: [Node]
---

可能用到的一些库
``` js
var fs = require('fs');
var path = require('path');
var req = require('request');
var chee = require('cheerio');
var sql = require('mssql');
```

我用的mssql，先配置：
``` js
var sql_config = {
    user: 'sa',
    password: 'xxx',
    server: 'localhost',
    database: 'd',
    pool: {
        max: 100,
        min: 0,
        idleTimeoutMillis: 30000
    }
};
```

判断是否已经有数据（根据主键id），有的话就更新，没有就添加数据
``` js
function insertVal(data, callback) {
    var conn = new sql.Connection(sql_config, function (err) {
        if (err) {
            return console.error(err);
        }
        var request = conn.request();
        request.input('id', data.id);
        request.query('select count(1) as count from [xxx] where id=@id', function (err, e1, e2) {
            if (err) {
                return console.error(err);
            }
            console.log(e1, e2);
            request = conn.request();
            if (e1 &amp;&amp; e1[0] &amp;&amp; e1[0].count) {
                console.log('update');
                //update
            }
            else {
                console.log('insert');
                //insert
            }
            if (typeof callback === 'function') {
                callback();
            }
        })
    });
}
```

异步请求
``` js
var count = 0;
function get() {
    if (current > end) {
        return console.log('over');
    }
    var url = 'xx';
    setTimeout(function () {
        //current++;
        get();
    }, 10);
    req(url, function (err, res, body) {
        if (err) return console.error(err);
        if (res.statusCode != 200) {
            console.warn('status:' + res.statusCode + ',response:' + res);
            return;
        }
        //dom操作
        var dom = JSON.parse(body);
        if (dom.status) {
            var data = dom.data;
            console.log(data.mid);
            data.level_info = data.level_info || {};
            insertVal(data, function () {
                count++;
            });
        }
    });
}
get();
```

效率有点慢啊平均每条数据40ms~