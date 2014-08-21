- [Synchronous query](Usage-examples.md#synchronous-query)
- [Asynchronous query](Usage-examples.md#asynchronous-query)
- [Interactive console](Usage-examples.md#interactive-console)
- [Twisted integration](Usage-examples.md#twisted-integration)

### Synchronous query
Following example presents how to execute simple, synchronous query against a remote q process:

```python
from qpython import qconnection

if __name__ == '__main__':
    # create connection object
    q = qconnection.QConnection(host = 'localhost', port = 5000)
    # initialize connection
    q.open()

    print q
    print 'IPC version: %s. Is connected: %s' % (q.protocol_version, q.is_connected())

    # simple query execution via: QConnection.__call__
    data = q('{`int$ til x}', 10)
    print 'type: %s, numpy.dtype: %s, meta.qtype: %s, data: %s ' % (type(data), data.dtype, data.meta.qtype, data)
    
    # simple query execution via: QConnection.sync
    data = q.sync('{`long$ til x}', 10)
    print 'type: %s, numpy.dtype: %s, meta.qtype: %s, data: %s ' % (type(data), data.dtype, data.meta.qtype, data)
    
    # low-level query and read
    q.query(qconnection.MessageType.SYNC,'{`short$ til x}', 10) # sends a SYNC query
    msg = q.receive(data_only = False, raw = False) # retrieve entire message
    print 'type: %s, message type: %s, data size: %s, is_compressed: %s ' % (type(msg), msg.type, msg.size, msg.is_compressed)
    data = msg.data
    print 'type: %s, numpy.dtype: %s, meta.qtype: %s, data: %s ' % (type(data), data.dtype, data.meta.qtype, data)
    # close connection
    q.close()
```

This code prints to the console:
```
:localhost:5000
IPC version: 3. Is connected: True
type: <class 'qpython.qcollection.QList'>, numpy.dtype: int32, meta.qtype: 6, data: [0 1 2 3 4 5 6 7 8 9] 
type: <class 'qpython.qcollection.QList'>, numpy.dtype: int64, meta.qtype: 7, data: [0 1 2 3 4 5 6 7 8 9] 
type: <class 'qpython.qreader.QMessage'>, message type: 2, data size: 34, is_compressed: False 
type: <class 'qpython.qcollection.QList'>, numpy.dtype: int16, meta.qtype: 5, data: [0 1 2 3 4 5 6 7 8 9] 
```

### Asynchronous query
Following example presents how to execute simple, asynchronous query against a remote q process:

```python
import random
import threading
import time

from qpython import qconnection
from qpython.qtype import QException
from qpython.qconnection import MessageType
from qpython.qcollection import QDictionary


class ListenerThread(threading.Thread):
    
    def __init__(self, q):
        super(ListenerThread, self).__init__()
        self.q = q
        self._stop = threading.Event()

    def stop(self):
        self._stop.set()

    def stopped(self):
        return self._stop.isSet()

    def run(self):
        while not self.stopped():
            print '.'
            try:
                message = self.q.receive(data_only = False, raw = False) # retrieve entire message
                
                if message.type != MessageType.ASYNC:
                    print 'Unexpected message, expected message of type: ASYNC'
                    
                print 'type: %s, message type: %s, data size: %s, is_compressed: %s ' % (type(message), message.type, message.size, message.is_compressed)
                print message.data
                
                if isinstance(message.data, QDictionary):
                    # stop adter 10th query
                    if message.data['queryid'] == 9:
                        self.stop()
                    
            except QException, e:
                print e


if __name__ == '__main__':
    # create connection object
    q = qconnection.QConnection(host = 'localhost', port = 5000)
    # initialize connection
    q.open()

    print q
    print 'IPC version: %s. Is connected: %s' % (q.protocol_version, q.is_connected())

    try:
        # definition of asynchronous multiply function
        # queryid - unique identifier of function call - used to identify
        # the result
        # a, b - parameters to the query
        q.sync('asynchMult:{[queryid;a;b] res:a*b; (neg .z.w)(`queryid`result!(queryid;res)) }');

        t = ListenerThread(q)
        t.start()
         
        for x in xrange(10):
            a = random.randint(1, 100)
            b = random.randint(1, 100)
            print 'Asynchronous call with queryid=%s with arguments: %s, %s' % (x, a, b)
            q.async('asynchMult', x, a, b);
        
        time.sleep(1)
    finally:
        q.close()
```

### Interactive console
This example depicts how to create a simple interactive console for communication with a q process:

```python
from qpython import qconnection
from qpython.qtype import QException


if __name__ == '__main__':
    q = qconnection.QConnection(host = 'localhost', port = 5000)
    q.open()

    print q
    print 'IPC version: %s. Is connected: %s' % (q.protocol_version, q.is_connected())

    while True:
        try:
            x = raw_input('Q)')
        except EOFError:
            print
            break

        if x == '\\\\':
            break

        try:
            result = q(x)
            print type(result)
            print result
        except QException, msg:
            print 'q error: \'%s' % msg

    q.close()
```

### Twisted integration
This example presents how the `qPython` can be used along with [Twisted framework](https://twistedmatrix.com/trac/) to build asynchronous client:


```python
import struct
import sys

from twisted.internet.protocol import Protocol, ClientFactory

from twisted.internet import reactor
from qpython.qconnection import MessageType, QAuthenticationException
from qpython.qreader import QReader
from qpython.qwriter import QWriter, QWriterException

class IPCProtocol(Protocol):
    class State(object):
        UNKNOWN = -1
        HANDSHAKE = 0
        CONNECTED = 1

    def connectionMade(self):
        self.state = IPCProtocol.State.UNKNOWN
        self.credentials = self.factory.username + ':' + self.factory.password if self.factory.password else ''

        self.transport.write(self.credentials + '\3\0')
        
        self._message = None

    def dataReceived(self, data):
        if self.state == IPCProtocol.State.CONNECTED:
            try:
                if not self._message:
                    self._message = self._reader.read_header(source = data)
                    self._buffer = ''
                    
                self._buffer += data
                buffer_len = len(self._buffer) if self._buffer else 0
                
                while self._message and self._message.size <= buffer_len:
                    complete_message = self._buffer[:self._message.size]
                    
                    if buffer_len > self._message.size:
                        self._buffer = self._buffer[self._message.size:]
                        buffer_len = len(self._buffer) if self._buffer else 0
                        self._message = self._reader.read_header(source = self._buffer)
                    else:
                        self._message = None
                        self._buffer = ''
                        buffer_len = 0

                    self.factory.onMessage(self._reader.read(source = complete_message))
            except:
                self.factory.onError(sys.exc_info())
                self._message = None
                self._buffer = ''
                
        elif self.state == IPCProtocol.State.UNKNOWN:
            # handshake
            if len(data) == 1:
                self._init(data)
            else:
                self.state = IPCProtocol.State.HANDSHAKE
                self.transport.write(self.credentials + '\0')
                
        else:
            # protocol version fallback
            if len(data) == 1:
                self._init(data)
            else:
                raise QAuthenticationException('Connection denied.')

    def _init(self, data):
        self.state = IPCProtocol.State.CONNECTED
        self.protocol_version = min(struct.unpack('B', data)[0], 3)
        self._writer = QWriter(stream = None, protocol_version = self.protocol_version)
        self._reader = QReader(stream = None)

        self.factory.clientReady(self)

    def query(self, msg_type, query, *parameters):
        if parameters and len(parameters) > 8:
            raise QWriterException('Too many parameters.')

        if not parameters or len(parameters) == 0:
            self.transport.write(self._writer.write(query, msg_type))
        else:
            self.transport.write(self._writer.write([query] + list(parameters), msg_type))


class IPCClientFactory(ClientFactory):
    protocol = IPCProtocol

    def __init__(self, username, password, connect_success_callback, connect_fail_callback, data_callback, error_callback):
        self.username = username
        self.password = password
        self.client = None

        # register callbacks
        self.connect_success_callback = connect_success_callback
        self.connect_fail_callback = connect_fail_callback
        self.data_callback = data_callback
        self.error_callback = error_callback

    def clientConnectionLost(self, connector, reason):
        print 'Lost connection.  Reason:', reason
        # connector.connect()

    def clientConnectionFailed(self, connector, reason):
        if self.connect_fail_callback:
            self.connect_fail_callback(self, reason)

    def clientReady(self, client):
        self.client = client
        if self.connect_success_callback:
            self.connect_success_callback(self)

    def onMessage(self, message):
        if self.data_callback:
            self.data_callback(self, message)

    def onError(self, error):
        if self.error_callback:
            self.error_callback(self, error)

    def query(self, msg_type, query, *parameters):
        if self.client:
            self.client.query(msg_type, query, *parameters)


def onConnectSuccess(source):
    print 'Connected, protocol version: ', source.client.protocol_version
    source.query(MessageType.SYNC, '.z.ts:{(handle)((1000*(1 ? 100))[0] ? 100)}')
    source.query(MessageType.SYNC, '.u.sub:{[t;s] handle:: neg .z.w}')
    source.query(MessageType.ASYNC, '.u.sub', 'trade', '')

def onConnectFail(source, reason):
    print 'Connection refused: ', reason

def onMessage(source, message):
    print 'Received: ', message.type, message.data

def onError(source, error):
    print 'Error: ', error

if __name__ == '__main__':
    factory = IPCClientFactory('user', 'pwd', onConnectSuccess, onConnectFail, onMessage, onError)
    reactor.connectTCP('localhost', 5000, factory)
    reactor.run()
```