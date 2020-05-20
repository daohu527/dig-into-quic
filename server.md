# dig-into-quic
Analysis of facebook quic project mvfst, if you don't know about quic, pls first read the rfc or some introductions about quic. Quic has many benefits, such as fast and reliable. If you want to start with quic, the mvfst is a good project. Of course, there are two other projects related to quic, namely "libquic" and "quiche".
* libquic - It hasn’t been updated in about 4 years, so I wonder if it’s on time and someone maintains it.  
* quiche - google's quic project, which now is a submodule in chromium project. It's hard to build, because you need to download the whole chromium project.  

So I chose facebook's mvfst to start my journey!  

## content
The mvfst's directory structure is as follows.  
```
.
├── CMakeLists.txt           // cmake file
├── CODE_OF_CONDUCT.md
├── CONTRIBUTING.md
├── LICENSE
├── QUIC_TRACE_FORMAT.md
├── README.md
├── build                   // build dependencies, which is a facebook style
├── build_helper.sh         // build script, used to build the whole project
├── cmake                   // cmake
├── getdeps.sh
├── install.sh
├── logo.png
└── quic                    // quic source code
```
As you see, the source code is in "quic" directory, and you can use "./build_helper.sh" to build the project. The project build is base on cmake, we will find more details in the feture, but for now we just start to read the quic source code.  


## quic
We first browse the quic directory and understand the general functions.  
```
.
├── CMakeLists.txt
├── QuicConstants.cpp        // Constants
├── QuicConstants.h
├── QuicException.cpp        // Exception
├── QuicException.h
├── api                      // api for quic
├── client                   // client
├── codec                    //
├── common
├── congestion_control
├── fizz                     // facebook's implementation of the TLS-1.3 standard
├── flowcontrol
├── handshake
├── happyeyeballs            // ???
├── logging
├── loss                     // ???
├── samples                  // a sample for quic
├── server
├── state
└── tools
```
From the directory description, we can know that quic implements the client and server, and provides a flow control and congestion control mechanism. But how quic implement that?  

We first start to the client side.  

## client
client fisrt defined "QuicClientTransport", which inherit "QuicTransportBase", "folly::AsyncUDPSocket::ReadCallback" and "folly::AsyncUDPSocket::ErrMessageCallback".  
```c++
class QuicClientTransport
    : public QuicTransportBase,
      public folly::AsyncUDPSocket::ReadCallback,
      public folly::AsyncUDPSocket::ErrMessageCallback,
      public std::enable_shared_from_this<QuicClientTransport> {
```

create new client  
```c++
  template <class TransportType = QuicClientTransport>
  static std::shared_ptr<TransportType> newClient(
      folly::EventBase* evb,
      std::unique_ptr<folly::AsyncUDPSocket> sock,
      std::shared_ptr<ClientHandshakeFactory> handshakeFactory,
      size_t connectionIdSize = 0) {
  	// 1. create a new client, why the connectionIdSize is 0 ???
    auto client = std::make_shared<TransportType>(
        evb, std::move(sock), std::move(handshakeFactory), connectionIdSize);
    // 2. make client shared_ptr own this ???
    client->setSelfOwning();
    return client;
  }
```

add new peer address.  
```c++
void QuicClientTransport::addNewPeerAddress(folly::SocketAddress peerAddress) {
  CHECK(peerAddress.isInitialized());

  if (happyEyeballsEnabled_) {
  	// 1. UDP send socket min packet size, what's happyEyeballsEnabled_ ???
    conn_->udpSendPacketLen = std::min(
        conn_->udpSendPacketLen,
        (peerAddress.getFamily() == AF_INET6 ? kDefaultV6UDPSendPacketLen
                                             : kDefaultV4UDPSendPacketLen));
    happyEyeballsAddPeerAddress(*conn_, peerAddress);
    return;
  }
  // 2. set connection's peer address and packet length.  
  conn_->udpSendPacketLen = peerAddress.getFamily() == AF_INET6
      ? kDefaultV6UDPSendPacketLen
      : kDefaultV4UDPSendPacketLen;
  conn_->originalPeerAddress = peerAddress;
  conn_->peerAddress = std::move(peerAddress);
}
```

set local address. Then peer address is the server's IP address and the local address is the client's IP address ?  
```c++
void QuicClientTransport::setLocalAddress(folly::SocketAddress localAddress) {
  CHECK(localAddress.isInitialized());
  conn_->localAddress = std::move(localAddress);
}
```

