# serial-synapse
tldr - Orchestrates serial communication to MCU device utilizing CmdMessenger protocol

This sounds a lot more complicated than it is, so I will have some easy to use example code soon!

# Install
```
npm install serial-synapse
```

## Update Handlers vs Commands
*serial-synapse* implements both **commands** and **update handlers**.

* **Commands** are executed at the behest of the master, triggering functionality/a response from the secondary.
* **Commands** can either be silent (no response generated) or responsive (default, a response will occur to this command, feeding back data or an acknowledgement of command execution)
* **Update Handlers** have a function triggered when the secondary fires back information with its identifier. A good example would be a sensor detecting collision, or periodic updates of an accelerometer or distance sensor.

##Messaging Protocol
Wherein we discuss the expected communication protocol between master and secondary, where *master* is your node process and *secondary* is your MCU/Arduino/robot/IoT Toaster:

###From master (node) to secondary (MCU):
```
identifier,uuid,args,args,args;
```
* *identifier* - the enum identifying what function to execute
* *uuid* - this is generated by *serial-synapse* if the function is not "silent", and needs to be echoed back first by the MCU so we know it's responding to the command.
* *args*, args, args - *n* arguments, from 0 to any number, that your function on the MCU may need to execute
* **;** - CmdMessenger parses messages by scanning for the **;**, so we include this at the end of our message.

For example:
```
0,dfegjk,25,47,13;
```

###From secondary to master:
```
identifier/uuid,data,data,data
```
* *identifier/uuid* - If the data coming back is in response to a command, a uuid is passed first - the same passed to execute the command. This notifies serial-synapse that the function has complete, and the follow message is for it. If it is an update handler, just the identifier is used to let *serial-synapse* know that it needs to trigger the update handler function.
* *data, data, data* - *n* data, seperated by commas. No limits to how much or little data can be passed
* *parsing the end* - Please read on serialport parses its "lines". I use "\r\n" to detect a newline as my parser. *synapse-serial* does not care HOW you denote the differentiation between these lines, but it needs to be set via the serialport instance.

# Commands
##Responsive commands
```
synapse.addCommand(opts)
```
WHERE opts =
```
{
	name: "commandName",
	identifier: #,
	returns: [],
	timeout: 1000,
	timeoutFnc: function(){...},
	silent: false
}
```
* **name** - required. Must not start with _, must be unique, and it is compared to a list of reserved names/words. Must follow the same rules as a javascript function as it is going to BE the name of your function.
* **identifier** - required - the enum CmdMessenger will associate to your function on the MCU.
* **returns** - optional - default [] - the order of, and label of, returning data.
* **timeout** - optional, default -1 - if over 0, it will institute a timeout when the function is called. If the timeout is reached before the MCU responds, it will trigger the timeout.
* **timeoutFnc** - optional - The function to trigger if a timeout occurs waiting for a command to fire off. *By default, serial-synapse will call the callback of an execeuted command with the string "Time out occured" if timeoutFnc is not specified.**
* **silent** - optional, default false - If a command is to be responsive, this must be set to false (and is by default). See below for "silent" commands. 

This command can then be utilized in your code by going
```
synapse.commandName(argument1, argument2, function(err, [data, rawMsg]){
	//do stuff
});
```
The callback for responsive commands gets the following arguments:

* *err* - An error, if it occured. If the *timeout* is above 0 and *timeoutFnc* is NOT set, this will also be set to "Time out occured" upon a timeout.
* *data* - optional unless you have data coming back - An object with the return data, as specified by returns. For example - if the returns on the command instantiation was set to **["abc", "xyz", "kjh"]**, and we receive a serial message of *"uuid,100,127,255"* then the return object is automagically formatted for us, and would look like
```
{
	abc: 100,
	xyz: 127,
	kjh: 255
}
```
* *rawMSG* - optional, can be ignored - the raw message that triggered this callback function. If you get more data back than returns, it'll be in here.

##Silent Commands
Silent commands are constructed the same way as responsive commands, but don't have a callback. They have no timeout, and there is no data returned from them. Use this when you don't expect a response. To make a silent command, set *silent* to true.

