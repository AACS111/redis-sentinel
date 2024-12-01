
# 前言
本篇文章主要是因为工作需要搭建一个哨兵模式的集群，提供给后端服务使用，所以编写的一个搭建使用流程。
采用的是Docker Compose 进行统一管理搭建的一个**一主二从，三个哨兵**的集群，版本是选的一个`redis:6.0`的版本或者更新也行。


## **Docker Compose介绍**
Docker Compose 是 Docker 提供的一个工具，用于定义和运行多容器 Docker 应用程序。它的主要目的是简化多服务应用程序的部署和管理，特别是在开发和测试环境中。

**1. 功能与目的**
- 简化容器管理：Docker Compose 允许用户在一个 YAML 文件（默认是 docker-compose.yml）中定义多个服务及其配置，使用单个命令进行创建和启动。这些服务可以是应用程序的不同微服务、数据库、缓存、队列等。
- 定义应用的架构：它不仅仅是启动 Docker 容器的命令工具，还可以定义容器之间如何通信、环境变量、卷、网络等，提供一种可复用的方式来描述和配置应用的运行环境。
- 开发和测试环境：Compose 特别适用于开发环境，因为它可以快速重建、重启或清除测试或开发环境，非常适合持续集成（CI）和开发周期。


**2. 主要命令**

- `docker-compose up -d` 创建并启动定义在配置中的所有服务。
- `docker-compose down` 停止并删除所有创建的容器、网络、卷。
- `docker-compose ps` 查看正在运行的容器。
- `docker-compose logs` 查看服务的日志。
- `docker compose up -d --force-recreate` 停止删除原有配置中的所有服务，并且重新创建启动








### 最终实现效果
1.**监控**：监控主从服务的运行，当被监控的redis服务实例发生问题时(变为 down）日志输出状态,
2.  **自动故障转移**：发生故障达到指定时间后哨兵会从所有副本服务器中选举出一个新的主服务器，并将其他副本重新配置为该新的主服务器的从服务器。

**架构图**

![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b986baf028554f2d9ee65417d0c60a1d.png)





# 一、搭建集群



### 1、创建文件结构
- 先切换为 root角色，要不然可能有些会没权限
	```bash
	sudo su root 
	# 密码
	```

	
- 创建文件结构，用于存放文件
选择自己要存放的目录下，执行创建文件夹的目录
	```bash
	mkdir -p redis-sentinel/master-slave/config redis-sentinel/master-slave/data/master  redis-sentinel/master-slave/data/master redis-sentinel/master-slave/data/slave1 redis-sentinel/master-slave/data/slave2 redis-sentinel/sentinel/sentinel1 redis-sentinel/sentinel/sentinel2 redis-sentinel/sentinel/sentinel3
	```

	具体结构展示
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/540e615da5d44d99b7091e44075d0030.png)

### 2、创建redis节点
- 先进入master-slave/config文件夹，创建配置文件

	```bash
	cd master-slave/config
	```
- 创建配置文件

	1.	编写主节点redis-master配置文件
		```bash
		vim redis-master.conf
		```
		写入内容
			`slave-announce-ip`需要改为自己docker宿主机的ip
		```bash
		#集群客户端连接密码
		requirepass 123456
		# 从节点连接主节点的验证密码，因为每个主节点都可能挂掉变为从节点，所以都需要masterauth用于校验
		masterauth 123456   
		#指定 master的ip  这边设置是为了防止主挂掉后，重启后ip会变成自己容器中的IP
		slave-announce-ip 192.168.192.12
		#指定 master的端口，需要和下面文件映射的ports端口一致
		slave-announce-port 6479
		#0.0.0.0在服务器的环境中，指的就是服务器上所有的ipv4地址
		bind 0.0.0.0
		```
	2.	编写从节点redis-slave1配置文件
		```bash
		vim redis-slave1.conf
		```
		**写入内容**
		从节点需要添加`slaveof`，是master的ip和端口
		`slave-announce-ip`一样需要改为自己docker宿主机的ip
		```bash
		#集群客户端连接密码
		requirepass 123456
		# 从节点连接主节点的验证密码，因为每个主节点都可能挂掉变为从节点，所以都需要masterauth用于校验
		masterauth 123456   
		#slave节点需要添加slaveof用于指定主节点
		slaveof 192.168.192.12 6479
		#指定 slave的ip  这边设置是为了防止主挂掉后，重启后ip会变成自己容器中的IP
		slave-announce-ip 192.168.192.12
		#指定 salve的端口
		slave-announce-port 6480
		#0.0.0.0在服务器的环境中，指的就是服务器上所有的ipv4地址
		bind 0.0.0.0
		```
	3.	编写从节点redis-slave2配置文件
		```bash
		vim redis-slave2.conf
		```
		**写入内容**
		从节点需要添加`slaveof`，是master的ip和端口
		`slave-announce-ip`一样需要改为自己docker宿主机的ip
		```bash
		#集群客户端连接密码
		requirepass 123456
		# 从节点连接主节点的验证密码，因为每个主节点都可能挂掉变为从节点，所以都需要masterauth用于校验
		masterauth 123456   
		#slave节点需要添加slaveof用于指定主节点
		slaveof 192.168.192.12 6479
		#指定 slave的ip  这边设置是为了防止主挂掉后，重启后ip会变成自己容器中的IP
		slave-announce-ip 192.168.192.12
		#指定 salve的端口
		slave-announce-port 6481
		#0.0.0.0在服务器的环境中，指的就是服务器上所有的ipv4地址
		bind 0.0.0.0
		```
	
	- **执行命令步骤**![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/7cdd921a429d4442bd791ea4a229c17a.png)

