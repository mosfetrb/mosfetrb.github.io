---
layout: page
title:  "计算机网络Chap2套接字(Ruby实现)"
author: mosfet
category: miscellaneous
tags: Tetris
---
程序如下：

①客户端从键盘读取一行字符(作为数据)并将数据发送到服务器。  
②服务器接收数据并将字符转换为大写。  
③服务器将修改后的数据发送给客户端。  
④客户端接收修改后的数据并在屏幕上显示该行。  

## UDP
```ruby
# https://docs.ruby-lang.org/ja/latest/class/Socket.html
# https://docs.ruby-lang.org/ja/latest/class/Socket.html#I_RECVFROM
require 'socket'
server_addr = Socket.sockaddr_in(12000, "127.0.0.1")

# 服务器套接字
s2 = Socket.new(Socket::AF_INET, Socket::SOCK_DGRAM)
s2.bind(server_addr)
puts("The server is ready to receive")
while true
  msg, sender_addr = s2.recvfrom(128)
  puts Socket.unpack_sockaddr_in(sender_addr)
  s2.send("u sent me: #{msg}", 0, sender_addr)
end

# 客户端套接字
s1 = Socket.new(Socket::AF_INET, Socket::SOCK_DGRAM)
s1.send("hello?", 0, server_addr)
puts s1.recv(128)
s1.close

puts "client closed" if s1.closed?
```

## TCP
```ruby
# https://docs.ruby-lang.org/ja/latest/class/Socket.html#I_CONNECT
# https://docs.ruby-lang.org/ja/latest/class/Socket.html#I_LISTEN
# https://docs.ruby-lang.org/ja/latest/class/Socket.html#I_ACCEPT
require 'socket'
server_addr = Socket.sockaddr_in(12000, "127.0.0.1")

# 服务器套接字
s2 = Socket.new(Socket::AF_INET, Socket::SOCK_STREAM)
s2.bind(server_addr)
s2.listen(1)
puts("The server is ready to receive")
while true
  connection_sock, sender_addr = s2.accept()
  puts Socket.unpack_sockaddr_in(sender_addr)
  msg = connection_sock.recv(128)
  connection_sock.send("u sent me: #{msg}", 0)
  connection_sock.close
end

# 客户端套接字
s1 = Socket.new(Socket::AF_INET, Socket::SOCK_STREAM)
s1.connect(server_addr)
s1.send("hello?", 0)
puts "From Server: " + s1.recv(128)
s1.close

puts "ur closed." if s1.closed?
```