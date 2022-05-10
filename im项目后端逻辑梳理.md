# im项目后端逻辑梳理

## 接口层

- 10000端口
- 接口层处理ReqParams，ResParams

### auth

认证相关

#### auth/user_register

- 用户注册
- 调用authRPC的user_register函数和user_token函数



#### auth/user_token

- 用户登录，获取token
- token负载为uid和platform
- 调用authRPC的user_token函数



### user

用户信息相关

#### user/update_user_info

- 更新用户信息

#### user/get_users_info

- 获取用户信息

#### user/get_self_user_info

- 获取自己的用户信息



### friend

#### friend/add_friend

- 添加好友

#### friend/delete_friend



### Message(Chat)

#### /msg/newest_seq



#### /msg/send_msg



#### /msg/pull_msg_by_seq







## 逻辑层

### rpcAuth

#### user_register

- 调用`user_model`的`UserRegister`方法，增加一个用户

#### user_token

- 调用`user_model`的`FindUserByUID`方法，能查到则生成token返回

### rpcUser

### rpcFriend

#### AddFriend

1. 检查是否有权限发送请求
2. 调用user表getUserById，检查要添加的用户是否存在
3. 写入frien_request表
4. 调用msgRPC的FriendApplicationNotification（但是这个地方不是通过rpc调用，是import了msgRPC，最后再通过msgRPC的SendMsg发送消息）



### rpcMsg

#### SendMsg



#### GetMaxAndMinSeq

- 从根据uid从redis查询最大、最小seq









## 数据层

### user_model

| uid  | name | icon | gender | mobile | birth | email | ex   | create_time |
| ---- | ---- | ---- | ------ | ------ | ----- | ----- | ---- | ----------- |
|      |      |      |        |        |       |       |      |             |

#### UserRegister

- Create() 增

#### FindUserByUID

- 查找uid=?的用户



## msg_gateway

- webSocket端口17778，用于与客户端连接
- rpc端口10400，用于被服务端调用，向客户端发送信息



## marshal

- unmarshal

  将byte数据变为语言结构

- 



## Open-IM-SDK-Core

### 为什么会有这个

- open-im-sdk-core会有一个本地数据库
- 如果是在web运行，这个是运行在linux服务器，作为一个接入层
- 如果是在app上运行，这个sdk是装在设备本地的

### 入口 ws_wrapper/cmd/open_im_sdk_server

```go
ws_local_server.WS.OnInit(*sdkWsPort)
ws_local_server.WS.Run()
```



### 入口逻辑 ws_wrapper/ws_local_server

- 入口 ws_server

```go
type WServer struct {
	wsAddr       string
	wsMaxConnNum int
	wsUpGrader   *websocket.Upgrader
	wsConnToUser map[*UserConn]map[string]string
	wsUserToConn map[string]map[string]*UserConn
	ch           chan ChanMsg
}
```

- 调用关系

```go
func (ws *WServer) Run() {
  http.HandleFunc("/", ws.wsHandler)
}
```

```go
type UserConn struct {
	*websocket.Conn
	w *sync.Mutex
}

func (ws *WServer) wsHandler(w http.ResponseWriter, r *http.Request) {
  conn, err := ws.wsUpGrader.Upgrade(w, r, nil)
  newConn := &UserConn{conn, new(sync.Mutex)}
  go ws.readMsg(newConn)
}
```

```go
func (ws *WServer) readMsg(conn *UserConn) {
  msgType, msg, err := conn.ReadMessage()
  ws.msgParse(conn, msg)
}
```



- handle_func

```go
func (ws *WServer) msgParse(conn *UserConn, jsonMsg []byte) {
  m := Req{}
  json.Unmarshal(jsonMsg, &m)
  // 登录时，通过 GenUserRouterNoLock 将 RefRouter 保存在 UserRouteMap
  if m.ReqFuncName == "Login" {
    GenUserRouterNoLock(m.UserID)
  }
  // 从 UserRouteMap 取出 RefRouter
  urm, ok := UserRouteMap[m.UserID]
  
  // 调用对应方法
  parms := []reflect.Value{reflect.ValueOf(m.Data), reflect.ValueOf(m.OperationID)}
	vf, ok := (*urm.refName)[m.ReqFuncName]
  vf.Call(parms)
}
```

