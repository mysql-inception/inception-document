#Inception安装说明
看到这个手册，想必已经得到了源码，恭喜你。  

首先就是编译，在源码根目录下面有一个文件inception_build.sh，执行命令`sh inception_build.sh`，会输出使用方法。
实际上只需要执行`inception_build.sh debug [Xcode]`即可，后面的平台是可选的，如果不指定就是linux平台，而如果要指定是Xcode，就后面指定Xcode，而debug是编译的目录，编译之后，所有的生成文件都在这个目录下面，包括可执行文件Inception。可执行文件在`debug/sql/Debug/`目录下面（不同平台有可能不相同）。

编译完成之后，就是使用了，那么需要一个配置文件（inc.cnf）:
````
[inception]
general_log=1
general_log_file=inception.log
port=6669
socket=/data/workspace/inception_data/inc.socket
character-set-client-handshake=0
character-set-server=utf8
inception_remote_system_password=root
inception_remote_system_user=wzf1
inception_remote_backup_port=3306
inception_remote_backup_host=127.0.0.1
inception_support_charset=utf8mb4
inception_enable_nullable=0
inception_check_primary_key=1
inception_check_column_comment=1
inception_check_table_comment=1
inception_osc_min_table_size=1
inception_osc_bin_dir=/data/temp
inception_osc_chunk_time=0.1
inception_ddl_support=1
inception_enable_blob_type=1
inception_check_column_default_value=1
````
上面这些参数的配置都是本人随便举例而已。

现在就到启动时间了，那么启动有两种方式，和MySQL是一样的  
1. Inception --defaults-file=inc.cnf  
2. Inception --port=7777  

第二种方法就是只指定一个端口，其它参数都是默认值，而第一种方法就是在配置文件中可以指定很多参数，按照自己喜欢的规则来配置。

启动如果不报错的话，说明已经启动成功了，实际上很难让它报错，因为非常轻量级

启动成功之后，可以简单试一下看，通过MySQL客户端    
`mysql -uroot -h127.0.0.1 -P6669`    
登录上去之后，再执行一个命令：  
`inception get variables;`  
输出了所有的变量，恭喜你，已经启动成功了，都说了非常简单。  
具体的使用的命令等在后面相应章节都会讲到，继续往后看吧！！！  
