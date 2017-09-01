title: Unity NetworkServer and NetworkClient Tutorial (LLAPI)
author: Austin Jeane
date: 2017-08-29 15:25:30
banner: https://static1.squarespace.com/static/5431b49ae4b0e9e639ec442d/t/57499d99d51cd43e7b7754bb/1464442268020/cool-youtube-banners-black-and-white_295911.png
tags:
---





# Unity NetworkServer and NetworkClient Tutorial (LLAPI)

## What is NetworkServer and NetworkClient?

Unity's NetworkServer and NetworkClient are higher level than the Network Transport Layer API and lower level than using NetworkBehavior or NetworkManager.   

![test](/images/NetworkLayers.png)  

Using the NetworkServer and NetworkClient allow you to take advantage of Unity's Network serialization without conforming to the use of their higher level concepts like NetworkManager. This tutorial will show you how to set up a basic server that can connect to multiple clients, and send custom data back and forth between them. 

## Setting Up the Server

#### Server.cs 

``` c#
using UnityEngine;
using UnityEngine.Networking;
 
public class Server : MonoBehaviour {

    void Start () {
        SetupServer();
    }
 
    public void SetupServer()
    {
        NetworkServer.Listen(4444);
 
        NetworkServer.RegisterHandler(MsgType.Connect, OnPlayerConnect);
    }
 
    public void OnConnected(NetworkMessage netMsg)
    {
        Debug.Log("Connected to client");
    }
}
```

We activate the server by calling NetworkServer.Listen and pass in 4444 as the port. Any port number will do, just make sure not to use ports that are already used by other things, such as the mail port (25 and 587, some server hosts block these).   

