ExBuffer，NodeJs的TCP中的粘包、分包问题的解决方案！

推荐结合ByteBuffer来做通信协议！https://github.com/play175/ByteBuffer

C版本的ExBuffer：https://github.com/play175/exbuffer.c

参考Node端对TCP Socket的封装库（TLV格式）：[node-socket](https://github.com/xiaoping6688/node-socket)
```
npm install --save node-socket
```

```javascript
var ExBuffer = require('./ExBuffer');

/*********************** TLV包结构 示列 ********************************/
exBuffer = new ExBuffer().int8Tag().uint32Head().bigEndian()
exBuffer.on('data', onReceivePackData)

socket.on('data', function (data) {
  exBuffer.put(data)
})

function onReceivePackData(data) {
  var tag = data.tag
  var value = data.value ? data.value.toString() : null
  debuger('[Received] tag: ' + tag + ' value: ' + value)

  if (value){
    value = JSON.parse(value)
  }

  if (typeof receiveCallback === "function"){
    receiveCallback(tag, value)
  }
}

/**
 * Send data to server
 * @param cmd means tag
 * @parma arg means value
 */
function send(cmd, arg){
  debuger('[Send] tag: ' + cmd + ' value: ' + (arg ? JSON.stringify(arg) : ''))
  if (client){
    var data = null, len = 0
    if (arg){
      data = JSON.stringify(arg)
      len = Buffer.byteLength(data)
    }

    // TLV pack:
    // Write the tag: use one byte
    var tagBuf = Buffer.alloc(1)
    tagBuf.writeInt8(cmd, 0)
    client.write(tagBuf)

    // Write the length of value: use four bytes
    var headBuf = Buffer.alloc(4)
    headBuf.writeUInt32BE(len, 0)
    client.write(headBuf)

    // Then, write the value
    if (len > 0){
      var bodyBuf = Buffer.alloc(len)
      bodyBuf.write(data)
      client.write(bodyBuf)
    }
  } else {
    debuger("The socket has not connected yet.")
  }
}

/*************************基本操作****************************/

//构造一个ExBuffer，采用4个字节（uint32无符号整型）表示包长，而且是little endian 字节序
var exBuffer = new ExBuffer().uint32Head().littleEndian();
//或者构造一个ExBuffer，采用2个字节（ushort型）表示包长，而且是big endian 字节序 (默认)
var exBuffer = new ExBuffer().ushortHead().bigEndian();

//只要收到满足的包就会触发事件
exBuffer.on('data',function(buffer){
    console.log('>> receive data,length:'+buffer.length);
    //console.log(buffer);
});


//传入一个9字节长的数据，分多次put （对应于TCP中的分包的情况）
exBuffer.put(new Buffer([0,9]));
exBuffer.put(new Buffer([1,2,3,4,5,6,7]));
exBuffer.put(new Buffer([8,9]));

//传入一个3个字节的数据和一个6个字节的数据，一次put（对应于TCP中的粘包的情况）
exBuffer.put(new Buffer([0,3,1,2,3,0,6,1,2,3,4,5,6]));


//大数据处理测试 (20MB)
var exBuffer = new ExBuffer().uint32Head().bigEndian();
exBuffer.on('data',function(buffer){
    console.log('>> receive data,length:'+buffer.length);
    console.log(buffer);
});
var sbuf = new Buffer(4);
sbuf.writeUInt32BE(1024*1024*20,0);//写入包长
exBuffer.put(sbuf);
exBuffer.put(new Buffer(1024*1024*20));


/*************************在socket中的应用****************************/

console.log('-----------------------use in socket------------------------');

var net = require('net');

//测试服务端
var server = net.createServer(function(socket) {
  console.log('client connected');
  new Connection(socket);//有客户端连入时
});
server.listen(8124);

//服务端中映射客户端的类
function Connection(socket) {
    var exBuffer = new ExBuffer();
    exBuffer.on('data',onReceivePackData);

    socket.on('data', function(data) {
        exBuffer.put(data);//只要收到数据就往ExBuffer里面put
    });

    //当服务端收到完整的包时
    function onReceivePackData(buffer){
        console.log('>> server receive data,length:'+buffer.length);
        console.log(buffer.toString());

        var data = 'wellcom, I am server';
        var len = Buffer.byteLength(data);

        //写入2个字节表示本次包长
        var headBuf = new Buffer(2);
        headBuf.writeUInt16BE(len, 0)
        socket.write(headBuf);

        var bodyBuf = new Buffer(len);
        bodyBuf.write(data);
        socket.write(bodyBuf);
    }
}

//测试客户端
var exBuffer = new ExBuffer();
var client = net.connect(8124, function() {

  var data = 'hello I am client';
  var len = Buffer.byteLength(data);

  //写入2个字节表示本次包长
  var headBuf = new Buffer(2);
  headBuf.writeUInt16BE(len, 0)
  client.write(headBuf);

  var bodyBuf = new Buffer(len);
  bodyBuf.write(data);
  client.write(bodyBuf);
  
});

client.on('data', function(data) {
  exBuffer.put(data);//只要收到数据就往ExBuffer里面put
});

//当客户端收到完整的数据包时
exBuffer.on('data', function(buffer) {
    console.log('>> client receive data,length:'+buffer.length);
    console.log(buffer.toString());
});

```

安装

```javascript
npm install ExBuffer
```