- ws_handle

```go
type WsFuncRouter struct {
	uId string
}

func GenUserRouterNoLock(uid string) *RefRouter {
  // RouteMap1保存 wsRouter1 funcName -> func 的映射
  RouteMap1 := make(map[string]reflect.Value, 0)
  // wsRouter1 添加事件监听
	var wsRouter1 WsFuncRouter
  
  // 调用wsRouter1的方法，将回调保存在LoginMgr上
  wsRouter1.InitSDK(ConfigSvr, "0")
	wsRouter1.SetAdvancedMsgListener()
	wsRouter1.SetConversationListener()
	wrapSdkLog("SetFriendListener() ", uid)
	wsRouter1.SetFriendListener()
	wrapSdkLog("SetGroupListener() ", uid)
	wsRouter1.SetGroupListener()
	wrapSdkLog("SetUserListener() ", uid)
	wsRouter1.SetUserListener()
  
  var rr RefRouter
	rr.refName = &RouteMap1
	rr.wsRouter = &wsRouter1
	// rr保存了一个uid对应的函数回调
	UserRouteMap[uid] = rr
  
  return &rr
}
```



- ws_init_login 

```go
func (wsRouter *WsFuncRouter) InitSDK(config string, operationID string) {
	var initcb InitCallback
	initcb.uid = wsRouter.uId
	wrapSdkLog("Initsdk uid: ", initcb.uid)
	c := sdk_struct.IMConfig{}
	json.Unmarshal([]byte(config), &c)
	// 初始化一个LoginMgr或从map中取得LoginMgr
	userWorker := open_im_sdk.GetUserWorker(wsRouter.uId)
	// 将回调保存在 LoginMgr.connListener上
	userWorker.InitSDK(c, &initcb, operationID)
}
```



### 通用方法open_im_sdk

- 获取uid对应的LoginMgr

```go
func GetUserWorker(uid string) *login.LoginMgr {}
```



### 核心逻辑 internal

- login/init_login

  ```go
  type LoginMgr struct {
  	friend       *friend.Friend
  	group        *group.Group
  	conversation *conv.Conversation
  	user         *user.User
  	full         *full.Full
  	db           *db.DataBase
    // 这里连着主逻辑的ws，msg_gateway 17778
  	ws           *ws.Ws
  	msgSync      *ws.MsgSync
  
  	heartbeat *ws.Heartbeat
  
  	token        string
  	loginUserID  string
  	connListener open_im_sdk_callback.OnConnListener
  
  	justOnceFlag bool
  
  	groupListener        open_im_sdk_callback.OnGroupListener
  	friendListener       open_im_sdk_callback.OnFriendshipListener
  	conversationListener open_im_sdk_callback.OnConversationListener
  	advancedMsgListener  open_im_sdk_callback.OnAdvancedMsgListener
  	userListener         open_im_sdk_callback.OnUserListener
  
  	conversationCh chan common.Cmd2Value
  	cmdWsCh        chan common.Cmd2Value
  	heartbeatCmdCh chan common.Cmd2Value
  	imConfig       sdk_struct.IMConfig
  }
  ```
  

### 初始化config

```go
type IMConfig struct {
	Platform      int32  `json:"platform"`
	ApiAddr       string `json:"api_addr"`
	WsAddr        string `json:"ws_addr"`
	DataDir       string `json:"data_dir"`
	LogLevel      uint32 `json:"log_level"`
	ObjectStorage string `json:"object_storage"` //"cos"(default)  "oss"
}
```

- Ws_wrapper

  ```go
  ws_local_server.InitServer(&sdk_struct.IMConfig{ApiAddr: "http://" + utils.ServerIP + ":" + utils.IntToString(*openIMApiPort),
  			WsAddr: "ws://" + utils.ServerIP + ":" + utils.IntToString(*openIMWsPort), Platform: utils.WebPlatformID, DataDir: "../db/sdk/"})
  ```

