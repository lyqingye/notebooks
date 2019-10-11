## 利用kibana进行可视化操作

### 基本命令

1. 查看`elasticsearch`的状态 

   `GET /_cat/health?v`

2. 查看`elasticsearch`的索引

   `GET /_cat/indices?v`

3. 添加索引

   `PUT indexName`

4. 删除索引

   `DELETE indexName`

### 对文档进行CRUD操作

1. 添加document

   ```js
   // 最后面跟的是id
   PUT /school/_doc/1
   {
     "name" : "xuf",
     "age" : "2"
   }
   ```

2. 查询document

   ```js
   GET /school/_doc/1
   ```

3. 更新document

   ```js
   POST /school/_update/1/
   {
     "doc":{
         "name" : "modify1"
     }
   }
   
   ```

4. 删除document

   ```js
   DELETE /school/_doc/1
   ```

5. 使用脚本更新值

   ```js
   // 在原有的值上加上2
   POST /school/_update/1
   {
     "script": "ctx._source.age += 2"
   }
   ```