add new socket. Add the socket to the connection??
```c++
void QuicClientTransport::addNewSocket(
    std::unique_ptr<folly::AsyncUDPSocket> socket) {
  happyEyeballsAddSocket(*conn_, std::move(socket));
}

void happyEyeballsAddSocket(
    QuicConnectionStateBase& connection,
    std::unique_ptr<folly::AsyncUDPSocket> socket) {
  connection.happyEyeballsState.secondSocket = std::move(socket);
}
```

start the connect.  
```c++
void QuicClientTransport::start(ConnectionCallback* cb) {
  if (happyEyeballsEnabled_) {
    // TODO Supply v4 delay amount from somewhere when we want to tune this
    startHappyEyeballs(
        *conn_,
        evb_,
        happyEyeballsCachedFamily_,
        happyEyeballsConnAttemptDelayTimeout_,
        happyEyeballsCachedFamily_ == AF_UNSPEC
            ? kHappyEyeballsV4Delay
            : kHappyEyeballsConnAttemptDelayWithCache,
        this,
        this,
        socketOptions_);
  }

  // 1. check if peer address is set.  
  CHECK(conn_->peerAddress.isInitialized());

  if (conn_->qLogger) {
    conn_->qLogger->addTransportStateUpdate(kStart);
  }
  QUIC_TRACE(fst_trace, *conn_, "start");
  // 2. set connection callback function
  setConnectionCallback(cb);
  try {
    happyEyeballsSetUpSocket(
        *socket_,
        conn_->localAddress,
        conn_->peerAddress,
        conn_->transportSettings,
        this,
        this,
        socketOptions_);
    // 3. adjust the GRO buffers, does not exceed the maximum value "kMaxNumGROBuffers"
    adjustGROBuffers();
    // 4. start crypto handshake process
    startCryptoHandshake();
  // 5. exception process branch
  } catch (const QuicTransportException& ex) {
    runOnEvbAsync([ex](auto self) {
      auto clientPtr = static_cast<QuicClientTransport*>(self.get());
      clientPtr->closeImpl(std::make_pair(
          QuicErrorCode(ex.errorCode()), std::string(ex.what())));
    });
  } catch (const QuicInternalException& ex) {
    runOnEvbAsync([ex](auto self) {
      auto clientPtr = static_cast<QuicClientTransport*>(self.get());
      clientPtr->closeImpl(std::make_pair(
          QuicErrorCode(ex.errorCode()), std::string(ex.what())));
    });
  } catch (const std::exception& ex) {
    LOG(ERROR) << "Connect failed " << ex.what();
    runOnEvbAsync([ex](auto self) {
      auto clientPtr = static_cast<QuicClientTransport*>(self.get());
      clientPtr->closeImpl(std::make_pair(
          QuicErrorCode(TransportErrorCode::INTERNAL_ERROR),
          std::string(ex.what())));
    });
  }
}
```

read the data, 
```c++
void QuicClientTransport::onReadData(
    const folly::SocketAddress& peer,
    NetworkDataSingle&& networkData) {
  // 1. if connection is closed, then skip.
  if (closeState_ == CloseState::CLOSED) {
    // If we are closed, then we shoudn't process new network data.
    // TODO: we might want to process network data if we decide that we should
    // exit draining state early
    if (conn_->qLogger) {
      conn_->qLogger->addPacketDrop(0, kAlreadyClosed);
    }
    QUIC_TRACE(packet_drop, *conn_, "already_closed");
    return;
  }
  // 2. whether received packets
  bool waitingForFirstPacket = !hasReceivedPackets(*conn_);
  // 3. start to deal with UDP data
  processUDPData(peer, std::move(networkData));
  // 4. Called after the transport successfully processes the received packet.
  if (connCallback_ && waitingForFirstPacket && hasReceivedPackets(*conn_)) {
    connCallback_->onFirstPeerPacketProcessed();
  }
  // 5. ???
  if (!transportReadyNotified_ && hasWriteCipher()) {
    transportReadyNotified_ = true;
    CHECK_NOTNULL(connCallback_)->onTransportReady();
  }

  // 6. ???
  // Checking connCallback_ because application will start to write data
  // in onTransportReady, if the write fails, QuicSocket can be closed
  // and connCallback_ is set nullptr.
  if (connCallback_ && !replaySafeNotified_ && conn_->oneRttWriteCipher) {
    replaySafeNotified_ = true;
    // We don't need this any more. Also unset it so that we don't allow random
    // middleboxes to shutdown our connection once we have crypto keys.
    socket_->setErrMessageCallback(nullptr);
    connCallback_->onReplaySafe();
  }
}
```