Next We register handlers to receive events from the client. We can do that by calling NetworkServer.RegisterHandler.  The first parameter is the MsgType (int). Unity.Networking comes with a lot of pre-made MsgTypes such as "Connect" and "Disconnect" (see [here](https://docs.unity3d.com/ScriptReference/Networking.MsgType.html) for list of other types). We can also create out own custom MsgTypes. The second parameter is the method that will handle the event. In this case we created the OnConnected method, but the method name can be whatever you want. These handlers always have the same signature and take one parameter of NetworkMessage.

## Setting Up the Client

#### Client.cs (If client is not the server)

``` c#
using UnityEngine;
using UnityEngine.Networking;
 
public class Client : MonoBehaviour {
 
    void Start () {
        SetupClient();
    }
 
    public void SetupClient()
    {
        myClient = new NetworkClient();
 
        myClient.RegisterHandler(MsgType.Connect, OnConnected);
 
        myClient.Connect("127.0.0.1", 4444);
    }
 
    public void OnConnected(NetworkMessage netMsg)
    {
        Debug.Log("Connected to server");
    }
}

```

#### LocalClient.cs (If client is also server)

``` c#
using UnityEngine;
using UnityEngine.Networking;
 
public class LocalClient : MonoBehaviour {
 
    void Start () {
        SetupServer();
        SetupClient();
    }

    public void SetupServer()
    {
        NetworkServer.Listen(4444);
 
        NetworkServer.RegisterHandler(MsgType.Connect, OnPlayerConnect);
    }
 
    public void SetupClient()
    {
        myClient = ClientScene.ConnectLocalServer(); 
 
        myClient.RegisterHandler(MsgType.Connect, OnConnected);
    }
 
    public void OnConnected(NetworkMessage netMsg)
    {
        Debug.Log("Connected to local server");
    }
}

```


We activate the client by creating a new instance of NetworkClient, registering our  handlers, then connecting to the server. The Connect method takes in 2 parameters: ip address and port number. The ip address is the ip address of the server. Ours is "127.0.0.1" because that is the ip address for your local machine (I am assuming you run the client and server on the same machine).   

If your server is in the same project as the client, then look at LocalClient.cs.   

The MsgTypes that we register here are the events that we want to be able to receive from the server. Right now we are only listening for Unity's internal Connect MsgType. Lets create a custom type and send some data from client to server and back.  

## Creating Custom MsgTypes

Unity Networking gives us the ability to create custom MsgTypes and send our own data with the message. First create a new file for our MsgTypes, I'll name mine NetworkMessages.cs. Then use the code below as a template to create your own custom messages. If your client and server are in different projects, this file will need to be in both of them.

``` c#
using UnityEngine;
using UnityEngine.Networking;
 
public static class MyMsgType
{
	public static short RegisterPlayerWithServer = MsgType.Highest + 1;
};
 
public class RegisterPlayerWithServerMessage : MessageBase
{
	public string PlayerName;
}
 
```

Since MsgTypes are int (each type is a unique number), to add new ones we need to know the highest number that unity uses for their internal MsgTypes and add 1 to that. This is why we use "MsgType.Highest + 1" to create our first customer MsgType. For our next type we will use "MsgType.Highest + 2" and so on.
 
From [Unity docs on MsgType.Highest](https://docs.unity3d.com/ScriptReference/Networking.MsgType.Highest.html):
>The highest value of built-in networking system message ids. User messages must be above this value.

Below our MsgTypes we have created the custom message RegisterPlayerWithServerMessage that will hold our data that we're sending. Our message just contains the string PlayerName for now. We need to inherit from the MessageBase class because it contains Serialize and Deserialize methods for serialization. You can overwrite these methods with your own implementation or use Unity's default way. 

From [Unity docs on Network Messages](https://docs.unity3d.com/Manual/UNetMessages.html):
>Message classes can contain members that are basic types, structs, arrays, and most of the common Unity Engine types such as Vector3. They cannot contain members that are complex classes or generic containers.

### Receiving Custom Messages

To use this new MsgType, we need to register the handler for it. This message will only be received by the server so we will put it in Server.cs. Your setup client method should now look something like this:

 ``` c#
public void SetupClient()
{
    myClient = ClientScene.ConnectLocalServer(); 
 
    myClient.RegisterHandler(MsgType.Connect, OnConnected);
    myClient.RegisterHandler(MsgType.RegisterPlayerWithServer, OnRegisterPlayerWithServer);
}
 ```

 We also need to add the new method handler "OnRegisterPlayerWithServer":

 ``` c#
public void OnRegisterPlayerWithServer(NetworkMessage netMsg)
{
    var registerMessage = msg.ReadMessage<RegisterPlayerWithServerMessage>();
     
    Debug.Log(string.Format("{0} connected to server", registerMessage.PlayerName));
}
 ```

In our method handler we cast the netMsg to the custom type that we created. Once we do that we are able to get the PlayerName field.

### Sending Custom Messages

To send our new message from the client we use this code: 

``` c#
var message = new RegisterPlayerWithServerMessage
{
    PlayerName = "austin jeane"
};
 
myClient.Send(MyMsgType.RegisterPlayerWithServer, message);
```

The server should print out _austin jeane connected to server_.

### Reliable vs Unreliable Messages

In the above example, I send my Register Player message using _myClient.Send_. Messages sent using the send method are send through the "Reliable" channel, meaning that "Each message is guaranteed to be delivered but not guaranteed to be in order". You should send messages through the reliable channel that you require to be delivered, such as registering a player or requesting to join a game. Even though the message is guaranteed to be delivered, it is not the fastest way to send a message.

The fastest way to send a message is through the "Unreliable" channel. To send messages through this channel you use the method _SendUnreliable_ instead of _Send_. Sending messages through the unreliable channel are faster than reliable messages but are not guaranteed to be delivered. This is ideal for fast paced games and is often used for things such as updating users positions.

There are other channels as well, but these 2 are the most used. 

## Conclusion

From here you should be able create all the custom MsgTypes that you need to make a complete game. In the next tutorial I will turn this bare bones server and client into a top down multiplayer game, with authoritative server movement, client side prediction, and server reconciliation. 


-