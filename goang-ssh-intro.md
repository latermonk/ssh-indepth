#  golang ssh intro 


https://xieys.club/webssh/



##  01 创建基本配置信息

```
	sshHost := "10.252.39.6"
	sshUser := "root"
	sshPasswrod := "root"
	sshType := "password"  // password或者key
	//sshKeyPath := "" // ssh id_rsa.id路径
	sshPort := 22
```

![0101](_image/0101.png)





##   dial 获取ssh client
![0201](_image/0201.png)



##  创建ssh-session
![0301](_image/0301.png)



##  执行远程命令
![0401](_image/0401.png)