#Update Handlers
Update handlers are functions that are triggered when we need to allow the MCU to alert us to a change/value without being polled for it.

An update handler is created by doing
```
synapse.addUpdateHandler(opts);
```
WHERE opts is
```
{
	name: 'updateHandlerName',
	identifier: #,
	returns: [],
	then: function(){ ... }
}
```
* **name** - required. Must not start with _, must be unique, and it is compared to a list of reserved names/words. Must follow the same rules as a javascript function.
* **identifier** - required - the enum you told the MCU to report when broadcasting. Make sure it does NOT match any of the enums from the commands for simplicity's sake.
* **returns** - optional - default [] - the order of, and label of, returning data.
* **then** - required - a function that is described as follows:
```
then: function([data, rawMsg]){
	//do stuff
}
```
* *data* - optional, defaults to {} - parsed data as per the returns (same as how return and data works for commands).
* *rawMsg* - optional - the raw msg passed that triggered the update handler

Note that you cannot trigger an updateHandler after creation - it can only be triggered by the secondary.


##Example Usage:
In this simple example, we're going to set up CmdMessenger on an Arduino (I'm assuming an Uno) - and placing a button on pin 4. The Uno has an LED already on pin 13. We're going to have node blink the LED every second, reporting back once it's done. We're also going to have a console log of when the button switches states.

On your Arduino:
```
#include <CmdMessenger.h>

#define LED_PIN 13
#define BUTTON_PIN 4

CmdMessenger cmdMessenger = CmdMessenger(Serial);
int lastValue = 0;

enum
{
	kSetLED,
	kButton
};

void setup()
{
	Serial.begin(9600);
	pinMode(LED_PIN, OUTPUT);
	pinMode(BUTTON_PIN, INPUT);
}

void setLED(){
	char* uuid = cmdMessenger.readStringArg();
	int ledStatus = cmdMessenger.readIntArg();
	
	digitalWrite(LED_PIN, ledStatus);
	
	Serial.print(uuid);
	Serial.println(ledStatus);
}

void loop()
{
	cmdMessenger.feedinSerialData();
	
	cmdMessenger.attach(kSetLED, setLED);
	
	//Check the button and if we need to trigger the update handler
	int buttonValue = digitalRead(BUTTON_PIN);
	if(lastValue != buttonValue){
		lastValue = buttonValue;
		Serial.print(kButton);
		Serial.print(",");
		Serial.println(buttonValue);
	}
}
```
On your master:
```
var SerialSynapse = require('serial-synapse');
var SerialPort = require('serialport').SerialPort;
var parsers = require('serialport').parsers;

var arduino = new SerialSynapse();

arduino.addUpdateHandler(
  {
    name: 'button',
    identifier: 1,
    returns: ["buttonState"],
    then: function(data){
      console.log("The button has been " + data.buttonState ? "pressed" : "released");
    }
  }
);

arduino.addCommand(
  {
    name: 'setLED',
    identifier: 0,
    returns: ['ledState']
  }
);

arduino.connection = new SerialPort('COM13',
	{
		baudrate: 9600,
		parser: parsers.readline('\r\n')
	}
);

var ledState = 0;
setInterval(function(){
  arduino.setLED(ledstate, function(data){
    ledState = ledState ? 0 : 1;
    console.log("LED has been set to " + data.ledState);
  });
}, 1000);

```

## A robot example
I've coded up [a quick example](https://github.com/hlfshell/redbot-synapse-example) using a sub $100 robot from Sparkfun using both this and [here](https://github.com/hlfshell/serial-synapse-socket)


# Socket Connection
As a work in progress, I am also developing a socket wrapper which will immediately expose your synapse commands and update handlers to a socket connection. AKA 3 extra lines of code will bring your device online! It can be seen [here](https://github.com/hlfshell/serial-synapse-socket). Here is an example on how easy it is to expose a synapse object to the internet:
```
//Assuming you've already created a serial-synapse object called synapse
var SynapseServer = require('serial-synapse-socket');
var server = new SynapseServer({
	port: 8080,
	synapse: synapse
});
```
And that's it!

# TODO:
1 - Tests should be written to ensure that this module works as required

2 - Better documentation

3- Examples
