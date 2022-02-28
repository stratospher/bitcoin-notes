## Version Handshake

Conceptual:
- What does the "version handshake" refer to? What is it's significance?
  <details open>
  <summary>Answer</summary>
  
  - version handshake = after the node has identified a peer it wants to connect to, it sends a version message to the peer and waits for
  the peer to send a version message and a verack message back to it. It then sends a verack back to the peer to establish
  the connection.
  - significance: 
      1. each node can get information about the other node like its version number, block and current time.
         - [src/version.h](https://github.com/bitcoin/bitcoin/blob/master/src/version.h) contains different protocol versions
         - todo: these numbers are explicitly decided? 
      2. it's how a reliable connection can be established.(similar to how TCP-handshake establishes TCP connection)
         Only after the version handshake is over can the peers accept other messages.
      3. service advertising - the version message would contain the service bits a peer advertises and hence
         our node would know the services the peer supports.
         - they are obtained by performing '|' on the [service flags](https://github.com/bitcoin/bitcoin/blob/159f89c118645c0f9e23e453f01cede84abac795/src/protocol.h#L271)
         - Ex: `nServices=9` means `"NODE_WITNESS = (1 << 3)" | "NODE_NETWORK = (1 << 0)`
         - todo: why version message services are not same (1 and 9)
  - [version message](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L1140)
    `msg_version(nVersion=70016 nServices=9 nTime=Mon Feb 28 18:18:02 2022 addrTo=CAddress(nServices=1 net=IPv4 addr=127.0.0.1 port=14837)
    addrFrom=CAddress(nServices=1 net=IPv4 addr=0.0.0.0 port=0) nNonce=0x25883261BB54682B strSubVer=/python-p2p-tester:0.0.3/ nStartingHeight=-1 relay=1)`
  - [verack message](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2657)
  - todo: service bit advertising
  - todo: why a nonce?
  </details>
- What other P2P messages are potentially sent during the handshake?
  <details open>
  <summary>Answer</summary>

  - [NetMsgType::WTXIDRELAY](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2645) - using 
  wtxid in place of the txid when announcing and fetching transactions.
  - [NetMsgType::SENDADDRV2](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2654) - to support
  torv3 and other networks. [link](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-December/018301.html)
   

  - todo: why send them during the handshake?
  - todo: why not `message.nVersion >= 70016` when [sending](https://github.com/bitcoin/bitcoin/blob/25290071c434638e2719a99784572deef44542ad/test/functional/test_framework/p2p.py#L439)
    `msg_sendaddrv2()` in python code since check is being done in c++.
  </details>

Bitcoin Core:
- Where does a node send the `version` and `verack` messages?  What logical
  conditions need to be met to trigger sending each message?
  <details open>
  <summary>Answer</summary>
  
  - `version` message is sent by `PeerManagerImpl::PushNodeVersion()` [here](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L1140).
      - triggered when:
        1. When a node is initialised and outbound connection needs to be established. (In [PeerManagerImpl::InitializeNode(CNode *pnode)](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L1201))
        2. When the node receives version message from it's inbound peers. (In [PeerManagerImpl::ProcessMessage](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2634))
  - `verack` message is sent by `PeerManagerImpl::ProcessMessage()` [here](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2657) as a part of the version handshake.
      - triggered when:
        1. version message received from the peer. (In [PeerManagerImpl::ProcessMessage](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2657))
  </details>
- Where does a node recieve the `version` and `verack` messages? What does it
  do with this information?
  <details open>
  <summary>Answer</summary>
  
  - All messages are read in `V1TransportDeserializer::vRecv` by `CNode::ReceiveMsgBytes()`
  - `version` message is received in `PeerManagerImpl::ProcessMessage()`
    - it performs checks like:
    1. make sure it's not a redundant version message - version can only be reset by disconnecting.
    2. if it's an outbound connection & we received version from peer. this means peer has all the services we need. so we can set our services. 
    3. disconnect if peer doesn't have services we need
    4. disconnect if it's a self-connection
    - todo: [download peer?](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2684)
    - todo: [harder for others to tamper with our adjusted time - means? also isn't this an outbound peer. see L2790.](https://github.com/bitcoin/bitcoin/blob/ee8c99712561bfbe823d9cd787a421b5424a75d9/src/net_processing.cpp#L2757)
  - `verack` message is received in `PeerManagerImpl::ProcessMessage()`
    - it performs checks like:
    1. not a redundant verack
    2. send `NetMsgType::SENDHEADERS`, `NetMsgType::SENDCMPCT` if allowed
    3. set `fSuccessfullyConnected` to `true`
  </details>
- In the test framework function `add_p2p_connection`, what is the point of the
  `wait_for_verack` param?
  <details open>
  <summary>Answer</summary>

  - to fully establish the connection before receiving other messages
  - todo: to prevent race conditions?
  </details>
- Write a test where a node adds a `P2PConnection`. What are different ways you
  can observe the messages sent by both the node & p2p connection to complete
  the version handshake?
  <details open>
  <summary>Answer</summary>

  - Different ways to observe:
    - `on_version()`, `on_verack()`
    - `getpeerinfo()`
    - combine_logs
    - lldb-pdb combo :P
  - test
```
"""
Write a test where a node adds a `P2PConnection`.
What are different ways you can observe the messages sent by both the node & p2p connection to complete the version handshake?
"""
from test_framework.p2p import P2PInterface
from test_framework.test_framework import BitcoinTestFramework

class BaseNode(P2PInterface):
  def __init__(self):
    super().__init__()
  
  def on_version(self, message):
    super().on_version(message)
    print(message)
  
  def on_verack(self, message):
     super().on_verack(message)
     print(message)

class VersionHandshakeTest(BitcoinTestFramework):
  def set_test_params(self):
    self.num_nodes = 1

  def run_test(self):
    peer = self.nodes[0].add_p2p_connection(BaseNode())
    info = self.nodes[0].getpeerinfo()
    assert_greater_than(info[0]['bytesrecv_per_msg']['version'], 0)
    assert_greater_than(info[0]['bytesrecv_per_msg']['verack'], 0)
    print(info)

if __name__ == '__main__':
VersionHandshakeTest().main()

```
  </details>