- ws_wrapper/ws_local_server

  ```go
  func InitServer(config *sdk_struct.IMConfig) {
  	data, _ := json.Marshal(config)
  	ConfigSvr = string(data)
  	UserRouteMap = make(map[string]RefRouter, 0)
  	open_im_sdk.InitOnce(config)
  }
  ```

- Open_im_sdk/open_im_sdk_interface

  ```go
  func InitOnce(config *sdk_struct.IMConfig) bool {
  	sdk_struct.SvrConf = *config
  	return true
  }
  ```



### ws心跳

- 初始化心跳， internal/login

  ```go
  func (u *LoginMgr) login(userID, token string, cb open_im_sdk_callback.Base, operationID string) {
    // 初始化LoginMgr里ws相关属性
    // u.wsNotification[msgIncr]，用于接收到对服务器请求的response，channel
    wsRespAsyn := ws.NewWsRespAsyn()
  	wsConn := ws.NewWsConn(u.connListener, token, userID)
  	u.conversationCh = make(chan common.Cmd2Value, 1000)
  	u.cmdWsCh = make(chan common.Cmd2Value, 10)
    pushMsgAndMaxSeqCh := make(chan common.Cmd2Value, 1000)
    // 开始接收流程
  	u.ws = ws.NewWs(wsRespAsyn, wsConn, u.cmdWsCh, pushMsgAndMaxSeqCh)
    
    u.msgSync = ws.NewMsgSync(db, u.ws, userID, u.conversationCh, pushMsgAndMaxSeqCh)
    u.heartbeatCmdCh = make(chan common.Cmd2Value, 10)
    // 开始发送心跳流程
  	u.heartbeat = ws.NewHeartbeat(u.msgSync, u.heartbeatCmdCh)
  }
  ```



#### 发送心跳流程

- Internal/interaction/heartbeat

  ```go
  func NewHeartbeat(msgSync *MsgSync, cmcCh chan common.Cmd2Value) *Heartbeat {
  	p := Heartbeat{MsgSync: msgSync, cmdCh: cmcCh}
  	p.heartbeatInterval = 30
    // 线程里发送接收都是阻塞的，设置了超时
  	go p.Run()
  	return &p
  }
  
  func (u *Heartbeat) Run() {
  	//	heartbeatInterval := 30
  	reqTimeout := 30
  	retryTimes := 0
  	heartbeatNum := 0
    for {
  		operationID := utils.OperationIDGenerator()
      // 从WaitResp收到第一个心跳包的response以后，回到这里
  		if heartbeatNum != 0 {
  			select {
  			case r := <-u.cmdCh:
  				if r.Cmd == constant.CmdLogout {
  					u.SetLoginState(constant.Logout)
  					u.CloseConn()
  					runtime.Goexit()
  				}
  				log.Warn(operationID, "other cmd...", r.Cmd)
  			case <-time.After(time.Millisecond * time.Duration(u.heartbeatInterval*1000)):
  				log.Debug(operationID, "heartbeat waiting(ms)... ", u.heartbeatInterval*1000)
  			}
  		}
      
      heartbeatNum++
      resp, err := u.SendReqWaitResp(&server_api_params.GetMaxAndMinSeqReq{}, constant.WSGetNewestSeq, reqTimeout, retryTimes, u.loginUserID, operationID)
      // 开始处理心跳包的返回值
      var wsSeqResp server_api_params.GetMaxAndMinSeqResp
  		err = proto.Unmarshal(resp.Data, &wsSeqResp)
      // type Heartbeat 包含了MsgSync，传入PushMsgAndMaxSeqCh
      err = common.TriggerCmdMaxSeq(sdk_struct.CmdMaxSeqToMsgSync{OperationID: operationID, MaxSeqOnSvr: wsSeqResp.MaxSeq}, u.PushMsgAndMaxSeqCh)
    }
  }
  ```

