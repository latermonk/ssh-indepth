# golang第三方库-远程ssh


## 安装

设置goproxy环境变量为 set GOPROXY=https://goproxy.io

```
go mod init webssh
go get -u golang.org/x/crypto/ssh
```



## 示例

连接包含了认证，可以使用password或者sshkey 两种方式来认证

```
package main

import (
   "fmt"
   "golang.org/x/crypto/ssh"
   "io/ioutil"
   "log"
   "time"
)

func publicKeyAuthFunc(kPath string)ssh.AuthMethod{
   key,err := ioutil.ReadFile(kPath)
   if err != nil {
      log.Fatal("ssh key file read failed", err)
   }
   // Create the Signer for this private key.
   signer, err := ssh.ParsePrivateKey(key)
   if err != nil {
      log.Fatal("ssh key signer failed", err)
   }
   return ssh.PublicKeys(signer)
}

func main()  {
   //可以使用 password 或者 sshkey 2种方式来认证。
   sshHost := "192.168.1.58" // 主机名
   sshUser := "root"     //用户名
   sshPassword := "120110" //密码
   sshType := "password"   //ssh认证类型
   sshKeyPath := ""        //ssh id_rsa.id路径
   sshPort := 22

   //创建ssh登陆配置
   config := &ssh.ClientConfig{
      Timeout: time.Second, //ssh 连接timeout时间一秒钟，如果ssh验证错误 会在1秒内返回
      User: sshUser, //指定ssh连接用户
      HostKeyCallback: ssh.InsecureIgnoreHostKey(), //这个可以，但是不够安全
   }

   if sshType == "password"{
      config.Auth = []ssh.AuthMethod{ssh.Password(sshPassword)}
   }else {
      config.Auth = []ssh.AuthMethod{publicKeyAuthFunc(sshKeyPath)}
   }

   //dial获取ssh Client
   addr := fmt.Sprintf("%s:%d",sshHost,sshPort)
   sshClient,err := ssh.Dial("tcp",addr,config)
   if err != nil {
      log.Fatal("创建ssh client 失败",err)
   }
   defer sshClient.Close()

   //创建ssh-session
   session,err := sshClient.NewSession()
   if err != nil{
      log.Fatal("创建ssh session 失败",err)
   }
   defer  session.Close()

   //执行远程命令
   combo,err := session.CombinedOutput("whoami; cd /; ls -al")
   if err != nil{
      log.Fatal("远程执行cmd 失败",err)
   }
   log.Println("命令输出:",string(combo))
}
```



### 代码详解

#### 1 配置ssh.ClientConfig

- 建议Timeout自定义一个比较短的时间
- 自定义HostKeyCallback 如果想简便使用就使用 ssh.InsecureIgnoreHostKey回调，这种方式不是很安全
- publicKeyAuthFunc 如果使用key登陆，就需要用这个函数来读取id_rsa私钥，当然你可以自定义这个访问让他支持字符串



#### 2 ssh.Dial创建ssh客户端

拼接字符串得到ssh连接地址,同时不要忘记 defer client.Close()



#### 3 sshClient.NewSession 创建session会话

- 可以自定义stdin,stdout
- 可以创建pty
- 可以SetEnv



### 执行命令的方式

#### CombinedOutput

```
func (s *Session) CombinedOutput(cmd string) ([]byte, error)
```

在远程主机上运行CMD命令，标准输出和标准错误都通过`[]byte`返回，否则就返回 `nil, errors.New("ssh: Stdout already set")`

每个session下只能调用一次



#### Run

```
func (s *Session) Run(cmd string) error
```

在远程主机上运行CMD命令，不关心命令执行结果

每个session下只能调用一次



#### Output

同CombinedOutput作用一样

```
func (s *Session) Output(cmd string) ([]byte, error)
```



#### 模拟交互terminal

```
func main()  {
	//可以使用 password 或者 sshkey 2种方式来认证。
	sshHost := "192.168.1.58" // 主机名
	sshUser := "root"     //用户名
	sshPassword := "120110" //密码
	sshType := "password"   //ssh认证类型
	sshKeyPath := ""        //ssh id_rsa.id路径
	sshPort := 22
	//fmt.Print("请输入主机地址:")
	//fmt.Scanln(&sshHost)
	//fmt.Print("请输入主机端口:")
	//fmt.Scanln(&sshPort)
	//fmt.Print("请输入主机用户:")
	//fmt.Scanln(&sshUser)
	//fmt.Print("请输入主机密码:")
	//fmt.Scanln(&sshPassword)

	//创建ssh登陆配置
	config := &ssh.ClientConfig{
		Timeout: time.Second, //ssh 连接timeout时间一秒钟，如果ssh验证错误 会在1秒内返回
		User: sshUser, //指定ssh连接用户
		//HostKeyCallback: ssh.InsecureIgnoreHostKey(), //这个可以，但是不够安全
		HostKeyCallback: func(hostname string, remote net.Addr, key ssh.PublicKey) error {
			return nil
		},
	}

	if sshType == "password"{
		config.Auth = []ssh.AuthMethod{ssh.Password(sshPassword)}
	}else {
		config.Auth = []ssh.AuthMethod{publicKeyAuthFunc(sshKeyPath)}
	}

	//dial获取ssh Client
	addr := fmt.Sprintf("%s:%d",sshHost,sshPort)
	sshClient,err := ssh.Dial("tcp",addr,config)
	if err != nil {
		log.Fatal("创建ssh client 失败",err)
	}
	defer sshClient.Close()


	//创建ssh-session
	session,err := sshClient.NewSession()
	if err != nil{
		log.Fatal("创建ssh session 失败",err)
	}
	defer  session.Close()
	//将当前终端的stdin文件句柄设置给远程给远程终端，这样就可以使用tab键
	fd := int(os.Stdin.Fd())
	state ,err := terminal.MakeRaw(fd)
	if err != nil{
		panic(err)
	}
	defer terminal.Restore(fd,state)


	session.Stdout = os.Stdout // 会话输出关联到系统标准输出设备
	session.Stderr = os.Stderr  // 会话错误输出关联到系统标准错误输出设备
	session.Stdin = os.Stdin   // 会话输入关联到系统标准输入设备

	//设置终端模式
	modes := ssh.TerminalModes{
		ssh.ECHO: 0, //禁止回显 （0 禁止,1 启动）
		ssh.TTY_OP_ISPEED: 14400, // input speed = 14.4kbaud
		ssh.TTY_OP_OSPEED: 14400, //output speed = 14.4kbaud
	}

	// 请求伪终端
	if err = session.RequestPty("linux",32,160,modes);err != nil{
		log.Fatalf("request pty error: %s", err.Error())
	}
	
	//启动远程shell
	if err = session.Shell(); err != nil {
		log.Fatalf("start shell error: %s", err.Error())
	}
	
	//等待远程命令（终端）退出
	if err = session.Wait(); err != nil {
		log.Fatalf("return error: %s", err.Error())
	}
}
```

```
func (s *Session) RequestPty(term string, h, w int, termmodes TerminalModes) error

term为linux环境变量，可通过/usr/share/terminfo/x 查看看所有类型。具体可看
https://blog.csdn.net/ly890700/article/details/53229063?locationNum=2&fps=1
```



注意：

这里的ssh.InsecureIgnoreHostKey是不检查host key，需要检查的话得参考client源码重写函数

