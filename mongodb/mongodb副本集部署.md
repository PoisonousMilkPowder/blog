mongodb副本集部署
=

#### 基于mongodb v4.0.9 [安装](https://docs.mongodb.com/manual/installation/)

+ **本地开发环境部署** [参考](https://docs.mongodb.com/manual/tutorial/deploy-replica-set-for-testing/)
    
    + 新环境部署
    
        ***将在本机创建3个mongodb成员实例，1个主节点，2个从节点，规定副本集名称为rs0***
    
        1. 创建必要的目录
        
            ```
            # -p是为了创建嵌套目录
            # 这里在根目录下创建了3个测试目录（可能会有权限限制），用于存放mongodb实例的数据文件
            mkdir -p /srv/mongodb/rs0-0  /srv/mongodb/rs0-1 /srv/mongodb/rs0-2
            ```
    
        2. 启动3个mongod服务
                
            ```
            # 数据节点1
            mongod --replSet rs0 --port 27017 --dbpath /srv/mongodb/rs0-0 --smallfiles --oplogSize 128
            
            # 数据节点2
            mongod --replSet rs0 --port 27018 --dbpath /srv/mongodb/rs0-1 --smallfiles --oplogSize 128
            
            # 数据节点3
            mongod --replSet rs0 --port 27019 --dbpath /srv/mongodb/rs0-2 --smallfiles --oplogSize 128
            ```
            
            `--dbpath`默认: /data/db on Linux and macOS, \data\db on Windows
            
            可以使用`--fork`启动一个后台进程，但同时必须指定`--logpath`
            
            3个实例默认绑定到localhost，也可通过[--bind_ip](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-bind-ip)设置其他主机名或ip
             
            设置[--smallfiles](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-smallfiles)和[--oplogSize](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-oplogsize)是为了减少每个mongod实例使用的磁盘空间，对于测试和开发部署非常理想，可以防止机器过载。    
            
            mongod的启动参数也可单独放在[配置文件](https://docs.mongodb.com/manual/reference/configuration-options/)中，使用[--config](https://docs.mongodb.com/manual/reference/program/mongod/index.html#cmdoption-mongod-config)来引用
            
            可以参考的配置文件：
            
            ```
            systemLog:
			  path: /srv/mongodb/rs0-0/log/mongodb.log
			  destination: file
			  logAppend: true
			storage:
			  dbPath: /srv/mongodb/rs0-0/data
			  journal:
			    enabled: true
			net:
			  port: 27017
			replication:
			  replSetName: rs
			processManagement:
			  fork: true
			  pidFilePath: /srv/mongodb/rs0-0/mongod.pid
			security:
			  keyFile: /srv/mongodb/rs0-0/mongod.key
            ```
            
            [mongod参数参考](https://docs.mongodb.com/manual/reference/program/mongod/#mongod)
            
        3. 随便连接到一个节点实例，这里选择27017
        
            ```
            # 将在这个端口的实例的下初始化
            mongo --port 27017
            ```
            
            不用`--port`指定端口时，将默认连接到27017
            
        4. 在当前连接的mongo下初始化
        
            在mongo shell下执行后续命令
            
            ``` 
            rsconf = {
              _id: "rs0",
              members: [
                {
                 _id: 0,
                 host: "<hostname>:27017"
                },
                {
                 _id: 1,
                 host: "<hostname>:27018"
                },
                {
                 _id: 2,
                 host: "<hostname>:27019"
                }
               ]
            }
            ```

            记得替换`<hostname>`，我本地配成了`localhost`
            
            然后执行：
            
            ``` 
            rs.initiate(rsconf)
            ```
            
            正常情况下会输出类似内容
            ``` 
            {
                "ok" : 1,
                ...
            }
            ```
            
            这时会在三个节点中选举产生一个PRIMARY节点，剩下的两个节点就变成SECONDARY节点，`mongo shell`会变成
            ``` 
            rs0:SECONDARY> 
            或者
            rs0:PRIMARY> 
            ```
            
        5. 显示当前副本集的配置信息
        
            执行 `rs.conf()`
            
            正常情况下输出的信息类似于：
            
            ```  
            {
            	"_id" : "rs0",
            	"members" : [
            		{
            			"_id" : 0,
            			"host" : "localhost:27017",
            			"arbiterOnly" : false,
            			"buildIndexes" : true,
            			"hidden" : false,
            			"priority" : 1,
            			"tags" : {
            				
            			},
            			"slaveDelay" : NumberLong(0),
            			"votes" : 1
            		},
            		{
            			"_id" : 1,
            			"host" : "localhost:27018",
            			"arbiterOnly" : false,
            			"buildIndexes" : true,
            			"hidden" : false,
            			"priority" : 1,
            			"tags" : {
            				
            			},
            			"slaveDelay" : NumberLong(0),
            			"votes" : 1
            		},
            		{
            			"_id" : 2,
            			"host" : "localhost:27019",
            			"arbiterOnly" : false,
            			"buildIndexes" : true,
            			"hidden" : false,
            			"priority" : 1,
            			"tags" : {
            				
            			},
            			"slaveDelay" : NumberLong(0),
            			"votes" : 1
            		}
            	],
            	...
            }
            ```
            
            用mongoose连接的时候，url要写成`mongodb://localhost:27017,localhost:27018,localhost:27019/${DATABASENAME}`
            
        6. 副本集故障转移测试
        
            [关闭mongod节点](https://docs.mongodb.com/manual/tutorial/manage-mongodb-processes/#stop-mongod-processes)
            
            这里关闭节点使用：
            
            ``` 
            use admin
            db.shutdownServer()
            ```
            
            现在将之前的primary节点关闭后，副本集会自动从剩下的两个secondary节点中选举出一个primary节点。
            
            此时如果再关一个节点，最后的那1个节点只能是secondary节点，因为这个节点不能确定是自身问题还是其他节点的问题，所以此时整个副本集宕掉了。
            
    + 在已有mongodb单实例基础上部署
    
        ***将在本机创建3个mongodb成员实例，1个主节点，2个从节点，规定副本集名称为rs1***
    
        1. 关闭原来的mongod实例
        
        2. 重启mongod实例，使用`--replSet`指定副本集名称
        
        3. 连接到mongod实例
        
        4. 在mongo shell中执行`rs.initiate()`
        
            没有传入配置对象，会使用一个默认的配置，此时会输出类似信息：
            
            ``` 
            {
            	"info2" : "no configuration specified. Using a default configuration for the set",
            	"me" : "localhost:27020",
            	"ok" : 1,
            	...
            }
            ```
            
            `rs.conf()`查看副本集配置
            
            ``` 
            {
            	"_id" : "rs1",
            	"members" : [
            		{
            			"_id" : 0,
            			"host" : "localhost:27020",
            			"arbiterOnly" : false,
            			"buildIndexes" : true,
            			"hidden" : false,
            			"priority" : 1,
            			"tags" : {
            				
            			},
            			"slaveDelay" : NumberLong(0),
            			"votes" : 1
            		}
            	],
            	...
            }
            ```
             此时副本集中只有这一个主节点，下面将为它添加其他节点。
             
        5. 为副本集添加节点
        
            启动两个mongod实例，使用`--replSet`指定副本集名称
            
            在之前mongo shell中，执行`rs.add("<hostname><:port>")`将两个新实例加进去，添加成功后会有如下信息：
            
            ``` 
            {
            	"ok" : 1,
            	...
            }
            ```
            
            再执行`rs.conf()`可以看到`members`里面已经有了刚刚加进去的两个节点
            
        6. 测试
    
+ **生产环境部署** [参考](https://docs.mongodb.com/manual/tutorial/deploy-replica-set/#deploy-a-replica-set)
    
    生产环境的部署步骤与测试环境大致相同，但额外要注意：
    
    + 副本集节点要部署在不同的机器上，端口最好是27017；
    
    + 建议使用逻辑DNS主机名，而不是直接使用ip，避免ip变化带来的配置更改；
    
    + 使用`--bind_ip`配置一个可供远程客户端连接的ip或主机名，而不是仅仅使用默认的`localhost`
    
        ``` 
        测下来只有localhost, 127.0.0.1, 0.0.0.0这些是可以正常启动的
        ```
        
    + 尽可能保证副本集节点数为奇数，如果非要偶数，请添加一个仲裁节点（只参与投票选primary节点，不同步数据），在primary节点的mongo shell里执行`rs.addArb("<hostname><:port>")`添加仲裁节点。
        
        每个副本集最多有一个仲裁节点。
        
    + 关于分片

		[官方文档](https://docs.mongodb.com/manual/sharding/#considerations-before-sharding)
		
    	[分片集群搭建](https://blog.51cto.com/bigboss/2160311?source=dra)

    + 根据实际情况决定是否开启[访问控制](https://docs.mongodb.com/manual/tutorial/enable-authentication/)
    	
    	在副本集上设置访问控制需要配置两个地方
    	
		1. 副本集成员之间的[内部认证](https://docs.mongodb.com/manual/core/security-internal-authentication/)

    		+ keyfile验证
    		
    			1. 创建一个keyfile
	
					```
					# 生成756长度的随机数，经base64编码之后写入文件，文件名随意
					openssl rand -base64 756 > mongodb.key
					# 修改权限
					chmod 400 mongodb.key
					```
	
				2. 将密钥文件复制到每个复制集成员服务上
	
					确保启动mongod实例的用户对密钥文件有访问权限
	
				3. 启动每个启用了访问控制的副本集成员
	
					```
					# 在配置文件中添加
					security:
					  keyFile: <path-to-keyfile>
					replication:
					  replSetName: <replicaSetName>
					  
					或者
					
					# 在mongod命令后加参数
					mongod --keyFile <path-to-keyfile> --replSet <replicaSetName>
					```
					
					配置keyfile之后，就默认开启了`--auth`。意味着后续需要做用户验证。
			
				4. 连接到每个mongo shell
		
					此时还没有创建用户，也就意味着没有验证，有些方法会报错。
					
				5. init replset
		
					```
					rs.initiate(
					  {
					    _id : 'rs',
					    members: [
					      { _id : 0, host : "localhost:27017" },
					      { _id : 1, host : "localhost:27018" },
					      { _id : 2, host : "localhost:27019" }
					    ]
					  }
					)
					```
					
				6. 创建一个超级管理员
	
					必须在primary节点下创建超级管理员，超级管理员在admin数据库下至少要有`userAdminAnyDatabase`权限。
					```
					admin = db.getSiblingDB("admin")
					admin.createUser(
					  {
					    user: "admin",
					    pwd: "admin",
					    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
					  }
					)
					```
					
					我为了方便直接创建了一个root用户
					
					```
					use admin
					db.createUser({user:"root",pwd:"root",roles:["root"]})
					```
					
					`use admin`会改变shell环境下的`db`变量，`db.getSiblingDB("admin")`只是返回指定的数据库。
					
					[mongodb内置角色](https://docs.mongodb.com/manual/reference/built-in-roles/#built-in-roles)
	
				7. 验证超级管理员
		
					因为管理员是在admin数据库下创建的，所以要在admin数据库下验证
					
					`db.getSiblingDB("admin").auth("admin", "admin" )`
					
					或者也可以创建一个新的mongo shell连接,
					
					`mongo -u "admin" -p "admin" --authenticationDatabase "admin"`
		
				8. 为其他数据库创建用户

					```
					db.getSiblingDB("home").createUser(
					  {
					    "user" : "home",
					    "pwd" : "home",
					    roles: [ { "role" : "dbAdmin", "db" : "home" } ]
					  }
					)
					```
					
					`roles`字段也可写成`[ "dbAdmin" ]`，也就是没指定数据库，此时以调用`createUser() `的数据库为准，比如此处是home。
					
					`mongoose`连接`home`数据库的时候就得把用户名密码带上了
					
					`mongodb://home:home@127.0.0.1:27017,127.0.0.1:27018,127.0.0.1:27019/home`
    			
    		+ x.509证书验证

    			[官方文档](https://docs.mongodb.com/manual/tutorial/configure-x509-member-authentication/)
    
		2. mongo客户端与副本集的用户认证

			通过用户名，密码，数据库名来进行用户认证,当副本集用了内部认证，这步就是必须的。
            
#### 其他参考

+ [为什么是奇数](https://segmentfault.com/q/1010000010636309)

+ [关于 MongoDB 复制集的几个问题](https://learnku.com/laravel/t/1498/some-problems-about-mongodb-replication-set)

+ [生产环境部署带keyfile安全认证和用户权限的mongodb副本集步骤示例](https://github.com/johnnian/Blog/issues/8)

+ [mongodb复制集维护小结](https://www.cnblogs.com/zhaowenzhong/p/5667312.html)

+ [使用正确的姿势连接复制集](http://www.mongoing.com/archives/2642)

+   有两种方法实现从机的查询：

    第一种方法：db.getMongo().setSlaveOk();
    
    第二种方法：rs.slaveOk();
    
    上面两行命令即允许此连接从副本读取.但是这种方式有一个缺点就是，下次再通过mongo进入实例的时候，查询仍然会报错，为此可以通过下列方式
    
    `vi ~/.mongorc.js`
    
    增加一行rs.slaveOk()，以后每次通过mongo命令进入都可以查询了。
    