- Internal/interaction/ws

  ```go
  func (w *Ws) SendReqWaitResp(m proto.Message, reqIdentifier int32, timeout, retryTimes int, senderID, operationID string) (*GeneralWsResp, error) {
    var wsReq GeneralWsReq
  	var connSend *websocket.Conn
  	var err error
  	wsReq.ReqIdentifier = reqIdentifier
  	wsReq.OperationID = operationID
  	// 在u.wsNotification[msgIncr] 保存一个GeneralWsResp 的channel
  	msgIncr, ch := w.AddCh(senderID)
  	defer w.DelCh(msgIncr)
  	wsReq.SendID = senderID
  	wsReq.MsgIncr = msgIncr
  	wsReq.Data, err = proto.Marshal(m)
    // 重试几次writeBinaryMsg,调用websocket的WriteMessage方法，向服务端发送消息
    for i := 0; i < retryTimes+1; i++ {
  		connSend, err = w.writeBinaryMsg(wsReq)
  		break
  	}
    // 阻塞的，从GeneralWsResp 的channel获取服务端返回的信息
    // 流程结束
  	r1, r2 := w.WaitResp(ch, timeout, wsReq.OperationID, connSend)
  	return r1, r2
  }
  ```

  

#### 接收心跳流程

- internal/interaction/ws

  ```go
  func NewWs(wsRespAsyn *WsRespAsyn, wsConn *WsConn, cmdCh chan common.Cmd2Value, pushMsgAndMaxSeqCh chan common.Cmd2Value) *Ws {
  	p := Ws{WsRespAsyn: wsRespAsyn, WsConn: wsConn, cmdCh: cmdCh, pushMsgAndMaxSeqCh: pushMsgAndMaxSeqCh}
    // 读取线程
  	go p.ReadData()
  	return &p
  }
  ```

  ```go
  func (w *Ws) ReadData() {
    for {
      // 读取来自底层服务器的消息
  		msgType, message, err := w.WsConn.conn.ReadMessage()
      if msgType == websocket.BinaryMessage {
  			go w.doWsMsg(message)
    }
  }
  ```

  ```go
  func (w *Ws) doWsMsg(message []byte) {
    wsResp, err := w.decodeBinaryWs(message)
    switch wsResp.ReqIdentifier {
  	// 心跳包的类型
  	case constant.WSGetNewestSeq:
      w.doWSGetNewestSeq(*wsResp)
  }
  ```

  ```go
  func (w *Ws) doWSGetNewestSeq(wsResp GeneralWsResp) error {
    // 写入u.wsNotification[msgIncr]
    // WaitResp会继续往下走，开始下一个心跳包
  	if err := w.notifyResp(wsResp); err != nil {
  		return utils.Wrap(err, "")
  	}
  	return nil
  }
  ```

  

#### 处理心跳流程

- pkg/common/trigger_channel

```go
// 第二个参数为PushMsgAndMaxSeqCh
func TriggerCmdMaxSeq(seq sdk_struct.CmdMaxSeqToMsgSync, ch chan Cmd2Value) error {
	c2v := Cmd2Value{Cmd: constant.CmdMaxSeq, Value: seq}
	return sendCmd(ch, c2v, 1)
}
```

```go
func sendCmd(ch chan Cmd2Value, value Cmd2Value, timeout int64) error {
	var flag = 0
	select {
	case ch <- value:
		flag = 1
	case <-time.After(time.Second * time.Duration(timeout)):
		flag = 2
	}
}
```

- 处理PushMsgAndMaxSeqCh channel收到的值

- Internal/interaction/msg_sync

  ```go
  func NewMsgSync(dataBase *db.DataBase, ws *Ws, loginUserID string, ch chan common.Cmd2Value, pushMsgAndMaxSeqCh chan common.Cmd2Value) *MsgSync {
  	p := &MsgSync{DataBase: dataBase,
  		Ws: ws, loginUserID: loginUserID, conversationCh: ch, PushMsgAndMaxSeqCh: pushMsgAndMaxSeqCh}
  	p.compareSeq()
  	go common.DoListener(p)
  	return p
  }
  ```

- pkg/common/trigger_channel

  ```go
  func DoListener(Li goroutine) {
  	log.Info("internal", "doListener start.", Li.GetCh())
  	for {
  		select {
  		case cmd := <-Li.GetCh():
  			if cmd.Cmd == constant.CmdUnInit {
  				log.Info("doListener goroutine return")
  				return
  			}
  			Li.Work(cmd)
  		}
  	}
  }
  ```

