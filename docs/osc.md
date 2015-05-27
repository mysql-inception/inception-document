#Inception 对OSC的支持
Inception已经支持Percon ToolKit工具**`pt-online-schema-change`**，这样对表大表的修改操作，就不需要跳过Inception而手动去执行了，给线上操作又带来了非常大的方便性。  
为了更友好的实现对OSC的集成，增加了下面的一些参数：  

--------
|参数名称                                	| 作用域  	 |默认值 	 |说明
|:-----------------------------------------------|:---------|:-------------------|:-------|
|inception_osc_bin_dir                   	| GLOBAL  	 |无     	 |用于指定pt-online-schema-change脚本的位置，不可修改，在配置文件中设置|
|inception_osc_check_interval            	| SESSION 	 |5秒    	 |对应参数--check-interval，意义是Sleep time between checks for --max-lag.|
|inception_osc_chunk_size                	| SESSION 	 |1000   	 |对应参数--chunk-size|
|inception_osc_chunk_size_limit          	| SESSION 	 |4      	 |对应参数--chunk-size-limit|
|inception_osc_chunk_time                	| SESSION 	 |1      	 |对应参数--chunk-time|
|inception_osc_critical_thread_connected 	| SESSION 	 |1000   	 |对应参数--critical-load中的thread_connected部分|
|inception_osc_critical_thread_running   	| SESSION 	 |80     	 |对应参数--critical-load中的thread_running部分|
|inception_osc_drop_new_table            	| SESSION 	 |1      	 |对应参数--[no]drop-new-table|
|inception_osc_drop_old_table            	| SESSION 	 |1      	 |对应参数--[no]drop-old-table|
|inception_osc_max_lag                   	| SESSION 	 |3      	 |对应参数--max-lag|
|inception_osc_max_thread_connected      	| SESSION 	 |1000   	 |对应参数--max-load中的thread_connected部分|
|inception_osc_max_thread_running        	| SESSION 	 |80     	 |对应参数--max-load中的thread_running部分|
|inception_osc_min_table_size            	| SESSION 	 |16     	 |这个参数实际上是一个OSC的开关，如果设置为0，则全部ALTER语句都走OSC，如果设置为非0，则当这个表占用空间大小大于这个值时才使用OSC方式。单位为M，这个表大小的计算方式是通过语句： **"select (DATA_LENGTH + INDEX_LENGTH)/1024/1024 from information_schema.tables where table_schema = 'dbname' and table_name = 'tablename'"**来实现的。|
|inception_osc_on                        	| GLOBAL  	 |1      	 |一个全局的OSC开关，默认是打开的，如果想要关闭则设置为OFF，这样就会直接修改|
|inception_osc_print_sql                 	| GLOBAL  	 |1      	 |对应参数--print|
|inception_osc_print_none                	| GLOBAL  	 |1      	 |用来设置在Inception返回结果集中，对于原来OSC在执行过程的标准输出信息是不是要打印到结果集对应的错误信息列中，如果设置为1，就不打印，如果设置为0，就打印。而如果出现错误了，则都会打印|
|
----------


参数名称、作用域、默认值及意义上面都已经列出来了，针对全局的参数，比如inception_osc_on，都是用来控制所有的OSC行为的，这些参数的修改及查看都是通过上面第六节中所介绍的一样，当修改之后，立即生效。

而针对会话级的参数，因为这些参数是只能影响当前要执行的语句的行为，所以不能设置全局的值，必须要在执行前设置当前线程的值，Inception可以通过语句：inception set session session_variable_name=value来修改，这样就只会影响当前语句的执行。因为Inception只是一个服务器，那么在具体页面实现时，可能还需要在页面上加入对这些参数修改并且设置的窗口，针对每一个设置，在内部拼写对应的inception set session语句来设置，然后再开始ALTER TABLE。

对于ALTER比较大的表时，因为所用时间比较长，在修改时可能需要关注一下进度，因为OSC工具本身在执行时是打印进度信息的，所以Inception完全 可以提供这方面的信息，出于更友好的实现方式，Inception新加入一个语句，可以查询当前执行的语句的进度，语句语法为：
`inception get osc_percent '当前执行的SQL语句以及一些基本信息生成的SHA1哈希值'`  
通过这个语句，可以查看进度信息，这个返回的结果集包括下面5个列：

* **TABLENAME**：当前被修改的表名；
* **DBNAME**：当前被修改表所在的库名；
* **SQLSHA1**：当前要查询的语句的SHA1字符串；
* **PERCENT**：当前修改已经完成的百分比，这个值是0到100的值。
* **REMAINTIME**：当前修改语句还需要多久才能完成，如03:55表示还需要三分55秒，01:33:44表示还需要1小时33分44秒。

下图是在ALTER的时候，查到的正在做的信息：
![](images/osc.png)

上面是正在做的，而如果语句块中有多个修改表的操作，则前面的会看到执行完成的进度信息：
![](images/osccomplete.png)

具体在上层应用使用时，可以选择性的使用所需要的列。

至于上面提到的SHA1值是如何获得的，因为在应用提交语句时，都会审核通过才能提交到流程管理数据库中，那么这个语句就不会被修改了，而此时在审核通过的时候，返回的结果集新增一个列sqlsha1，而只有当Inception判断到当前语句满足使用OSC方式执行时，这个列才会有值，就会根据当前语句信息生成一个哈希值，这个值存储起来以方便后面执行时使用。

当进入执行阶段之后，在执行当前语句时，OSC会向inception返回进度信息，Inception在收到之后，会根据当前语句的SHA1值更新对应的进度信息，OSC会每百分之一返回一次进度信息，那Inception都会更新当前语句对应的进度信息，OSC不会返回100%，最大99%，而Inception做了处理，当检查到有successfully altered的信息之后，就将进度信息改为100%，剩余时间为00:00，而99%到100%之间做的事情包括清除环境的操作，所以时间可能比之前的1%的时间要长。

进度信息缓存有生命周期的，在整个语句块执行完成在退出前，相应的缓存信息就会被清除出去，之后再查询进度就查不到了。查不到的话，当前线程的阻塞也就返回了，说明已经完成。

**需要注意的是**，OSC全局参数最好别频繁修改，因为针对某一个语句的SHA1是分阶段的，生成是在审核阶段的，如果在审核时候没有打开，或者设置的表大小没有满足OSC方式，则不会生成SHA1，那么在执行时候，这个进度就不能被查询了，这个语句的执行情况就不能获取到，影响执行过程的体验。当然这个影响也不大，因为在执行完成之后，如果执行成功了，并且参数`inception_osc_print_none`为OFF，则会看到打印信息，里面包括成功或者失败的所有信息，而如果为ON，则如果结果集中有信息，则说明是执行错误了，如果没有则说明成功。
