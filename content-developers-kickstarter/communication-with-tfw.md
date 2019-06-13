# Communication with TFW

## Named pipes
The communication beetween an application and the TFW server now happens via **named pipes**.
> In computing, a named pipe (also known as a FIFO for its behavior) is an extension to the traditional pipe concept on Unix and Unix-like systems, and is one of the methods of inter-process communication (IPC) (Wikipedia)

Pipes can be read and written (only). Once something written to a pipe as **data stream**, that can be read only once. 


## Pipe event handlers

Below you can find some examples for pipe event handlers with the files they are using by default and with several inputs you can use to trigger TFW-related actions.

#### SignMessagePipeIOEventHandler

(Used files: tfw_sign_recv, tfw_sign_send.)
Signs a valid TFW message with HMAC. Note that the running process needs root permissions in order to read the authentication key. When forwarding is true, it will send the signed message to the TFW server before writing it into the output pipe.
	
#### VerifyMessagePipeIOEventHandler

(Used files: tfw_verify_recv, tfw_verify_send.)
Verifies a signed TFW message. This pipe also needs root permissions. Send the serialized JSON object to the pipe, then wait for its boolean response.

#### BotPipeIOEventHandler

(Used files: tfw_bot_recv, tfw_bot_send.)
Sends bot messages to the frontend. If you assign @originator, it will be the default message sender. When you write a line to the pipe, it will be considered as a single message and gets appended to the queue until an empty line is received, which triggers forwarding the messages to the TFW server.

#### DeployPipeIOEventHandler

(Used files: tfw_deploy_recv, tfw_deploy_send.)
Manages deployment in the IDE. When you receive "deploy", then you have to answer with a "true" or "false" depending whether you are satisfied with the result or not. The `@process` parameter is the name of the supervised service.
	
#### IdePipeIOEventHandler

(Used files: tfw_ide_recv, tfw_ide_send.)
Manipulates the content of the IDE. You can observe a file, and when the user edits it, you will receive the new contents where newlines are escaped as "\\n". In order to overwrite the file, send an escaped text back to the pipe. Since the pipe doesn't know if the file is selected initially in the IDE, you have to provide this information by yourself with @selected, but it will track it later on.

#### FSMPipeIOEventHandler

(Used files: tfw_fsm_recv, tfw_fsm_send.)
Handles FSM steps. When the FSM enters the next state, you will receive a line containing its name. To trigger a state change, send the name of the transition to the pipe.

Example inputs: "step_2"; "step_next", etc.

#### PipeIOEventHandler

The following pipe is special: it creates general purpose pipes: `PipeIOEventHandler: tfw_json_recv, tfw_json_send`. If you want to create a custom pipeIO class, you have to inherit that from `PipeIOEventHandler`.
```
json_pipe = PipeIOEventHandler(
	'',
	'/tmp/tfw_json_send',
	'/tmp/tfw_json_recv',
	permissions=0o666
)
```

The first parameter associates the receiving pipe with a key, which is an empty string in this case. It has a special meaning, you can subscribe to every kind of message with this key.
If you wish to filter incoming data, specify a single or more keys in a list, eg.: processmanager, ide, key...
You can send/receive JSON messages to/from the TFW server as any user, because we gave read+write permissions, without that parameter, only the owner has access to the pipes. 

#### Custom PipeIOEventHandler

To see how you can define a custom `PipeIOEventHandler`, check `pipe_io_auxlib.py` here: https://github.com/avatao-content/test-tutorial-framework/blob/master/solvable/src/pipe_io_auxlib.py
As you can see, two methods should be declared: 
	* `handle_event(self, message)`: this method is invoked when a message comes that you are subcribed to it's key
	* `handle_pipe_event(self, message_bytes)`

## A few words about safety and using pipes

Using pipes will be the safest when the webservice is running by a user whose group has the only privilege (along with `root`) to write the named pipes. Thus, in the Dockerfile, use `chown` command for these pipes that are in the `/tmp` directory to define `0o660` permission to them. So, please create a user only for running the webservice. You can set the this user in the `/solvable/supervisord/webservice.conf` file. This protective measure is because the webservice is usually vulnerable and think about what can a creative user can do in the Docker image when the webservice running as root.
