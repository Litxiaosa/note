### **SQL**

```mysql
# sql 四舍五入,达到保留两位小数位的目的：
    select cast(13.145 as   decimal(10,   2))
    结果为：13.15
```



```mysql
下单时间 <= DATE_SUB(CURDATE(), INTERVAL 30 DAY) 或者 date_format(p.payTime, '%Y-%m') = date_format(now(), '%Y-%m')
```



```mysql
# 创建索引 ：
 ALTER TABLE 表名 ADD INDEX 自定义索引名 (字段名)
# 查看索引
      mysql> show index from tblname;
      mysql> show keys from tblname;
    · Table
    表的名称。
    · Non_unique
    如果索引不能包括重复词，则为0。如果可以，则为1。
    · Key_name
    索引的名称。
    · Seq_in_index
    索引中的列序列号，从1开始。
    · Column_name
    列名称。
    · Collation
    列以什么方式存储在索引中。在MySQL中，有值‘A’（升序）或NULL（无分类）。
    · Cardinality
    索引中唯一值的数目的估计值。通过运行ANALYZE TABLE或myisamchk -a可以更新。基数根据被存储为整数的统计数据来计数，所以即使		对于小型表，该值也没有必要是精确的。基数越大，当进行联合时，MySQL使用该索引的机 会就越大。
    · Sub_part
    如果列只是被部分地编入索引，则为被编入索引的字符的数目。如果整列被编入索引，则为NULL。
    · Packed
    指示关键字如何被压缩。如果没有被压缩，则为NULL。
    · Null
    如果列含有NULL，则含有YES。如果没有，则该列含有NO。
    · Index_type
    用过的索引方法（BTREE, FULLTEXT, HASH, RTREE）。
    · Commen
```



```mysql
# 批量更新sql
UPDATE categories
    SET dingdan = CASE id
         WHEN 1 THEN 3
         WHEN 2 THEN 4
         WHEN 3 THEN 5
     END,
     title = CASE id
         WHEN 1 THEN 'New Title 1'</update>
         WHEN 2 THEN 'New Title 2'
         WHEN 3 THEN 'New Title 3'
     END
   WHERE id IN (1,2,3)
   
   
 # 例如：
   <update id="updatePackageById" parameterType="java.util.List">
      update app_price_package set
      current_price = CASE id
      <foreach collection="list" item="item" index="index" >
        WHEN #{item.id} THEN #{item.currentPrice}
      </foreach>
        END,
      times = CASE id
      <foreach collection="list" item="item" index="index" >
        WHEN #{item.id} THEN #{item.times}
      </foreach>
        END
        WHERE id in
      <foreach collection="list" item="item" index="index"  open="(" close=")"  separator=",">
        #{item.id}
      </foreach>
    </update>
```



```mysql
<foreach item='item' index='index' collection='itemIds' open=' and ta.cargo_id in ('separator=',' close=')'>
     #{item}
</foreach>
```



```mysql
# 模糊查询
like concat('%',#{warehouseName},'%')
```