- Internal/interaction/msg_sync

  ```go
  func (m *MsgSync) Work(cmd common.Cmd2Value) {
  	switch cmd.Cmd {
  	case constant.CmdPushMsg:
  		m.doPushMsg(cmd)
  	case constant.CmdMaxSeq:
      // 心跳包的cmd
  		m.doMaxSeq(cmd)
  	default:
  		log.Error("", "cmd failed ", cmd.Cmd)
  	}
  }
  ```

  ```go
  func (m *MsgSync) doMaxSeq(cmd common.Cmd2Value) {
    // 心跳包返回值
  	var maxSeqOnSvr = cmd.Value.(sdk_struct.CmdMaxSeqToMsgSync).MaxSeqOnSvr
  	operationID := cmd.Value.(sdk_struct.CmdMaxSeqToMsgSync).OperationID
    // 当当前消息seq大于请求回来的seq，不需要处理（没人给你发消息）
  	if maxSeqOnSvr <= m.seqMaxNeedSync {
  		return
  	}
  	m.seqMaxNeedSync = maxSeqOnSvr
  	m.syncMsg()
  }
  ```


#### seq改变后请求消息
- Internal/interaction/msg_sync
  ```go
  func (m *MsgSync) syncMsg() {
  	if m.seqMaxNeedSync > m.seqMaxSynchronized {
      // 从服务器请求更新的消息
  		m.syncMsgFromServer(m.seqMaxSynchronized+1, m.seqMaxNeedSync)
  		m.seqMaxSynchronized = m.seqMaxNeedSync
  	}
  }
  ```

  ```go
  func (m *MsgSync) syncMsgFromServer(beginSeq, endSeq uint32) {
  	var needSyncSeqList []uint32
  	for i := beginSeq; i <= endSeq; i++ {
  		needSyncSeqList = append(needSyncSeqList, i)
  	}
  	var SPLIT = 100
  	for i := 0; i < len(needSyncSeqList)/SPLIT; i++ {
  		m.syncMsgFromServerSplit(needSyncSeqList[i*SPLIT : (i+1)*SPLIT])
  	}
  	m.syncMsgFromServerSplit(needSyncSeqList[SPLIT*(len(needSyncSeqList)/SPLIT):])
  }
  ```

  ```go
  func (m *MsgSync) syncMsgFromServerSplit(needSyncSeqList []uint32) {
  	var pullMsgReq server_api_params.PullMessageBySeqListReq
  	pullMsgReq.SeqList = needSyncSeqList
  	pullMsgReq.UserID = m.loginUserID
  	for {
  		operationID := utils.OperationIDGenerator()
  		pullMsgReq.OperationID = operationID
      // 向服务器发送获取消息的请求,和心跳包的那个函数是一样的，
  		resp, err := m.SendReqWaitResp(&pullMsgReq, constant.WSPullMsgBySeqList, 30, 2, m.loginUserID, operationID)
  		var pullMsgResp server_api_params.PullMessageBySeqListResp
  		err = proto.Unmarshal(resp.Data, &pullMsgResp)
  		m.TriggerCmdNewMsgCome(pullMsgResp.List, operationID)
  		return
  	}
  }
  
  func (m *MsgSync) TriggerCmdNewMsgCome(msgList []*server_api_params.MsgData, operationID string) {
  	for {
  		err := common.TriggerCmdNewMsgCome(sdk_struct.CmdNewMsgComeToConversation{MsgList: msgList, OperationID: operationID}, m.conversationCh)
  		return
  	}
  }
  ```
  
- 和心跳包一样，固定流程
  
- pkg/common/trigger_channel
  
  ```go
  func TriggerCmdNewMsgCome(msg sdk_struct.CmdNewMsgComeToConversation, conversationCh chan Cmd2Value) error {
  	c2v := Cmd2Value{Cmd: constant.CmdNewMsgCome, Value: msg}
  	return sendCmd(conversationCh, c2v, 1)
  }
  ```
  
  ```go
  func sendCmd(ch chan Cmd2Value, value Cmd2Value, timeout int64) error {
  	var flag = 0
  	select {
    // conversationCh
    // 很多trigger都用到这个ch
  	case ch <- value:
  		flag = 1
  	case <-time.After(time.Second * time.Duration(timeout)):
  		flag = 2
  	}
  }
  ```
  
