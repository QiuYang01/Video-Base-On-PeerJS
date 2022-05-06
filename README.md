> 建议先过一遍之前的[这篇文章](https://blog.csdn.net/qq_43548154/article/details/124554150)，很多基础的api本文将不再介绍。

本文涉及到的新api为peer.call()

```bash
// 发送端
var call = peer.call('dest-peer-id',mediaStream);

 //接收端
peer.on('call', function(call) {
	// Answer the call, providing our mediaStream
	call.answer(mediaStream);
});
  
call.on('stream', function (stream) {
   console.log('received remote stream');
   remoteVideo.srcObject = stream;                            
});
```
其中mediaStream可以通过以下代码获得：

```bash
var localStream  = null;
const constraints = {
   audio: true,
   video: { width: 320, deviceId: videoSource ? { exact: videoSource } : undefined }
};
navigator.mediaDevices
    .getUserMedia(constraints)
    .then(getStream)
    .catch(handleError);
//获取流
function getStream(stream) {
    console.log('received local stream');
    localStream = stream;
    //localVideo.srcObject = localStream; //显示到页面中的video标签
    // console.log(`localStream${stream}`)
}
//获取设备错误
function handleError(error) {
    console.log('navigator.MediaDevices.getUserMedia error: ', error.message, error.name);
}
```

由于代码我昨天晚上写的，由于各种琐事导致我上午没有写此文档，现在已经下午4点了。我不记得昨天写下列代码具体的逻辑了，功能是完整的。
因此，我就直接把代码列出来，代码是可以直接运行的（需要引入的js和css文件再我github仓库中有，链接放在文末）。
> 效果截图

![在这里插入图片描述](https://img-blog.csdnimg.cn/61b87828a4194b8cb8d20bf58653886c.png)


> **代码：videoself.html**

```bash
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>视频聊天</title>
  <link rel="stylesheet" type="text/css" href="css/common.css" />
  <link rel="stylesheet" type="text/css" href="css/jquery-ui.css" />
  <!-- <script src="https://unpkg.com/peerjs@1.3.2/dist/peerjs.min.js"></script> -->
  <script src="./js/peerjs.min.js"></script>
  <script src="./js/jquery-1.12.4.js"></script>
  <script src="./js/jquery-ui.js"></script>
  <script>
    var peer = null; 
    var startId = null; 
    var receiveId = null;
    var conn = null;
    var localStream = null;
    
 
    //获取url后面的参数
    getRequest = function() { 
        var url = location.search; 
        var theRequest = new Object(); 
        if (url.indexOf("?") != -1) { 
            let str = url.substr(1); 
            let strs = str.split("&"); 
            for(let i = 0; i < strs.length; i++) { 
                theRequest[strs[i].split("=")[0]]=unescape(strs[i].split("=")[1]); 
            } 
        } 
        return theRequest; 
    }

    //绑定摄像头列表到下拉框
    function getDevices(deviceInfos) {
        if (deviceInfos===undefined){
            return
        }
        console.log(`设备信息${deviceInfos}`)
        for (let i = 0; i !== deviceInfos.length; ++i) {
          // console.log(deviceInfos[i])
          const deviceInfo = deviceInfos[i];
          const option = document.createElement('option');
          option.value = deviceInfo.deviceId;
          if (deviceInfo.kind === 'videoinput') {
              option.text = deviceInfo.label || `camera ${videoSelect.length + 1}`;
              console.log(option)
              videoSelect.appendChild(option);
          }
        }
    }

    //获取设备错误
    function handleError(error) {
        console.log('navigator.MediaDevices.getUserMedia error: ', error.message, error.name);
    }

    //获取流
    function getStream(stream) {
        console.log('received local stream');
        localStream = stream;
        localVideo.srcObject = localStream; //显示到页面
        // console.log(`localStream${stream}`)
    }

    //开启本地摄像头
    function start() {
        if (localStream) { //停止
            localStream.getTracks().forEach(track => {
                track.stop();
            });
        }
        const videoSource = videoSelect.value;
        const constraints = {
            audio: true,
            video: { width: 6020, deviceId: videoSource ? { exact: videoSource } : undefined }
        };
        navigator.mediaDevices
            .getUserMedia(constraints)
            .then(getStream)
            .then(getDevices)
            .catch(handleError);
    }
    
    //封装发送消息
    function sendMessage(from, to, action) {
        var message = { "from": from, "to": to, "action": action };
        if (!conn) {
            conn = peer.connect(receiveId);
            conn.on('open', () => {
                conn.send(JSON.stringify(message));
                console.log("发消息",message);
            });
        }
        if (conn.open){
            conn.send(JSON.stringify(message));
            console.log(message);
        }
    }
    
    //当此端改变摄像头 请求那边重新call一次 不然画面传不过去
    function pleaseCallMe(){
      sendMessage(startId, receiveId, "pleaseCallMe");
    }


    //页面加载后 
    window.onload = function () {
      var videoSelect = document.querySelector("select#videoSelect")
      var localVideo = document.querySelector("video#localVideo");
      var remoteVideo = document.querySelector("video#remoteVideo");
      var lblFrom = document.querySelector("label#lblFrom");
      var tip = document.querySelector("#tip");
      if (!navigator.mediaDevices ||
          !navigator.mediaDevices.getUserMedia) {
          console.log('webrtc is not supported!');
          alert("webrtc is not supported!");
          return;
      }

      //获取摄像头列表
      navigator.mediaDevices
        .enumerateDevices()
        .then(getDevices)
        .catch(handleError);

      //改变摄像头重新获取流
      videoSelect.onchange = function(){
        start();
        pleaseCallMe(); //让那边重新call 就能同步画面
      }; 
      start(); //默认直接获取流

      $("#dialog-confirm").hide(); //隐藏接收提示框


      //每次使用同一个startId
      if(localStorage.getItem("startId")==null){
        startId = new Date().getTime();
        localStorage.setItem("startId",startId)
      }else{
        startId = localStorage.getItem("startId")
      }

      peer = new Peer(startId);
      console.log(peer)

      peer.on('open', function(id) {
        startId = id;
        console.log('My peer ID is: ' + id);
        console.log(`接收端链接${location.href.replace("self","target")}?startId=${startId}`)
        document.getElementById("receivedUrl").innerHTML = `<a target="_blank" href="${location.href.replace("self","target")}?startId=${startId}">${location.href.replace("self","target")}?startId=${startId}</a>` 
      });

      peer.on('call', function (call) {
        console.log("oncall")
        call.answer(localStream);
      });

      peer.on('connection', function(conn) {
        console.log("被连接",conn);
        receiveId = conn.peer;
        conn.on('data', (data) => {
            var msg = JSON.parse(data);
            console.log("收到消息",msg);
            //收到视频邀请时，弹出询问对话框
            if (msg.action === "call") {
                console.log("call")
                tip.innerHTML = "被别人call"
                lblFrom.innerText = msg.from;
                $("#dialog-confirm").dialog({
                    resizable: false,
                    height: "auto",
                    width: 400,
                    modal: true,
                    buttons: {
                        "Accept": function () {
                            $(this).dialog("close");
                            sendMessage(msg.to, msg.from, "accept");
                        },
                        Cancel: function () {
                            $(this).dialog("close");
                        }
                    }
                });
            }

            //接受视频通话邀请后，通知另一端    
            if (msg.action === "accept-ok") {
                console.log("accept-ok call => " + JSON.stringify(msg));
                var call = peer.call(receiveId, localStream);
                call.on('stream', function (stream) {
                    console.log('received remote stream');
                    remoteVideo.srcObject = stream;                            
                });
                tip.innerHTML = "已连接"
            }
        });
      });

    }
  </script>
</head>
<body>
  <div class="container" style="text-align: center;">
    <h1>基于PeerJS的点对点视频聊天程序</h1>
    <p>
      <span style="font-weight: 700;">切换设备</span>
      <select id="videoSelect" style="width:100px"></select>
    </p>
    <p>
      把下方链接发送给你对象即可开始视频啦<br />
      <span>对象链接：</span><span id="receivedUrl">生成中...</span>
    </p>
    <div style="display: flex;flex-flow: wrap;justify-content: center;">
      <div style="border: 5px solid #ccc;">
        <h4>你的画面</h4>
        <video id="localVideo" autoplay controls></video>
      </div>
      <div style="border: 5px solid #ccc;">
        <h4>对象画面</h4>
        <video id="remoteVideo" autoplay controls></video>
      </div>
    </div>
    
    <div id="dialog-confirm" title="接受视频通话吗?">
        <p>
          <span class="ui-icon ui-icon-info" style="float:left; margin:12px 12px 20px 0;"></span><label id="lblFrom"></label>
          打你，是否接受？
        </p>
    </div>
    <div id="tip">
      等待分享链接给对象...
    </div>
  </div>
</body>

</html>
```

> 代码：videotarget.html

```bash
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta http-equiv="X-UA-Compatible" content="IE=edge">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>视频聊天</title>
  <link rel="stylesheet" type="text/css" href="css/common.css" />
  <link rel="stylesheet" type="text/css" href="css/jquery-ui.css" />
  <!-- <script src="https://unpkg.com/peerjs@1.3.2/dist/peerjs.min.js"></script> -->
  <script src="./js/peerjs.min.js"></script>
  <script src="./js/jquery-1.12.4.js"></script>
  <script>
    var peer = null; 
    var startId = null; 
    var receiveId = null;
    var conn = null;
    var localStream = null;
    

    //获取url后面的参数
    getRequest = function() { 
        var url = location.search; 
        var theRequest = new Object(); 
        if (url.indexOf("?") != -1) { 
            let str = url.substr(1); 
            let strs = str.split("&"); 
            for(let i = 0; i < strs.length; i++) { 
                theRequest[strs[i].split("=")[0]]=unescape(strs[i].split("=")[1]); 
            } 
        } 
        return theRequest; 
    }

    //绑定摄像头列表到下拉框
    function getDevices(deviceInfos) {
        if (deviceInfos===undefined){
            return
        }
        console.log(`设备信息${deviceInfos}`)
        for (let i = 0; i !== deviceInfos.length; ++i) {
          // console.log(deviceInfos[i])
          const deviceInfo = deviceInfos[i];
          const option = document.createElement('option');
          option.value = deviceInfo.deviceId;
          if (deviceInfo.kind === 'videoinput') {
              option.text = deviceInfo.label || `camera ${videoSelect.length + 1}`;
              console.log(option)
              videoSelect.appendChild(option);
          }
        }
    }

    //获取设备错误
    function handleError(error) {
        console.log('navigator.MediaDevices.getUserMedia error: ', error.message, error.name);
    }

    //获取流
    function getStream(stream) {
        console.log('received local stream');
        localStream = stream;
        localVideo.srcObject = localStream; //显示到页面
        // console.log(`localStream${stream}`)
    }

    //开启本地摄像头
    function start() {
        if (localStream) { //停止
            localStream.getTracks().forEach(track => {
                track.stop();
            });
        }
        const videoSource = videoSelect.value;
        const constraints = {
            audio: true,
            video: { width: 320, deviceId: videoSource ? { exact: videoSource } : undefined }
        };
        navigator.mediaDevices
            .getUserMedia(constraints)
            .then(getStream)
            .then(getDevices)
            .catch(handleError);
        if(conn){
            sendMessage(receiveId, start, "accept-ok");
        }
    }

    //封装发送消息
    function sendMessage(from, to, action) {
        var message = { "from": from, "to": to, "action": action };
        if (!conn) {
            conn = peer.connect(hashCode(to));
            conn.on('open', () => {
                conn.send(JSON.stringify(message));
                console.log(message);
            });
        }
        if (conn.open){
            conn.send(JSON.stringify(message));
            console.log(message);
        }
    }

    //页面加载后 
    window.onload = function () {
      var videoSelect = document.querySelector("select#videoSelect")
      var localVideo = document.querySelector("video#localVideo");
      var remoteVideo = document.querySelector("video#remoteVideo");
      var tip = document.querySelector("#tip");
      if (!navigator.mediaDevices ||
          !navigator.mediaDevices.getUserMedia) {
          console.log('webrtc is not supported!');
          alert("webrtc is not supported!");
          return;
      }

      //获取摄像头列表
      navigator.mediaDevices
        .enumerateDevices()
        .then(getDevices)
        .catch(handleError);

      //改变摄像头重新获取流 从新call一遍
      videoSelect.onchange = function(){
          start();
          sendMessage(receiveId,startId,"call");
      }
      start(); //默认直接获取流

      $("#dialog-confirm").hide(); //隐藏接收提示框


      //获取参数 获取发起对话者的id 用户建立连接
      param = getRequest();
      startId = param.startId;

      if(!localStorage.getItem("receiveId")){
        receiveId = new Date().getTime();
        localStorage.setItem("receiveId",receiveId)
      }else{
        receiveId = localStorage.getItem("receiveId")
      }
      peer = new Peer(receiveId); //第一个参数为id 第二个参数为选项 ('',{"debug":3})
      console.log(peer)
      peer.on('open', function(id) {
        receiveId = id;
        console.log('My peer ID is: ' + id);
        //这里先向start建立连接，让start知道这个端的id（reveivedId）
        conn = peer.connect(startId);
        conn.on('open', () => {
              //直接发送消息
              console.log("发送消息")
              sendMessage(receiveId,startId,"call")
              tip.innerHTML = "等待接听..."
        });
      });
      
      peer.on('connection', function(conn) {
        console.log("被连接",conn);
        conn.on('data', (data) => {
            console.log("收到消息",data);
        });
      });

      peer.on('call', function (call) {
        console.log("oncall")
        call.answer(localStream);
      });

  
      peer.on('connection', function(conn) {
        console.log("被连接",conn);
        receiveId = conn.peer;
        conn.on('data', (data) => {
            var msg = JSON.parse(data);
            console.log("收到消息",msg);
            
            //画面卡住了 重新call
            if(msg.action === "pleaseCallMe"){
                sendMessage(receiveId,startId,"call")
            }

            //接受视频通话邀请
            if (msg.action === "accept") {
                console.log("accept")
                console.log("accept call => " + JSON.stringify(msg));
                var call = peer.call(receiveId, localStream);
                call.on('stream', function (stream) {
                    console.log('received remote stream');
                    remoteVideo.srcObject = stream;
                    tip.innerHTML = "已连接!"
                    sendMessage(msg.to, msg.from, "accept-ok");
                });
            }
        });
      });

      

    }
  </script>
</head>
<body> 
    <div class="container" style="text-align: center;">
        <h1>基于PeerJS的点对点视频聊天程序</h1>
        <p>
          <span style="font-weight: 700;">切换设备</span>
          <select id="videoSelect" style="width:100px"></select>
        </p>
        <div style="display: flex;flex-flow: wrap;justify-content: center;">
          <div style="border: 5px solid #ccc;">
            <h4>你的画面</h4>
            <video id="localVideo" autoplay controls></video>
          </div>
          <div style="border: 5px solid #ccc;">
            <h4>对象画面</h4>
            <video id="remoteVideo" autoplay controls></video>
          </div>
        </div>
        <div id="dialog-confirm" title="接受视频通话吗?">
            <p>
              <span class="ui-icon ui-icon-info" style="float:left; margin:12px 12px 20px 0;"></span><label id="lblFrom"></label>
              打你，是否接受？
            </p>
        </div>
        <div id="tip">
            连接中...
        </div>
      </div>
  
</body>
</html>
```
> 源码地址：[https://github.com/QiuYang01/Video-Base-On-PeerJS](https://github.com/QiuYang01/Video-Base-On-PeerJS)

>预览地址：[https://map.gnnu.work/rm21/qy/video/videoself.html](https://map.gnnu.work/rm21/qy/video/videoself.html)