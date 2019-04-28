---
title: python-flask
date: 2017-6-25 15:17:16
tags: flask python
---


#### [['GET','POST]']一定按照这个顺序来写,否则get或post请求会无法访问
####  if request.method=='POST': #method或者request不能多写S 
#### url('.hello'),记得写''分号 
#### 提交表单插入数据时，表单的input框type一定的写成text，如果为email，则表单并不能提交，这个错误应该是bootstap开启了javascript表单验证不为email格式则不能提交表单
***
### 关于数据库操作对象的思考,如下代码
<!-- more -->

```python
SELECT STUNUM,NAME FROM STUDENT INNER JOIN COMSTU ON STUDENT.RFID=COMSTU.RFID AND COMNUM=1 AND SUBJECT='数学'
conn = sqlite3.connect('test.db')
c = conn.cursor()
b =conn.cursor()
if request.method=='POST':
    grade=request.form['grade']
    stuclass=request.form['stuclass'] 
    subject=request.form['subject'] 
    comnum =request.form['comnum']
    curclass=c.execute("SELECT STUNUM,NAME,CLASS,GRADE FROM STUDENT WHERE GRADE=:grade AND CLASS=:stuclass",{'grade':grade ,'stuclass':stuclass})
    curcom =b.execute("SELECT STUNUM,NAME,CLASS,GRADE FROM STUDENT INNER JOIN COMSTU ON STUDENT.RFID=COMSTU.RFID AND COMNUM=:comnum AND SUBJECT=:subject AND CLASS=:stuclass AND GRADE=:grade",{'comnum':comnum,'grade':grade,'stuclass':stuclass,'subject':subject})
    cursor= list(set(curclass).difference(set(curcom))) 
    return render_template('view.html', entries=cursor)



```
 上述代码执行了两次execute函数,但是第一次的数据会被覆盖掉，这是因为数据库连接对象只有一个，所以执行第二次查询的时候前面一次数据会被覆盖掉，虽然创建了两个游标，如需同时两次数据做差集，则需要创建两个数据库对象