- TriggerCmdNewMsgCome的listener
  
- internal/login/init_login
  
  ```go
  func (u *LoginMgr) login(userID, token string, cb open_im_sdk_callback.Base, operationID string) {
  	u.conversation = conv.NewConversation(u.ws, u.db, p, u.conversationCh,
  	u.loginUserID, u.imConfig.Platform, u.imConfig.DataDir,
  	u.friend, u.group, u.user, objStorage, u.conversationListener, u.advancedMsgListener
  }
  ```
  
- Internal/conversation_msg
  
  ```go
  func NewConversation(ws *ws.Ws, db *db.DataBase, p *ws.PostApi,
  	ch chan common.Cmd2Value, loginUserID string, platformID int32, dataDir string,
  	friend *friend.Friend, group *group.Group, user *user.User,
  	objectStorage common2.ObjectStorage, conversationListener open_im_sdk_callback.OnConversationListener, msgListener open_im_sdk_callback.OnAdvancedMsgListener) *Conversation {
    n := &Conversation{Ws: ws, db: db, p: p, ch: ch, loginUserID: loginUserID, platformID: platformID, DataDir: dataDir, friend: friend, group: group, user: user, ObjectStorage: objectStorage}
  	go common.DoListener(n)
  	n.SetMsgListener(msgListener)
  	n.SetConversationListener(conversationListener)
  }
  ```
  
- pkg/common/trigger_channel

  ```go
  func DoListener(Li goroutine) {
  	for {
  		select {
  		case cmd := <-Li.GetCh():
  			if cmd.Cmd == constant.CmdUnInit {
  				return
  			}
  			Li.Work(cmd)
  		}
  	}
  }
  ```
  
- Internal/conversation_msg
  
  ```go
  func (c *Conversation) Work(c2v common.Cmd2Value) {
  	switch c2v.Cmd {
  	case constant.CmdDeleteConversation:
  		c.doDeleteConversation(c2v)
    // 根据seq差值获取消息，本次执行的
  	case constant.CmdNewMsgCome:
  		c.doMsgNew(c2v)
  	case constant.CmdUpdateConversation:
  		c.doUpdateConversation(c2v)
  	}
  }
  ```
  
  
  
- 



### 演示一次用户发起请求的链路

- 相比http请求，鉴权前置到了ws建立的时候

- 用户请求

  ```json
  {
    data: "[\"13086632509\"]"
    operationID: "pi59g4ra6l164723090979913086632509"
    reqFuncName: "GetUsersInfo"
    uid: "13086632509"
  }
  ```

- ws_local_server/handle_func

  ```go
  func (ws *WServer) msgParse(conn *UserConn, jsonMsg []byte) {
    urm, ok := UserRouteMap[m.UserID]
    parms := []reflect.Value{reflect.ValueOf(m.Data), reflect.ValueOf(m.OperationID)}
  	vf, ok := (*urm.refName)[m.ReqFuncName]
    vf.Call(parms)
  }
  ```

- ws_local_server/ws_user

  ```go
  func (wsRouter *WsFuncRouter) GetUsersInfo(userIDList string, operationID string) {
    // userWorker 是 *login.LoginMgr
  	userWorker := open_im_sdk.GetUserWorker(wsRouter.uId)
  	userWorker.Full().GetUsersInfo(&BaseSuccFailed{runFuncName(), operationID, wsRouter.uId}, 	userIDList, operationID)
  }
  ```

- Internal/login

  ```go
  func (u *LoginMgr) Full() *full.Full {
  	return u.full
  }
  ```

