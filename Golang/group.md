## group 将表分组，sum等函数对每个组运算

### where 与having 作用的对象不同。

WHERE 子句作用于表，HAVING 子句作用于表中的组。

WHERE 在分组和聚集计算之前选取输入行（因此，它控制哪些行进入聚集计算）， 
HAVING 在分组和聚集之后选取分组的行。因此，WHERE 子句不能包含聚集函数； 
因为试图用聚集函数判断那些行输入给聚集运算是没有意义的。 相反，HAVING 子句总是包含聚集函数。如having sum(qty)>1000

having一般跟在group by之后，执行记录组选择的一部分来工作的。
where则是执行所有数据来工作的。

### 学生，科目，成绩表的分组查询
```
select * from score;

# 查询每个学生成绩大于90的科目 
select a.student_id,a.subject_id, score from score as a where a.score > 90 group by a.student_id, a.subject_id, score;

# 查询每个学生成绩大于90的科目的数目
select a.student_id, count(*) from score as a where a.score > 90 group by a.student_id;

# 计算每个学生的科目总成绩
select a.student_id, sum(a.score) from score as a group by a.student_id;

# 每个学生的科目按成绩排序
select a.student_id, a.subject_id, a.score from score as a group by a.student_id, a.subject_id, a.score order by a.student_id, a.score desc;

# 每个学生的最高成绩
select a.student_id, max(a.score) from score as a group by a.student_id;

# 语句没问题， 但sql里有中文字符或中文空格， 报语法错误, 提示select错误，一般是中文输入问题, 所在行的钱一行有中文也不行
select a.subject_id,a.student_id from score as a group by a.subject_id, a.student_id;

# 查询每个最高成绩和对应的科目
select DISTINCT a.subject_id,a.student_id,a.score from score as a group by a.subject_id,a.student_id,a.score having max(a.score);
```

```
select * from score;
select a.subject_id,a.student_id,a.score from score as a group by a.subject_id;
# select a.id,a.subject_id,a.student_id,a.score,b.id,b.subject_id,b.student_id,b.score from score as a left join score as b on a.subject_id=b.subject_id; 

# select a.id,a.subject_id,a.student_id,a.score,b.id,b.subject_id,b.student_id,b.score from score as a left join score as b on a.subject_id=b.subject_id and a.score<=b.score;

# select a.subject_id,a.student_id,a.score from score as a left join score as b on a.subject_id=b.subject_id and a.score>=b.score 
# group by a.subject_id,a.student_id,a.score  having count(a.subject_id)>=4 order by a.subject_id,a.score desc;

#select a.subject_id,a.student_id,a.score from score as a left join score as b on a.subject_id=b.subject_id and a.score>=b.score 
# group by a.subject_id,a.student_id,a.score   order by a.subject_id,a.score desc ;

select a.subject_id,a.student_id,a.score from score as a 
group by a.subject_id,a.student_id,a.score  having count(a.subject_id)>=1  order by a.subject_id,a.score desc ;

```





