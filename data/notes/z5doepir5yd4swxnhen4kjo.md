
[Tcl commands](https://www.tcl.tk/man/tcl/TclCmd/contents.html)
[Tcl tutorial](https://www.tcl.tk/man/tcl8.5/tutorial/tcltutorial.html)

```
#!/usr/bin/expect -f

set suffix [lindex $argv 0]
if { [string length $suffix] < 1 } {
  puts "target vm is required"
  exit 1
}

set     ip "192.168.122."
append  ip "$suffix"

set     userid root
set     passwd root
set     timeout 10

set prompt "\[root@sylixos:/root\]# "

puts   "Telnet $ip ..."

spawn telnet $ip
    expect "login: "
    send "$userid\r"

    expect "password: "
    send "$passwd\r"

    interact
```