- Internal/full

  ```go
  func (u *Full) GetUsersInfo(callback open_im_sdk_callback.Base, userIDList string, operationID string) {
  	fName := utils.GetSelfFuncName()
  	go func() {
  		log.NewInfo(operationID, fName, "args: ", userIDList)
  		var unmarshalParam sdk_params_callback.GetUsersInfoParam
  		common.JsonUnmarshalAndArgsValidate(userIDList, &unmarshalParam, callback, operationID)
  		result := u.getUsersInfo(callback, unmarshalParam, operationID)
  		callback.OnSuccess(utils.StructToJsonStringDefault(result))
  		log.NewInfo(operationID, fName, "callback: ", utils.StructToJsonStringDefault(result))
  	}()
  }
  
  func (u *Full) getUsersInfo(callback open_im_sdk_callback.Base, userIDList sdk.GetUsersInfoParam, operationID string) sdk.GetUsersInfoCallback {
  	friendList := u.friend.GetDesignatedFriendListInfo(callback, []string(userIDList), operationID)
  	blackList := u.friend.GetDesignatedBlackListInfo(callback, []string(userIDList), operationID)
  	notIn := make([]string, 0)
  	for _, v := range userIDList {
  		inFriendList := 0
  		for _, friend := range friendList {
  			if v == friend.FriendUserID {
  				inFriendList = 1
  				break
  			}
  		}
  		inBlackList := 0
  		for _, black := range blackList {
  			if v == black.BlockUserID {
  				inBlackList = 1
  				break
  			}
  		}
  		if inFriendList == 0 && inBlackList == 0 {
  			notIn = append(notIn, v)
  		}
  	}
  	//from svr
  	publicList := make([]*api.PublicUserInfo, 0)
  	if len(notIn) > 0 {
  		publicList = u.user.GetUsersInfoFromSvr(callback, notIn, operationID)
  	}
  
  	return common.MergeUserResult(publicList, friendList, blackList)
  }
  ```

- Internal/friend

  ```go
  func (f *Friend) getDesignatedFriendsInfo(callback open_im_sdk_callback.Base, friendUserIDList sdk.GetDesignatedFriendsInfoParams, operationID string) sdk.GetDesignatedFriendsInfoCallback {
  	log.NewInfo(operationID, utils.GetSelfFuncName(), "args: ", friendUserIDList)
  
  	localFriendList, err := f.db.GetFriendInfoList(friendUserIDList)
  	common.CheckDBErrCallback(callback, err, operationID)
  
  	blackList, err := f.db.GetBlackInfoList(friendUserIDList)
  	common.CheckDBErrCallback(callback, err, operationID)
  	for _, v := range blackList {
  		log.Info(operationID, "GetBlackInfoList ", *v)
  	}
  
  	r := common.MergeFriendBlackResult(localFriendList, blackList)
  	log.NewInfo(operationID, utils.GetSelfFuncName(), "return: ", r)
  	return r
  }
  ```

- pkg/db 直接从数据库取

  ```go
  func (d *DataBase) GetFriendInfoList(friendUserIDList []string) ([]*LocalFriend, error) {
  	d.mRWMutex.Lock()
  	defer d.mRWMutex.Unlock()
  	var friendList []LocalFriend
  	err := utils.Wrap(d.conn.Where("friend_user_id IN ?", friendUserIDList).Find(&friendList).Error, "GetFriendInfoListByFriendUserID failed")
  	var transfer []*LocalFriend
  	for _, v := range friendList {
  		v1 := v
  		transfer = append(transfer, &v1)
  	}
  	return transfer, err
  }
  ```




### 用户发送消息

