---
layout: post
title: websocket实现前端与后台全双工通信
category: 技术
tags: [websocket, 前端, 实时]
comments: true
---


##websocket实现前端与后台全双工通信

网站大多使用 HTTP 协议通信，而 HTTP 是无连接的协议。只有客户端请求时，服务器端才能发出相应的应答， HTTP 请求的包也比较大，如果只是很小的数据通信，开销过大。于是，我们可以使用 websocket 这个协议，用最小的开销实现面向连接的通信。

<!-- more -->

###例子，python和前端通信

```python

# _*_ coding:utf-8 _*_
__author__ = 'Patrick'
 
import socket
import threading
import sys
import os
import MySQLdb
import base64
import hashlib
import struct
  
# ====== config ======
HOST = 'localhost'
PORT = 3368
MAGIC_STRING = '258EAFA5-E914-47DA-95CA-C5AB0DC85B11'
HANDSHAKE_STRING = "HTTP/1.1 101 Switching Protocols\r\n" \
      "Upgrade:websocket\r\n" \
      "Connection: Upgrade\r\n" \
      "Sec-WebSocket-Accept: {1}\r\n" \
      "WebSocket-Location: ws://{2}/chat\r\n" \
      "WebSocket-Protocol:chat\r\n\r\n"
  
class Th(threading.Thread):
 def __init__(self, connection,):
  threading.Thread.__init__(self)
  self.con = connection
  
 def run(self):
  while True:
   try:
     pass
  self.con.close()
  
 def recv_data(self, num):
  try:
   all_data = self.con.recv(num)
   if not len(all_data):
    return False
  except:
   return False
  else:
   code_len = ord(all_data[1]) & 127
   if code_len == 126:
    masks = all_data[4:8]
    data = all_data[8:]
   elif code_len == 127:
    masks = all_data[10:14]
    data = all_data[14:]
   else:
    masks = all_data[2:6]
    data = all_data[6:]
   raw_str = ""
   i = 0
   for d in data:
    raw_str += chr(ord(d) ^ ord(masks[i % 4]))
    i += 1
   return raw_str
  
 # send data
 def send_data(self, data):
  if data:
   data = str(data)
  else:
   return False
  token = "\x81"
  length = len(data)
  if length < 126:
   token += struct.pack("B", length)
  elif length <= 0xFFFF:
   token += struct.pack("!BH", 126, length)
  else:
   token += struct.pack("!BQ", 127, length)
  #struct为Python中处理二进制数的模块，二进制流为C，或网络流的形式。
  data = '%s%s' % (token, data)
  self.con.send(data)
  return True
  
  
 # handshake
 def handshake(con):
  headers = {}
  shake = con.recv(1024)
  
  if not len(shake):
   return False
  
  header, data = shake.split('\r\n\r\n', 1)
  for line in header.split('\r\n')[1:]:
   key, val = line.split(': ', 1)
   headers[key] = val
  
  if 'Sec-WebSocket-Key' not in headers:
   print ('This socket is not websocket, client close.')
   con.close()
   return False
  
  sec_key = headers['Sec-WebSocket-Key']
  res_key = base64.b64encode(hashlib.sha1(sec_key + MAGIC_STRING).digest())
  
  str_handshake = HANDSHAKE_STRING.replace('{1}', res_key).replace('{2}', HOST + ':' + str(PORT))
  print str_handshake
  con.send(str_handshake)
  return True
  
def new_service():
 """start a service socket and listen
 when coms a connection, start a new thread to handle it"""
  
 sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
 try:
  sock.bind(('localhost', 3368))
  sock.listen(1000)
  #链接队列大小
  print "bind 3368,ready to use"
 except:
  print("Server is already running,quit")
  sys.exit()
  
 while True:
  connection, address = sock.accept()
  #返回元组（socket,add），accept调用时会进入waite状态
  print "Got connection from ", address
  if handshake(connection):
   print "handshake success"
   try:
    t = Th(connection, layout)
    t.start()
    print 'new thread for client ...'
   except:
    print 'start new thread error'
    connection.close()
  
  
if __name__ == '__main__':
 new_service()

```

前段代码

```javascript

var socket = new WebSocket('ws://localhost:3368');
ws.onmessage = function(result,nTime){
        alert("从服务端收到的数据:");
        alert("最近一次发送数据到现在接收一共使用时间:" + nTime);
        console.log(result);
}

var ws = new WebSocket(“ws://echo.websocket.org”);  
  
ws.send("Test");   //发送消息

ws.onopen = function(){ws.send(“Test!”); };  //成功链接
  
//ws.onmessage = function(evt){console.log(evt.data);ws.close();};     //消息到达
  
ws.onclose = function(evt){console.log(“WebSocketClosed!”);};   //关闭链接
  
ws.onerror = function(evt){console.log(“WebSocketError!”);};  //出错

```