- 编写docker-compose.yml文件
	1. 退回master-slave目录下
	
		```bash
		cd ..
		```

	2. 创建`docker-compose.yml`文件
		```bash
		vim docker-compose.yml
		```

		不要去自定义networks网络，会无法提供给容器之外的服务进行连接
		```yaml
		services:
		#主节点
		  redis-master:
		  # 版本
		    image: redis:6.0
		    container_name: redis-master
		    ports:
		      - "6479:6379"
		    #绑定数据卷，方便指向对应节点的配置文件和数据存放文件
		    volumes:
		      - ./config/redis-master.conf:/usr/local/etc/redis/redis.conf
		      - ./data/master:/data
		    #添加配置,指定配置文件位置，为映射到容器的配置文件位置
		    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
		 #从节点1
		  redis-slave1:
		    image: redis:6.0
		    container_name: redis-slave1
		    ports:
		      - "6480:6379"
		    #绑定数据卷，方便指向对应节点的配置文件和数据存放文件
		    volumes:
		      - ./config/redis-slave1.conf:/usr/local/etc/redis/redis.conf
		      - ./data/slave1:/data
		    #添加配置
		    depends_on:
		      - redis-master
		    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
		  redis-slave2:
		    image: redis:6.0
		    container_name: redis-slave2
		
		    ports:
		      - "6481:6379"
		    volumes:
		      - ./config/redis-slave2.conf:/usr/local/etc/redis/redis.conf
		      - ./data/slave2:/data
		    depends_on:
		      - redis-master
		    command: ["redis-server", "/usr/local/etc/redis/redis.conf"]
		
		# 这个地方注意不能自定义创建网络，服装外部服务不能连接
		# networks:
		#   redisco:
		
		```

-  创建容器
	- 创建容器
	在docker-compose文件的位置执行下面命令创建容器
		```bash
		docker-compose up -d
		```
	- 检查容器
		在使用`docker-compose ps`查看如果都启动则代表成功、
		
		**执行命令步骤**
	如果有问题则可以使用`docker-compose down`删除掉容器，排除问题后在重新执行创建命令
		![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/84eaf171b69949aeb1e67301e56dbe72.png)
### 3、验证节点
- 检查从节点是否绑定成功
	1. 进入主节点容器
	
		```bash
		docker exec -it redis-master /bin/bash
		```
	 2. 通过redis-cli连接redis
		```bash
		redis-cli -a [密码]
		```
	3. 通过`info Replication`命令查看连接信息
		是否分别显示了创建的两个从节点，并且`state=online`
		![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/743c110f70ea421d9607095156f5929c.png)
		> **state常见错误显示**
		> 1. state=disconnected :
		表示从节点与主节点的连接已经断开，可能是因为网络故障、主节点下线或者配置错误。
		> 2. state=wait_replconf_ack :
		从节点正等待主节点的 ACK 确认。这是哨兵模式下的常见状态，表示从节点知道主节点正在选举新的主节点，或者主节点正在进行故障转移。
		> 3. state=wait_bgsave :
		当主节点正在启动一个后台保存过程（如 AOF 重写或 RDB 快照），从节点会处于这个状态，等待主节点完成这个操作以便开始同步。
- 检查是否可以主节点正常写入，并且数据同步到从节点
	1. 直接在主节点写入
	`set name zhangsan`![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/8e305ddcc4d748b1add0ec2a8abac5dc.png)
	2. 连接从节点进行查看
	可以选择在外部通过客户端进行连接查看，ip是安装docker的宿主机的ip，端口就是设置的对应的映射出来的端口，可以看到刚刚在主节点写入的数据则代表当前主从集群搭建成功。接下来搭建哨兵节点，用来自动管理集群。
	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/b5aa8fc22573432a80f26645b0f42745.png)