- 用户请求

  ```json
  data: "{\"recvID\":\"18370287152\",\"groupID\":\"\",\"onlineUserOnly\":false,\"message\":\"{\\\"clientMsgID\\\":\\\"29f4a9aa024cbad9d2c6bceede9e7ea1\\\",\\\"serverMsgID\\\":\\\"\\\",\\\"createTime\\\":1647445763618568739,\\\"sendTime\\\":1647445763618568739,\\\"sessionType\\\":0,\\\"sendID\\\":\\\"13086632509\\\",\\\"recvID\\\":\\\"\\\",\\\"msgFrom\\\":100,\\\"contentType\\\":101,\\\"platformID\\\":5,\\\"forceList\\\":null,\\\"senderNickName\\\":\\\"xixixi\\\",\\\"senderFaceUrl\\\":\\\"https://echat-1302656840.cos.ap-chengdu.myqcloud.com/rc-upload-1644478797882-41%E5%BE%AE%E4%BF%A1%E5%9B%BE%E7%89%87_20220210155755.jpg\\\",\\\"groupID\\\":\\\"\\\",\\\"content\\\":\\\"无明月\\\",\\\"seq\\\":0,\\\"isRead\\\":false,\\\"status\\\":1,\\\"remark\\\":\\\"\\\",\\\"pictureElem\\\":{\\\"sourcePath\\\":\\\"\\\",\\\"sourcePicture\\\":{\\\"uuid\\\":\\\"\\\",\\\"type\\\":\\\"\\\",\\\"size\\\":0,\\\"width\\\":0,\\\"height\\\":0,\\\"url\\\":\\\"\\\"},\\\"bigPicture\\\":{\\\"uuid\\\":\\\"\\\",\\\"type\\\":\\\"\\\",\\\"size\\\":0,\\\"width\\\":0,\\\"height\\\":0,\\\"url\\\":\\\"\\\"},\\\"snapshotPicture\\\":{\\\"uuid\\\":\\\"\\\",\\\"type\\\":\\\"\\\",\\\"size\\\":0,\\\"width\\\":0,\\\"height\\\":0,\\\"url\\\":\\\"\\\"}},\\\"soundElem\\\":{\\\"uuid\\\":\\\"\\\",\\\"soundPath\\\":\\\"\\\",\\\"sourceUrl\\\":\\\"\\\",\\\"dataSize\\\":0,\\\"duration\\\":0},\\\"videoElem\\\":{\\\"videoPath\\\":\\\"\\\",\\\"videoUUID\\\":\\\"\\\",\\\"videoUrl\\\":\\\"\\\",\\\"videoType\\\":\\\"\\\",\\\"videoSize\\\":0,\\\"duration\\\":0,\\\"snapshotPath\\\":\\\"\\\",\\\"snapshotUUID\\\":\\\"\\\",\\\"snapshotSize\\\":0,\\\"snapshotUrl\\\":\\\"\\\",\\\"snapshotWidth\\\":0,\\\"snapshotHeight\\\":0},\\\"fileElem\\\":{\\\"filePath\\\":\\\"\\\",\\\"uuid\\\":\\\"\\\",\\\"sourceUrl\\\":\\\"\\\",\\\"fileName\\\":\\\"\\\",\\\"fileSize\\\":0},\\\"mergeElem\\\":{\\\"title\\\":\\\"\\\",\\\"abstractList\\\":null,\\\"multiMessage\\\":null},\\\"atElem\\\":{\\\"text\\\":\\\"\\\",\\\"atUserList\\\":[],\\\"isAtSelf\\\":false},\\\"locationElem\\\":{\\\"description\\\":\\\"\\\",\\\"longitude\\\":0,\\\"latitude\\\":0},\\\"customElem\\\":{\\\"data\\\":\\\"\\\",\\\"description\\\":\\\"\\\",\\\"extension\\\":\\\"\\\"},\\\"quoteElem\\\":{\\\"text\\\":\\\"\\\",\\\"quoteMessage\\\":null}}\"}"
  operationID: "x1sigktma81647445763544"
  reqFuncName: "SendMessage"
  uid: "13086632509"
  ```

- ws_local_server/ws_conversation_msg

  ```go
  // sendCallback 结构
  type SendCallback struct {
  	BaseSuccFailed
  	clientMsgID string
  	//uid         string
  }
  func (s *SendCallback) OnProgress(progress int)
  
  func (wsRouter *WsFuncRouter) SendMessage(input string, operationID string) {
  	m := make(map[string]interface{})
    json.Unmarshal([]byte(input), &m)
  
  	var sc SendCallback
  	sc.uid = wsRouter.uId
  	sc.funcName = runFuncName()
  	sc.operationID = operationID
  	userWorker := open_im_sdk.GetUserWorker(wsRouter.uId)
  	if !wsRouter.checkKeysIn(input, operationID, runFuncName(), m, "message", "recvID", "groupID", "offlinePushInfo") {
  		return
  	}
  	userWorker.Conversation().SendMessage(&sc, m["message"].(string), m["recvID"].(string), m["groupID"].(string), m["offlinePushInfo"].(string), operationID)
  
  }
  ```

- internal/open_im_sdk_conversation_msg
