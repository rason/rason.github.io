title: Jenkins通过SSH重启远程Tomcat失败
categories: Other
date: 2016-12-29 11:04:03
tags: Jenkins Tomcat
description: Jenkins SSH restart Tomcat failed
---

## 前言

在[上篇文章](http://rason.me/2016/12/28/Gitlab-and-jenkins-CI/)中，我们通过`publish over ssh`插件来进行发布操作。然而遇到了一个坑，在`Exec command`中配置启动脚本，代码传输完毕之后确实是执行了，也把Tomcat启动了。但是在构建完成之后，随着ssh链接的结束，由ssh启动的Tomcat进程也被杀掉了。


## 解决方法

谷歌了一下，有的建议使用BUILD_ID环境变量，有的建议使用`nohup`，但在我的机器上都不奏效。最后使用`expect`脚本来启动成功了。

```
#!/usr/local/bin/expect  -- 
set timeout 3000
set passwd [lindex $argv 2]

if { [llength $argv] < 4} {
  puts "usage: $argv0 ip user pwd cmd"  
  exit 1
}

proc wait {} {
    expect {
        "$*" { puts "\n" }
        ">*" { puts "\n" }
        "#*" { puts "\n" }
        "*assword*" {send "\n"; puts "\nPassword error" ; exit 1}
    }
}

#spawn	/usr/bin/ssh  -q -p22 [lindex $argv 0]@[lindex $argv 1] "[lindex $argv 2]"
spawn	/usr/bin/ssh  -q -p22 [lindex $argv 1]@[lindex $argv 0] "[lindex $argv 3]"
expect {
    "*assword*"  {
         send "$passwd\n"
         set timeout 300
         expect "*assword*" { send "qhee@temp\n";wait }
    }
    "*no)?*"  {
        send "yes\n"
        set timeout 300
        expect {
            "*assword*" {
                set timeout 300
                send "$passwd\n"
                expect "*assword*" { send "qhee@temp\n"; wait }
            }
            "*no)?*"  {
                send "yes\n"
                set timeout 300
                expect {
                    "*assword*" {
                        send "$passwd\n"
                        expect "*assword*" { send "qhee@temp\n"; wait }
                    }
                }
            }
        }
    }
    timeout {
        puts "\nTimeout"
        exit 2
    }
}
#puts "\nCommand success"
```

使用方法`expect xxx.exp ip user pwd cmd`，如果你没有expect则要先安装一下。
