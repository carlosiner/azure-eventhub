
This is not a agent, but its a general guide of how to sent events from Azure. 
You can follow this instructions to send events from an EventHub to Devo platform.
There are two kinds of events that will be sent from Azure to Devo: from Monitor and from Active Directory.

## Prerequisites
- Have a Azure account with the permissions
- Have a Devo account


# Create a EventHub

## Creating the namespace

- Click on _Create a resource_ on the left side, find and select the _Event Hubs_ resource and click on _Create_ button.

![alt text](resources/step_1.png)

- Fill the fields with the corresponding values. Maybe you will need to create a new _Resource group_ if you haven't.
Click _Create_. This will take few seconds.

![alt text](resources/step_2.png)

- Once the namespace is created you can access to it clicking in _All resources_ on the right side of menu and then in the name space.

![alt text](resources/step_3.png)

## Creating the Event Hubs

- Click on _Monitor_ option on left side menu, then _Activity Log_ and then _Export to Event Hub_ option.

![alt text](resources/step_4.png)

- Select the corresponds options with the susbcription, namespace and regions. Make sure to check _Export to an event hub_ option. 
Then save the changes.

![alt text](resources/step_5.png)

This can take several minutes. Once the event hub is created you can you it in the namespace resource associated.

![alt text](resources/step_6.png)


## Creating the Function App
- Click on _Create a resource_ option on left side menu, then find and select _Function App_ option. Then click _Create_.

![alt text](resources/step_7.png)

- Fill and select the fields corresponding your requirements. Make sure to select _JavaScript_ in _Runtime Stack_ option. 
Click _Create_. This can take several seconds.

![alt text](resources/step_8.png)

- Once that it was created you can check it on _All resources_ option. Select the function app and then click on _*+*_ icon in _Functions_ option.
Choose _In-portal_ option has development enironment, and the click in _Continue_ button.

![alt text](resources/step_9.png)

- Choose _More templates..._ option and then _Finish and view templates_ button.

![alt text](resources/step_10.png)

- Choose the _Azure Event Hub trigger_. It could ask you for install a extension. Install it.

![alt text](resources/step_12.png)

- Fill and select the field according your requirements. In _Event Hub connection_ you will need to select the namespace associated.

![alt text](resources/step_13.png)
![alt text](resources/step_14.png)

- Once the function app was created you should show something like the following image.

![alt text](resources/step_15.png)

In the right side you can see two files and can run the tests. 
In the bottom you can see the console and the logs generated. 
And, in the left side you can see the function app structure.

Now, you need to send all the events to Devo. You will need to upload the credentials of Devo. 
In your computer create a folder with the name _certs_, paste the credential in this folder and compress this folder to _zip_ format.
Then select _upload_ option on the right side and select this zip file. 
Upload the _package.json_ file contained in this tutorial.

The structure of your event hub should looks like the following image

![alt text](resources/step_16.png)

Unzip _certs.zip_ file from the console. 

````bash
> unzip certs.zip
````

You can delete the _zip_ file.

````bash
> rm certs.zip
````

Install the devo js SDK and all dependencies. This will generate a new folder (_node_modules_) with the packages.

```bash
npm install
```

Now, you need to update _index.js_ file according to send the events to Devo from the event hub. 

Copy the _index.js_ file content from this tutorial and paste it in the _index.js_ file of your event hub.

````javascript
const devo = require('@devo/nodejs-sdk');
const fs = require('fs');

module.exports = async function (context, eventHubMessages) {
    context.log(`JavaScript eventhub trigger function called for message array ${eventHubMessages}`);
    let zone = 'eu';
    let senders = {};
    let default_opt = {
        host: "eu.elb.relay.logtrust.net",
        port: 443,
        ca: fs.readFileSync(__dirname+"/certs/chain.crt"),
        cert: fs.readFileSync(__dirname+"/certs/mydomain.crt"),
        key: fs.readFileSync(__dirname+"/certs/mydomain.key")
    };
    let options = {
        'AuditLogs': `cloud.azure.ad.audit.${zone}`,
        'SignInLogs': `cloud.azure.ad.signin.${zone}`,
        'Delete': `cloud.azure.activity.delete.${zone}`,
        'Action': `cloud.azure.activity.events.${zone}`,
        'Write': `cloud.azure.activity.write.${zone}`,
        'default': `my.app.azure.losteventhublogs`
    };
    
    for (let opt in options) {
        let conf = Object.assign({}, default_opt);
        conf['tag'] = options[opt];
        senders[opt] = devo.sender(conf);
    };
    
    eventHubMessages.forEach(message => {
        if (message.constructor !== Object || (message.constructor === Object && !message['records'] )) {
            senders['default'].send(JSON.stringify(message));
        } else {
            for (let m of message.records) {
                if (options[m.category]){
                    senders[m.category].send(m);
                } else {
                    senders['default'].send(m);
                }
                
            }
        }
    });
    
    context.done();    
};
````