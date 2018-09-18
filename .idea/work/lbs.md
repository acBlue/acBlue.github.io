#   oralce  迁移 myql

****  

## 最近工作内容是把公司的一个项目的数据库从oralce 切换到 mysql修改内容主要有如下几点: ##

- 代码修改

  
>   修改hbm文件，把oralce sequece 改为mysql自增主键，修改代码中的SQL语句，修改oralce专有函数： to_char(),to_date()。改为MySQL中的函数  date_format(),str_to_date(),以及时间处理的 date_add(),date_sub()。 NVL() 函数 需要改成 IFNULL()函数,表名需要改成大写


-  函数，存储过程修改

>    拼接符处理:  oralce 拼接符 || 需改用 MySQL函数  concat()或者+，注意concat()函数需要保证拼接的每一部分都不为null，否则拼接的结果为空。


>    删除:     delete from   table   需要改成 delete table from table.

  
>    更新：     update  table set (a,b)   需改成  update table set a  = (xxxxx),b=(xxxxxxxx)  , 注意 在MySQL 中  被更新字段不能作为条件参与查询。
>      查询：      MySQL 中 作为子查询生成的虚拟表必须给出 别名。

>    存储过程拼接SQL 中''  需要改成" 
  
   
>   merge into  table  using()
>   语法 在MySQL 中可以使用  唯一约束 加上 repalce insert 语法来实现同等效果; 
    
>   oralce 中  for in()  语法在MySQL中可以使用游标来实现
      
     
>   oralce 中的  动态游标   在mysql 中需要添加临时表来实现  
   
     
>   mysql存储过程无法像oralce中一样处理异常，只能返回异常编码