### 4、创建sentinel哨兵
创建的是sentinel哨兵，因为只有达到三个哨兵才满足基本的投票选举机制
- 创建配置文件
	1. 进入sentinel目录
	

		```bash
		cd ../sentinel/
		```
	2. 分别编写3个sentinel节点的配置文件
	- 在sentinel1目录下创建`sentinel1.conf`节点文件
		```bash
		vim sentinel1/sentinel1.conf
		```
		编写文件
		ip需要改为自己ip

		```bash
		#设置为 no 意味着 Redis 实例将允许外部网络的客户端连接
		protected-mode no
		#Sentinel1哨兵节点的端口
		port 26379
		#监听redis主节点的  Ip  端口
		sentinel monitor mymaster 192.168.192.12 6479 2
		#定义了Redis主节点在多长时间内无响应，Sentinel将认为它已经宕机
		sentinel down-after-milliseconds mymaster 5000
		#设置 Sentinel 执行一次故障转移的超时时间
		sentinel failover-timeout mymaster 60000
		#redis节点密码
		sentinel auth-pass mymaster 123456
		#指定ip和端口
		sentinel announce-ip "192.168.192.12"
		sentinel announce-port 26379
		```
	- 在sentinel2目录下创建`sentinel2.conf`节点文件
		```bash
		vim sentinel2/sentinel2.conf
		```
		编写文件
		内容和sentinel1的一样，就是把端口改一下

		```bash
		#设置为 no 意味着 Redis 实例将允许外部网络的客户端连接
		protected-mode no
		#Sentinel1哨兵节点的端口
		port 26380
		#监听redis主节点的  Ip  端口
		sentinel monitor mymaster 192.168.192.12 6479 2
		#定义了Redis主节点在多长时间内无响应，Sentinel将认为它已经宕机
		sentinel down-after-milliseconds mymaster 5000
		#设置 Sentinel 执行一次故障转移的超时时间
		sentinel failover-timeout mymaster 60000
		#redis节点密码
		sentinel auth-pass mymaster 123456
		#指定ip和端口
		sentinel announce-ip "192.168.192.12"
		sentinel announce-port 26380
		```
	- 在sentinel3目录下创建`sentinel3.conf`节点文件
		```bash
		vim sentinel3/sentinel3.conf
		```
		编写文件
		内容和sentinel1的一样，就是把端口改一下

		```bash
		#设置为 no 意味着 Redis 实例将允许外部网络的客户端连接
		protected-mode no
		#Sentinel1哨兵节点的端口
		port 26381
		#监听redis主节点的  Ip  端口
		sentinel monitor mymaster 192.168.192.12 6479 2
		#定义了Redis主节点在多长时间内无响应，Sentinel将认为它已经宕机
		sentinel down-after-milliseconds mymaster 5000
		#设置 Sentinel 执行一次故障转移的超时时间
		sentinel failover-timeout mymaster 60000
		#redis节点密码
		sentinel auth-pass mymaster 123456
		#指定ip和端口
		sentinel announce-ip "192.168.192.12"
		sentinel announce-port 26381
		```
- 创建docker-compose文件
	1. 在sentinel目录下创建docker-compose文件
		```bash
		vim docker-compose.yml
		```
	2. 编写内容
需要加上`--sentinel`，不用指定具体的网络

		```yaml
		services:
		  sentinel1:
		    image: redis:6.0
		    container_name: sentinel1
		    ports:
		      - "26379:26379"
		    volumes:
		      - ./sentinel1:/usr/local/etc/redis
		    #--sentinel 表示将启动一个 Sentinel 实例，不加则表示启动的普通实例
		    command: redis-sentinel /usr/local/etc/redis/sentinel1.conf  --sentinel
		
		  sentinel2:
		    image: redis:6.0
		    container_name: sentinel2
		    ports:
		      - "26380:26379"
		    volumes:
		      - ./sentinel2:/usr/local/etc/redis
		    command: redis-sentinel /usr/local/etc/redis/sentinel2.conf  --sentinel
		
		  sentinel3:
		    image: redis:6.0
		    container_name: sentinel3
		    ports:
		      - "26381:26379"
		    volumes:
		      - ./sentinel3:/usr/local/etc/redis
		    command: redis-sentinel /usr/local/etc/redis/sentinel3.conf  --sentinel
		```
- 创建sentinel节点
	1. 在当前目录执行

		```bash
		docker-compose up -d
		```
	2. 查看节点
	使用`docker-compose ps`查看如果都启动则代表成功
			![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/2c0d354caf30416fa555c8387fe0a4be.png)
