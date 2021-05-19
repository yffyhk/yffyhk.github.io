simple chatroom by aws lambda + websocket gateway

//onDisconnect

//When one client disconnect , notice other connecting client
```
const AWS = require('aws-sdk');
var ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = function(event, context, callback) {
  
  const apigwManagementApi = new AWS.ApiGatewayManagementApi({
    apiVersion: '2018-11-29',
    endpoint: event.requestContext.domainName + '/' + event.requestContext.stage
  });
  
  let id = event.requestContext.connectionId;
  
  deleteID(id).then(
  getConnection()).then(function(c){ 
    
    if(c.Items.length == 0) return;
    
    var ids = c.Items.map(x => x.ConnectionId);
    
    var obj = { type: "Enter", message: ids };
    var buf = Buffer.from(JSON.stringify(obj));
    
    ids.forEach(async function(id){

      await apigwManagementApi.postToConnection({ 
        ConnectionId: id, 
        Data: buf
      }).promise();
      
    });
  });
  
  return{};
};

async function getConnection(){
  var params = {  TableName : 'ChatRoomTable',  };
  return await ddb.scan(params).promise();
}


async function deleteID(ID) {
    return await ddb.delete({
        TableName:'ChatRoomTable',
        Key:{ ConnectionId: ID}
    }).promise();
}
```
//onEnter

//When one client connect , notice other connecting client
```
const AWS = require('aws-sdk');
var ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = function(event, context, callback) {
  
  const apigwManagementApi = new AWS.ApiGatewayManagementApi({
    apiVersion: '2018-11-29',
    endpoint: event.requestContext.domainName + '/' + event.requestContext.stage
  });
  
  getConnection().then(function(c){ 
    var ids = c.Items.map(x => x.ConnectionId);
    
    var obj = { type: "Enter", message: ids };
    var buf = Buffer.from(JSON.stringify(obj));
    
    ids.forEach(async function(id){

      await apigwManagementApi.postToConnection({ 
        ConnectionId: id, 
        Data: buf
      }).promise();
      
    });
  });
  
  callback(null, {'statusCode':200});
};

async function getConnection(){
  var params = {  TableName : 'ChatRoomTable',  };
  return await ddb.scan(params).promise();
}
```
//onConnect

//When one client connect , add info. to database
```
var AWS = require('aws-sdk');
var ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = function(event, context, callback) {

  let id = event.requestContext.connectionId;
  addID(id);
  
  callback(null, {'statusCode':200});
};


async function addID(ID) {
    
    var params = {  TableName:'ChatRoomTable',
                    Item:{ ConnectionId: ID}  };
                    
    return await ddb.put(params).promise();
}
```
//onMessage

//When one client speak , notice other connecting client 

//*There is no message save in database

```
const AWS = require('aws-sdk');
var ddb = new AWS.DynamoDB.DocumentClient();

exports.handler = function(event, context, callback) {

  const apigwManagementApi = new AWS.ApiGatewayManagementApi({
    apiVersion: '2018-11-29',
    endpoint: event.requestContext.domainName + '/' + event.requestContext.stage
  });
  
  var senderID = event.requestContext.connectionId;
  
  var msg = JSON.parse(event.body).data;
  
  var obj = { type: "Message", message: `${senderID}> ${msg}` };
  var buf = Buffer.from(JSON.stringify(obj));
 
 
 
  getConnection().then((c) => c.Items.forEach(
    async function(item){
      let id = item.ConnectionId;
      
      await apigwManagementApi.postToConnection({ 
        ConnectionId: id, 
        Data: buf
      }).promise();

  }));
  
  callback(null, {'statusCode':200});
};

async function getConnection(){
  var params = {  TableName : 'ChatRoomTable',  };
  return await ddb.scan(params).promise();
}
```


