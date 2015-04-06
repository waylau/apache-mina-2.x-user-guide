IoService
====
  
MINA IoService - 正如前面[应用架构](Application Architecture.md)里提到过的，是支持所有 IO 服务的基类，不管是在服务器端还是在客户端。
        
它将处理所有与你的应用之间的交互，以及与远程对端的交互，发送并接收消息、管理 session、管理连接等等。
        
它是为一个接口，服务器端实现为 IoAcceptor，客户端为 IoConnector。