- 检查是否监听到redis集群节点
 1. 进入sentinel容器，
	```bash
	docker exec -it sentinel1 /bin/bash
	```
2. 通过redis-cli 连接
`-p` 表示对应节点的端口
	```bash
	redis-cli -p 26379
	```
3. 输入`info Sentinel`查看节点信息
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/29a63061b2784d30afa6b11df1428294.png)

	> **info Sentinel相关重要信息解读**
	> - `sentinel_masters:1`：表示当前这个Sentinel实例正在监控1个主（master）Redis实例
	> - `master0`：监听的主节点信息
	> 	- `name=mymaster`：这里给出了正在监控的主实例的名称
	>   - `status=ok` ：这个主实例的状态是正常的；status字段并不是一个静态值，还可能会显示下面的错误的信息
	>   	- `status=disconnected`：表示Sentinel已经丢失了与该master或slave的连接。当主从连接出现网络问题或者实例宕机时，Sentinel无法与之通信就会显示这个状态
	>     - `status=down`：当Sentinel判断Redis实例（主或从）已经宕机或不可用时，该状态显示为down。**监听的主实例显示down，Sentinel会尝试进行故障转移。**
	>     - `status=reconnecting`：Sentinel试图重新连接到之前丢失连接的实例时，状态可能为reconnecting
	>     - `status=become_slave`：在故障转移过程中，新选举出的主实例可能被标记为become_slave，因为这个实例原本可能是从实例，在故障转移过程中被Sentinel提升为新的主实例
	>   - `address=192.168.192.35:6479` ：主实例的IP地址和端口
	>   - slaves=2：这个主实例有2个从（slave）实例
	>   - sentinels=3：有3个Sentinel实例（包括当前这个）在监控这个主实例

### 验证Sentinel功能
- 先把主节点关闭掉

	```bash
	docker stop redis-master
	```

- 查看Sentinel状态
可以看到appress的监听到的端口已经变为了重新选举后的主节点(slave1)的端口，并且status也还是表示正常
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5de6f2ad74224943883766dcd6523ebe.png)
- 测试写入
我们可以连接新的主节点查看是否可以正常写入，然后在通过`info Replication`查看节点信息，是否都正常
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/482ef447b7d045868969a04801e3b1bb.png)
- 测试读取
	1. slave2节点读取
	连接上slave2号节点，我们可以看到主节点的信息变为了6480，新的主节点的信息，并且数据可以正常读取到
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/18cbde41d1a94dd985d63814585be24d.png)
	2. master节点读取
	我们可以通过`docker start redis-master`在启动master节点，可以看到数据也是直接同步过来了
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/5d28eb1e677d4af0818f70aac08461b4.png)

- 监听日志
在redis-sentinel目录下的docker-compose文件位置执行`docker-compose logs`查看哨兵节点的日志可以查看具体刚刚执行的流程的不同阶段信息

	![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/477e3e03bc46489c82400addd7a7a8d9.png)

# 二、spring连接
## 1、添加依赖

加入对应的redis依赖
gradle依赖，maven改为对应的写法即可，我的spring boot版本是3.3.5，不同版本可能yml中的配置会有一点区别
```kotlin
    implementation("org.springframework.boot:spring-boot-starter-data-redis")
    implementation("com.fasterxml.jackson.core:jackson-databind")
```

## 2、添加配置
编写的配置文件的这个就是sentinel的master
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/a34e2441a2664169800c335845e981c0.png)
哨兵模式方式连接redis，我们只需要配置哨兵的连接即可，不需要去配置具体的redis节点信息
具体编写
```yaml
spring:
  data:
    redis:
    #密码
      password: 123456
      timeout: 5000
      sentinel:
      #对应前面sentinel的配置文件信息
        master: mymaster
       #三个哨兵的ip端口
        nodes: 192.168.31.121:26379,192.168.31.121:36379,192.168.31.121:46379
```


## 3、启动测试
可以
```kotlin
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.data.redis.core.RedisTemplate


@SpringBootTest
class PayDomainApplicationTests {

    @Autowired
    private lateinit var redisTemplate: RedisTemplate<String, Any>

    @Test
    fun set() {
        val set = redisTemplate.opsForValue().set("spring", "test")
        println("写入成功====>$set")

    }


    @Test
    fun get() {
        val get = redisTemplate.opsForValue().get("spring")
        println("读取成功====>$get")
    }

}

```
可以实现正常读取，表示没问题
![在这里插入图片描述](https://i-blog.csdnimg.cn/direct/adf8351d6a9a4ebf957049049929cbec.png)
