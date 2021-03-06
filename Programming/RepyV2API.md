# Repy V2 Library Reference
----

For a detailed description of Network API call semantics see: [NetworkApiSemantics](https://github.com/SeattleTestbed/docs/blob/master/UnderstandingSeattle/NetworkApiSemantics.md).  (Read the description of calls on this page first.)
I have made the following resource consumption simplifications to make the interface easier to understand:
 * I assume that the minimum packet size is 64 bytes and all data consumes extra space.   This unfairly penalizes small packets.
 * I assume there is no maximum packet size.   This slightly benefits large packets / messages / sends.
 * I assume that every separate network API call will result in a separate packet.   This presumes no 'batching' occurs.
 * I assume that every disk I/O operation touches a separate block.
 * I assume that a disk block has size 4K.
 * I assume that every chunk of file data is aligned on 4K intervals.   This means that small reads / writes may cross blocks if they are done at 4K boundaries.
 * I assume that each file that is created results in 4K of space and disk I/O.

For more information on the Exception hierarchy for Repy 2.0, see FutureRepyExceptions.

If multiple exceptions should occur for a given operation, the exception listed first in the doc string will happen.   For example, if openfile is called on an invalid filename when there are no available file handle resources, RepyArgumentError will occur instead of ResourceExhaustedError.

## Introduction and Purpose
----
This document describes the narrow API available to repy programs.   This document includes all of the calls that are available to a repy program and the use and meaning of these calls.   For items built into the python programming language (like list operations, etc.) see the appropriate Python documentation.   

The intent is that we will build libraries that will provide 'rich' functionality on top of these abstractions.   For example, the file-like objects don't support next, but we can build this in a library.   Also, notice that the logging mechanism doesn't support multiple arguments / do type conversion.   We can do this in a user level library.   We will expect that all users will load this library (and may even do it for them).

## Cheat Sheet
----
|Variable Name|Short description|Tutorial Section|
|-------------|-----------------|----------------|
|callargs|callargs are the command line arguments for your program (similar to Python's sys.argv[1:])|example 2.2|

|Object Name|Short description|Tutorial Section|
|-----------|-----------------|----------------|
|file|similar to Python's file object|example 2.1|
|socket|similar to Python's socket object|example 1.2|
|lock|similar to Python's threading.Lock object|example 1.6|
|tcpserversocket|A way to obtain incoming connections|new|
|udpserversocket|A way to obtain incoming packets|new|
|virtualnamespace|A way to evaluate statically analyzed code|new|

|Object / Function Name|Short description|Tutorial Section|
|----------------------|-----------------|----------------|
|file.readat|Read data from a file (maybe seek first).|N/A (new call)|
|file.writeat|Write data to a file (maybe seek first).|N/A (new call)|
|file.close|Close an open file.|example 2.1|
|socket.recv|Read data from a socket in a non-blocking manner|example 1.2|
|socket.send|Send data on a socket in a non-blocking manner|example 1.2|
|socket.close|Closes a socket|example 3.2|
|lock.acquire|Acquire the lock|example 1.6|
|lock.release|Release the lock|example 1.6|
|tcpserversocket.getconnection|Returns a socket from a new connection| N/A (new call)|
|tcpserversocket.close|Stops listening for connections| new call|
|udpserversocket.getmessage|Returns a UDP packet| N/A (new call)|
|udpserversocket.close|Stops listening for connections| new call|
|virtualnamespace.evaluate|Evaluates the code in a given global context|new call|

|Function Name|Short description|Tutorial Section|
|-------------|-----------------|----------------|
|openfile|opens file objects. |example 2.1|
|listfiles|Returns a list of file names for the files in the VM|example 2.2|
|removefile|similar to Python's os.remove() function|example 2.2|
|sleep|similar to Python's time.sleep() function|example 1.6|
|randombytes|similar to Python's os.urandom() function|(new call) example 4.2|
|createlock|similar to Python's threading.Lock() constructor|example 1.6|
|exitall|similar to Python's sys.exit() or os._exit()|example 3.2|
|getruntime|gets the time since the program was started|example 1.4|
|gethostbyname|similar to Python's socket.gethostbyname() function|example 4.1|
|getmyip|gets an external IP address of the computer|example 4.1|
|sendmessage|sends a message to another computer|example 4.2|
|openconnection|opens a connection to another computer|example 3.1|
|listenforconnection|return a tcpserversocket that can be used to accept incoming connections|new|
|listenformessage|return a udpserversocket that can be used to accept incoming messages|new|
|createthread|creates a new thread to execute a function|new (example 1.4)|
|getthreadname|provides a unique string that indicates the name of the current thread|New call|
|createvirtualnamespace|Provides a virtualnamespace object that can be used to evaluate code|New call|
|getresources|Provides two dictionaries which represent the resource limits and usage and an array of stoptimes|New call|
|getlasterror|Obtain information about the last error (exception) that occurred in the current thread.|New call|

## Detailed description
----
### Shared variables
----
#### mycontext
----
Mycontext is a dictionary that is shared between all of the threads in your program. The intent is that mycontext will be used in the place of global variables. See the tutorial for examples using mycontext.

### API functions
----
#### Network functions
##### gethostbyname(name)
----
(from the Python documentation)   Translate a host name to IPv4 address format. The IPv4 address is returned as a string, such as '100.50.200.5'. If the host name is an IPv4 address itself it is returned unchanged.


 * Doc string:
```python
"""
  <Purpose>
    Provides information about a hostname. Calls socket.gethostbyname().
    Translates a hostname to IPv4 address format. The IPv4 address is
    returned as a string, such as '100.50.200.5'. If the host name is an
    IPv4 address itself it is returned unchanged.

  <Arguments>
    name:
       The host name to translate.

  <Exceptions>
    RepyArgumentError (descends from NetworkError) if the name is not a string

    NetworkAddressError (descends from NetworkError) if the address cannot
    be resolved.

  <Side Effects>
    None.

  <Resource Consumption>
    This operation consumes network bandwidth of 4K netrecv, 1K netsend.
    (It's hard to tell how much was actually sent / received at this level.)

  <Returns>
    The IPv4 address as a string.
"""
```

##### getmyip()
----
Returns the localhost's "Internet facing" IP address.   If there are multiple interfaces, this IP will be on the interface that will be used by default to handle traffic to Internet hosts.   Note that if the node is behind a NAT or similar, this may be a private IP.  It may raise an exception on hosts that are not connected to the Internet.

 * Doc string:
```python
"""
  <Purpose>
    Provides the IP of this computer on its public facing interface.   Does some clever trickery.

  <Arguments>
    None

  <Exceptions>
    InternetConnectivityError if the host is not connected to the internet.

  <Side Effects>
    None.

  <Resource Consumption>
    This operations consumes 256 netsend and 128 netrecv.

  <Returns>
    The localhost's IP address
"""
```

##### sendmessage(destip, destport, message, localip, localport)
----
(rename of ~~sendmess~~)

Sends a UDP message to a destination host / port using a specified localip and localport.   Returns the number of bytes sent.   **This may not be the entire message**

 * Doc string:
```python
"""
  <Purpose>
    Send a message to a host / port

  <Arguments>
    destip:
       The IP to send the message to
    destport:
       The port to send the message to
    message:
       The message to send
    localip:
       The local IP to send the message from 
    localport:
       The local port to send the message from

  <Exceptions>
    AddressBindingError (descends NetworkError) when the local IP isn't
      a local IP.

    ResourceForbiddenError (descends ResourceException) when the local
      port isn't allowed

    RepyArgumentError when the local IP and port aren't valid types
      or values

    AlreadyListeningError if there is an existing listening UDP socket
    on the same local IP and port.

    DuplicateTupleError if there is another sendmessage on the same
    local IP and port to the same remote host.

  <Side Effects>
    None.

  <Resource Consumption>
    This operation consumes 64 bytes + number of bytes of the message that
    were transmitted. This requires that the localport is allowed.

  <Returns>
    The number of bytes sent on success
"""
```
 
##### openconnection(destip, destport, localip, localport, timeout)
----
(rename of ~~openconn~~)

Open a TCP connection to a remote computer, returning a socket object.   There is a timeout value that can be set to limit the amount of time the system will wait for a response before abandoning the attempt to connect.

 * Doc string:
```python
"""
  <Purpose>
    Opens a connection, returning a socket-like object


  <Arguments>
    destip: The destination ip to open communications with

    destport: The destination port to use for communication

    localip: The local ip to use for the communication

    localport: The local port to use for communication

    timeout: The maximum amount of time (in seconds) to wait to connect.   This may
             be a floating point number or an integer


  <Exceptions>

    RepyArgumentError if the arguments are invalid.   This includes both
    the types and values of arguments. If the localip matches the destip,
    and the localport matches the destport this will also be raised.

    AddressBindingError (descends NetworkError) if the localip isn't 
    associated with the local system or is not allowed.

    ResourceForbiddenError (descends ResourceError) if the localport isn't 
    allowed.

    DuplicateTupleError (descends NetworkError) if the (localip, localport, 
    destip, destport) tuple is already used.   This will also occur if the 
    operating system prevents the local IP / port from being used.

    AlreadyListeningError if the (localip, localport) tuple is already used
    for a listening TCP socket.

    CleanupInProgressError if the (localip, localport, destip, destport) tuple is
    still being cleaned up by the OS.

    ConnectionRefusedError (descends NetworkError) if the connection cannot 
    be established because the destination port isn't being listened on.

    TimeoutError (common to all API functions that timeout) if the 
    connection times out

    InternetConnectivityError if the network is down, or if the host
    cannot be reached from the local IP that has been bound to.

  <Resource Consumption>
    This operation consumes 64*2 bytes of netsend (SYN, ACK) and 64 bytes 
    of netrecv (SYN/ACK). This requires that the localport is allowed. Upon 
    success, this call consumes an outsocket.

  <Returns>
    A socket-like object that can be used for communication. Use send, 
    recv, and close just like you would an actual socket object in python.
"""
```

##### socket.close()
----
Closes the socket.   Any further local calls to recv / send will result in an exception.

 * Doc string:
```python
"""
    <Purpose>
    Closes a socket.   Pending remote recv() calls will return with the 
    remaining information.   Local recv / send calls will fail after this.

  <Arguments>
    None

  <Exceptions>
    None

  <Side Effects>
    Pending local recv calls will either return or have an exception.

  <Resource Consumption>
    If the connection is closed, no resources are consumed. This operation
    uses 64 bytes of netrecv, and 128 bytes of netsend.
    This call also stops consuming an outsocket.

  <Returns>
    True if this is the first close call to this socket, False otherwise.
"""
```

##### socket.recv(numbytes)
----
Receives data that was sent by the connected party using send.   Note that this may return less than `numbytes` worth of data.  Also, note that if the other party does ```s.send('hello'); s.send('Guten Tag')```, the party who calls recv may get `'helloGuten Tag'`, `'h'`, or any other prefix of the total data.   

 * Doc string:
```python
"""
  <Purpose>
    Receives data from a socket.   It may receive fewer bytes than 
    requested.   

  <Arguments>
    numbytes: 
       The maximum number of bytes to read; must be at least 1.

  <Exceptions>
    SocketClosedLocal is raised if the socket was closed locally.

    SocketClosedRemote is raised if the socket was closed remotely.

    SocketWouldBlockError is raised if the socket operation would block.

    RepyArgumentError if `numbytes` is less than 1, not an `int`, or missing.

  <Side Effects>
    None.

  <Resource Consumptions>
    This operations consumes 64 + amount of data  in bytes
    worth of netrecv, and 64 bytes of netsend.

  <Returns>
    The data received from the socket (as a string).
"""
```
 
##### socket.send(message)
----
Sends data to the connected party.   Note that this may send less than the entire message.   If the connection is disconnected, there is no guarantee that the other party was able to recv the data.  If the other party doesn't call recv, send may raise an exception if the sender and receiver buffers fill (the buffer size is undefined).   If the other party closes the connection, send will raise an exception.

 * Doc string:
```python
"""
  <Purpose>
    Sends data on a socket.   It may send fewer bytes than requested.   

  <Arguments>
    message:
      The string to send.

  <Exceptions>
    SocketClosedLocal is raised if the socket is closed locally.

    SocketClosedRemote is raised if the socket is closed remotely.

    SocketWouldBlockError is raised if the operation would block.

    RepyArgumentError if `message` is not a string, or missing.

  <Side Effects>
    None.

  <Resource Consumption>
    This operations consumes 64 + size of sent data of netsend and
    64 bytes of netrecv.

  <Returns>
    The number of bytes sent.   Be sure not to assume this is always the 
    complete amount!
"""
```

##### listenforconnection(localip, localport)
---- 
Binds to an IP and port and waits for incoming TCP connections.  If this function is called multiple times on the same ip and port without the first tcpserversocket being closed, the second call will have an exception.  These ports are separate from the message ports and so both a message and connection listener can use the same port.   This call raises an exception instead of blocking.

* Doc string: 
```python
"""
  <Purpose>
    Sets up a TCPServerSocket to recieve incoming TCP connections. 

  <Arguments>
    localip:
        The local IP to listen on
    localport:
        The local port to listen on

  <Exceptions>
    Raises AlreadyListeningError if another TCPServerSocket or process has bound
    to the provided localip and localport.

    Raises DuplicateTupleError if another process has bound to the
    provided localip and localport.

    Raises RepyArgumentError if the localip or localport are invalid

    Raises ResourceForbiddenError if the ip or port is not allowed.

    Raises AddressBindingError if the IP address isn't a local ip.

  <Side Effects>
    The IP / Port combination cannot be used until the TCPServerSocket
    is closed.

  <Resource Consumption>
    Uses an insocket for the TCPServerSocket.

  <Returns>
    A TCPServerSocket object.
"""
```

##### tcpserversocket.getconnection()
----
Receives a connection that was initiated to an IP and port.   waitforconn can be built in a user level library on top of this.

 * Doc string:
```python
"""
  <Purpose>
    Accepts an incoming connection to a listening TCP socket.

  <Arguments>
    None

  <Exceptions>
    Raises SocketClosedLocal if close() has been called.

    Raises SocketWouldBlockError if the operation would block.

    Raises ResourcesExhaustedError if there are no free outsockets.

  <Resource Consumption>
    If successful, consumes 128 bytes of netrecv (64 bytes for
    a SYN and ACK packet) and 64 bytes of netsend (1 ACK packet).
    Uses an outsocket.

  <Returns>
    A tuple containing: (remote ip, remote port, socket object)
"""
```
 
##### tcpserversocket.close()
----
Closes the socket.   Any further local calls to getconnection will result in an exception.

 * Doc string:
```python
"""
  <Purpose>
    Closes the listening TCP socket.

  <Arguments>
    None

  <Exceptions>
    None

  <Side Effects>
    The IP and port can be re-used after closing.

  <Resource Consumption>
    Releases the insocket used.

  <Returns>
    True, if this is the first call to close.
    False otherwise.
"""
```
 
##### listenformessage(localip, localport)
---- 
Binds to an IP and port and waits for incoming UDP messages.  If this function is called multiple times on the same ip and port without the first udpserversocket being closed, the second call will have an exception.  These ports are separate from the connection ports and so both a message and connection listener can use the same port.   This call will raise an exception if it would block.

* Doc string: 
```python
""" 
  <Purpose>
      Sets up a UDPServerSocket to receive incoming UDP messages.

  <Arguments>
      localip:
          The local IP to register the handler on.
      localport:
          The port to listen on.

  <Exceptions>
      DuplicateTupleError (descends NetworkError) if the port cannot be
      listened on because some other process on the system is listening on
      it.

      AlreadyListeningError if there is already a UDPServerSocket with the same
      IP and port.

      RepyArgumentError if the port number or ip is wrong type or obviously
      invalid.

      AddressBindingError (descends NetworkError) if the IP address isn't a
      local IP.

      ResourceForbiddenError if the port is not allowed.

  <Side Effects>
      Prevents other UDPServerSockets from using this port / IP

  <Resource Consumption>
      This operation consumes an insocket and requires that the provided messport is allowed.

  <Returns>
      The UDPServerSocket.
"""
```

##### udpserversocket.getmessage()
----
Receives a message that was sent to an IP and port.   recvmess can be built in a user level library on top of this.

 * Doc string:
```python
"""
  <Purpose>
      Obtains an incoming message that was sent to an IP and port.

  <Arguments>
      None.

  <Exceptions>
      SocketClosedLocal if UDPServerSocket.close() was called.

      Raises SocketWouldBlockError if the operation would block.

  <Side Effects>
      None

  <Resource Consumption>
      This operation consumes 64 + size of message bytes of netrecv

  <Returns>
      A tuple consisting of the remote IP, remote port, and message.
"""
```
 
##### udpserversocket.close()
----
Closes the udp server socket.   Any further local calls to getmessage will result in an exception.

 * Doc string:
```python
"""
  <Purpose>
      Closes a socket that is listening for messages.

  <Arguments>
      None.

  <Exceptions>
      None.

  <Side Effects>
      The IP address and port can be reused by other UDPServerSockets after
      this.

  <Resource Consumption>
      If applicable, this operation stops consuming the corresponding
      insocket.

  <Returns>
      True if this is the first close call to this socket, False otherwise.
"""
```

#### File functions

##### openfile(filename, create)
----

(Replaces: ~~open~~, ~~file~~.)

Open a file, returning an object of the file type. 

Filenames may only be in the current directory and may only contain lowercase letters, numbers, the hyphen, underscore, and period characters.  Also, filenames cannot be '.', '..', the blank string or start with a period.   There is no concept of a directory or a folder in repy.  Filenames must be no more than 120 characters long.

Every file object is capable of both reading and writing. 

If create is True, the file is created if it does not exist.   Neither mode truncates the file on open.

```python
"""
 * Doc string:
  <Purpose>
    Allows the user program to open a file safely. 

  <Arguments>
    filename:
      The file that should be operated on. It must not contain 
      characters other than 'a-z0-9.-_' and cannot start with a '.' 
      or be the empty string.

    create:
       A Boolean flag which specifies if the file should be created
       if it does not exist.   If the file exists, this flag has no effect.

  <Exceptions>
    RepyArgumentError is raised if the filename is invalid or create is not a boolean type.

    FileInUseError is raised if a handle to the file is already open.

    ResourceExhaustedError is raised if there are no available file handles.

    FileNotFoundError is raised if the filename is not found, and create is False.


  <Side Effects>
    Opens a file on disk, uses a file descriptor.

  <Resource Consumption>
    Consumes 4K of fileread. If the file does not exist, and is created, 4K of filewrite is used.
    If a handle to the object is created, then a file descriptor is used.

  <Returns>
    A file-like object.
"""
```

##### file.close()
----

Close the file. A closed file cannot be read or written any more. Any operation which requires that the file be open will raise a FileClosedError after the file has been closed. 

 * Doc string:
```python
"""
  <Purpose>
    Allows the user program to close the handle to the file.

  <Arguments>
    None.

  <Exceptions>
    FileClosedError is raised if the file is already closed.

  <Resource Consumption>
    Releases a file handle.

  <Returns>
    None.
"""
```

##### file.readat(sizelimit, offset)
----

Seek to a location in a file and reads up to sizelimit bytes from the file, returning what is read. If sizelimit is None, the file is read to EOF.

 * Doc string:
```python
"""
  <Purpose>
    Reads from a file handle. Reading 0 bytes informs you if you have read
    past the end-of-file, but returns no data.

  <Arguments>
    sizelimit: 
      The maximum number of bytes to read from the file. Reading EOF will read less. By setting this value to None, the entire file is read.
    offset:
      Seek to a specific absolute offset before reading.

  <Exceptions>
    RepyArgumentError is raised if the offset or size is negative.

    FileClosedError is raised if the file is already closed.

    SeekPastEndOfFileError is raised if trying to read past the end of the file.

  <Resource Consumption>
    Consumes 4K of fileread for each 4K aligned-block of the file read.
    All reads will consume at least 4K.

  <Returns>
    The data that was read. This may be the empty string if we have reached the
    end of the file, or if the sizelimit was 0.
"""
```

##### file.writeat(data, offset)
----

Seek to the offset in the file and then write some data to a file.

 * Doc string:
```python
"""
  <Purpose>
    Allows the user program to write data to a file.

  <Arguments>
    data:
      The data to write
    offset:
      An absolute offset into the file to write

  <Exceptions>
    RepyArgumentError is raised if the offset is negative or the data is not
    a string.

    FileClosedError is raised if the file is already closed.

    SeekPastEndOfFileError is raised if trying to write past the EOF.


  <Side Effects>
    Writes to persistent storage.

  <Resource Consumption>
    Consumes 4K of filewrite for each 4K aligned-block of the file written.
    All writes consume at least 4K.

  <Returns>
    Nothing
"""
```

##### listfiles()
----
Returns a list of file names for the files in the VM.   

 * Doc string:
```python
"""
  <Purpose>
    Returns a list of files accessible to the program.

  <Arguments>
    None

  <Exceptions>
    None

  <Side Effects>
    None

  <Resource Consumption>
    Consumes 4K of fileread.

  <Returns>
    A list of strings (file names)
"""
```


##### removefile(filename)
----
Deletes a file in the VM.   If the file does not exist, an exception is raised.

 * Doc string:
```python
"""
  <Purpose>
    Allows the user program to remove a file in their area.

  <Arguments>
    filename: the name of the file to remove.  It must not contain 
      characters other than 'a-z0-9.-_' and cannot start with a '.' 
      or be the empty string.

  <Exceptions>
    RepyArgumentError is raised if the filename is invalid.

    FileNotFoundError is raised if the file does not exist

    FileInUseError is raised if the file is already open.

  <Side Effects>
    None

  <Resource Consumption>
    Consumes 4K of fileread, and 4K of filewrite if successful.

  <Returns>
    None
"""
```

#### Threading functions
##### createlock()
----
Returns a lock object that can be used for mutual exclusion and critical section protection.

 * Doc string:
```python
"""
  <Purpose>
    Returns a lock object to the user program.    A lock object supports
    two functions: acquire and release.

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    None.

  <Resource Consumption>
    None.

  <Returns>
    The lock object.
"""
```

##### lock.acquire(blocking)
----
Blocks until the lock is available, then takes it (lock is an object obtained by calling createlock()).

If the "blocking" argument is False, the method returns False immediately instead of waiting to acquire the lock; if the lock is available it takes it and returns True, as if it were called with no argument.

 * Doc string:
```python
"""
  <Purpose>
    Acquires a lock.

  <Arguments>
    blocking - if False, returns immediately instead of waiting to acquire the lock.

  <Exceptions>
    None.

  <Side Effects>
    Locks the object.

  <Resource Consumption>
    None.

  <Returns>
    True if the lock was acquired, False otherwise.
"""
```

##### lock.release()
----
Releases the lock. This call raises an exception (?) if the lock is already unlocked.

 * Doc string:
```python
"""
  <Purpose>
    Release a lock.

  <Arguments>
    None.

  <Exceptions>
    LockDoubleReleaseError if release() is called on an unlocked lock.

  <Side Effects>
    Unlocks the object.

  <Resource Consumption>
    None.

  <Returns>
    None.
"""
```

##### createthread(function)
----
Armon: Removed the "args" argument. We only accept a function. If they want to call a function with arguments,
they can define a closure around that. This simplifies the API, without sacrificing functionality.

Start a new thread to call a function.   The thread is charged to your program.

 * Doc string:
```python
"""
  <Purpose>
    Cause the execution of a function in another thread.   The current thread 
    of execution will also proceed.

  <Arguments>
    function:
       The function to call

  <Exceptions>
    RepyArgumentError if the function is not callable.

    ResourceExhaustedError if there are no events available.

  <Side Effects>
    Starts a new thread

  <Resource Consumption>
    This operation consumes an event.

  <Returns>
    None
"""
```

##### sleep(seconds)
----

Sleeps the current thread for some time (waits for a specific time before executing any further instructions).   This thread will not consume CPU cycles during this time.

 * Doc string:
```python
"""
  <Purpose>
    Allow the current thread to pause execution (similar to time.sleep()).
    This function will not return early for any reason

  <Arguments>
    seconds:
       The number of seconds to sleep.   This can be a floating point value

  <Exceptions>
    None.

  <Side Effects>
    None.

  <Resource Consumption>
    None.

  <Returns>
    None.
"""
```

##### getthreadname()
----
(added call)

Returns a unique name that indicates the thread's name.   Don't use this to derive any meaning other than string uniqueness.

 * Doc string:
```python
"""
  <Purpose>
    Returns a string identifier for the currently executing thread.
    This identifier is unique to this thread.

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    None.

  <Resource Consumption>
    None.

  <Returns>
    A string identifier.
"""
```

  
#### Miscellaneous functions
##### log(*args)
Prints output to the console

 * Doc string:
```python
"""
  <Purpose>
    Used to store program output. Prints output to the console by default.

  <Arguments>
    Takes a variable number of arguments to print. They are wrapped in str(), so it is not necessarily a string.

  <Exceptions>
    None

  <Returns>
    Nothing
"""
```



##### getruntime()
----
Returns a float containing the number of seconds the program has been running.   This time is guaranteed to be monotonic.

 * Doc string:
```python
"""
  <Purpose>
    Return the amount of time the program has been running.   This is in
    wall clock time. This is guaranteed to be monotonic.

  <Arguments>
    None

  <Exceptions>
    None.

  <Side Effects>
    None

  <Resource Consumption>
    None.

  <Returns>
    The elapsed time as float
"""
```

##### randombytes()
----

Returns a string of random bytes of data derived from a hardware source of randomness.   The size of the string will be equal to 1024.   It is assumed that a client library that regularly needs small amounts of random data will cache data received from this call and return smaller values.

 * Doc string:
```python
"""
  <Purpose>
    Return a string of random bytes with length 1024

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    This function is metered because it may involve using a hardware
    source of randomness.

  <Resource Consumption>
    This operation consumes 1024 bytes of random data.

  <Returns>
    The string of bytes.
"""
```

##### exitall()
----
Terminates the program immediately.   The program will not execute the "exit" callfunc or finally blocks.

 * Doc string:
```python
"""
  <Purpose>
    Allows the user program to stop execution of the program without
    passing an exit to the main program or calling finally blocks.

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    There isn't a guaranteed time which this function is realized.   This means that
    other threads may execute while this call is being processed.

  <Resource Consumption>
    None.

  <Returns>
    The current thread does not resume after exit
"""
```

##### createvirtualnamespace(code, name)
----
(added call)
Wraps any arbitrary code after performing a safety evaluation. Provides a method to evaluate the code.

 * Doc string:
```python
"""
  <Purpose>
   Returns a virtualnamespace object which supports evaluation of the
   code in an arbitrary global context.

  <Arguments>

   code:  The string code to check for safety and store for future evaluation.

   name: The name for this module. Used during initialization only, and so
   a stack trace can show the name of the module instead of just <string>.


  <Exceptions>
   A CodeUnsafeError will be raised if the static code analysis fails. This can be due to unsafe constructs,
   invalid syntax, or an evaluation timeout.

  <Side Effects>
   Other calls to createvirtualnamespace() will wait until the first completes since safety checks are performed serially.

   Another process will be launched to check the code for safety. This has memory and CPU ramifications,
   since we will not account for it's resource consumption. The memory use will likely exceed what is allowed
   per VM, since the memory used seems to grow faster-than-linarly with respect to LOC. 

   It usually takes 0.2-0.3 seconds to launch a process, pipe the code, evaluate for safety, and return.

  <Resource Consumption>
    None

  <Returns>
   A virtualnamespace object
"""
```
 
##### virtualnamespace.evaluate(context)
----
(added call)
Evaluates the code wrapped by the virtualnamespace in a given context.

 * Doc string:
```python
"""
  <Purpose>
    Evaluates the wrapped code within a context.

  <Arguments>
    context: A global context to use when executing the code.
    This should be a SafeDict object, but if a dict object is provided
    it will automatically be converted to a SafeDict object.

  <Exceptions>
    A RepyArgumentError exception will be raised if the provided context is not
    a dictionary or safe dictionary object 

    ContextUnsafeError is raised if the context is a dict but cannot be converted into a SafeDict.

    Any that may be raised by the code that is being evaluated.


  <Returns>
    The context dictionary that was used during evaluation.
    If the context was a dict object, this will be a new
    SafeDict object. If the context was a SafeDict object,
    then this will return the same context object.
"""
```


 
##### getresources()
----
(added call)
Determines the resource usage limits and current usage, and provides info on the last 100 times that repy was stopped.

 * Doc string:
```python
"""
  <Purpose>
    Returns the resource usage limits and the current usage and an array containing information about the  last 100 stoptimes.

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    Calls to this function will be serialized.

  <Resource Consumption>
    None

  <Returns>
    A tuple of  (limits, usage, stoptimes). Limits is a dictionary which contains the maximum utilization
    of a resource. It contains all of the resources as defined in the restrictions file. The user can also
    determine the allowed conn and mess ports by checking for their respective entries.

    The usage dictionary contains the current usage of all the resources. It has all the same entries as
    the limits dictionary, but has an additional "threadcpu" key, which is the amount of CPU time in seconds
    the current thread has executed for.

    stoptimes is an array of tuples. Each tuple contains the time that repy was stopped (WRT/ getruntime)
    and for how many seconds it was stopped. This array holds at most the latest 100 entries.
"""
```

 
##### getlasterror()
----
TODO: others should review this, change its name as needed, etc.

 * Doc string:
```python
"""
  <Purpose>
    Obtain information about the last error (exception) that occurred in the current thread.

  <Arguments>
    None.

  <Exceptions>
    None.

  <Side Effects>
    None.

  <Resource Consumption>
    None.

  <Returns>
    A string with details of the last exception that occurred in the current thread, or None if there is no such exception.
"""
```