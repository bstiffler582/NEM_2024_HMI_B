## HMI B

### Contents
1. [Intro | Setup](#setup)
2. [WTF is that?](#wtf)
3. [WTF is that?](#wtf)
7. [Server Requests](#server_requests)

<a id="setup"></a>

### 1. Intro | Setup

This course will be mostly hands-on. We will explore some interesting components of the TcHmi framework, some tips and tricks to make life easier, and explore third-party tools that can enhance the developer experience.

Open up a fresh empty TwinCAT HMI project. No need to run through the Project Generator.

<a id="wtf"></a>

### 2. WTF is that?

What does that do? TwinCAT HMI has a lot of interesting and seldom-used features that may be hard to find or understand. Let's dig into a few of them just in case they become useful for your future applications.

---
#### Context object
    
Both inline JavaScript Actions *and* Functions offer the option to "Inject context object". What does this mean? Infosys says the context object (`ctx`) is used for asynchronous scripts, and "can be used to provide feedback about the end of an action."

It offers the following example usage:
```js
// successful execution
ctx.success("Result");

// unsuccessful execution
ctx.error(TcHmi.Errors.ERROR);
```

This looks and sounds strikingly similar to the JavaScript `Promise` object. It is TcHmi's approach to handling asynchronous operations within the framework.

#### Exercise: Context object in an action

- On the default view, drop in a button and a textbox control
- Bind the textbox's Text property to a new Internal symbol of type `Number`
- Double-click the button to edit the click action
    - Add an empty JavaScript action, mark it as 'Asynchronous'
    - Add a *Write to symbol* action to increment the internal symbol

Test your button in Live view. Nothing happens, because our asynchronous JavaScript action never signals completion.

Add the following code to the JS action:
```js
// wait 1 second then signal completion with ctx.success
setTimeout(() => { ctx.success() }, 1000);
```
Now we will see the *Write to symbol* action execute after our JS action finishes and reports successful execution (after 1000ms).

>But wait, there's more. If we don't want our *Write to symbol* action to rely on the previous action at all (synchronous or asynchronous), we can simply right-click the JavaScript action and uncheck the default "*Wait for completion*" flag. You will also see options for "*Show success/error actions*", which allow us to branch based on whether the action results in `ctx.success()` or `ctx.error()`.

This same `ctx` object principal applies to Functions. If we create a new function and modify *Wait Mode* to be asynchronous, the `ctx` object is automatically passed in as a parameter. While we're here in a function definition, let's explore **Rest** parameters.

---
#### Rest Parameters

Per the Mozilla JavaScript developer docs: 
>"The rest parameter syntax allows a function to accept an indefinite number of arguments as an array."

#### Exercise: Rest parameters

- Create a new function (if you haven't already).
- Make the default `par1` parameter a **Rest** parameter by checking the box
- In the function body, just add `console.log(par1)`.
- Call your function from an action (create a new button)
    - Notice that you can add as many parameters as you want
    - Pass in a few parameters, call the function and check the console

If you used a TypeScript function, you should have good results. If you created a JavaScript function, there may be more work to do. Rest parameters rely on the spread operator (`...`), which is not automatically applied to the argument.

---
#### Buried Configs

Right-click on **Server** under the project name, and select both "Toggle Advanced" and "Toggle Advanced Settings". You will notice some additional extensions you can configure, as well as additional options within each extension. Some common ones:
- TcHmiUserManagement -> Password aging, complexity
- ADS -> Limit ADS response size; for `Sum ADS read request failed` errors
- TcHmiSrv -> Advanced -> Symbol complexity limit

---
#### Symbol escaping

---
#### Server symbols

Internal symbols are a great feature for storing data of any type or size in memory, but they are client-scoped and cannot be used to share information at the application level or between clients. An easy workaround for this might be to use the PLC; this is shared memory space with read/write access from any client. However, data that is unrelated to the PLC logic risks being removed by future maintainers that do not know the full context.

Luckily, we can create and use server symbols just like we do with internal symbols. The only difference is that they will live in the server's memory and be accessible from all clients.

In the HMI configuration page, right-click "Mapped Symbols" and "create new server symbol". You can select any type, default value, and even mark the symbol as persistent. Upon creation, you can view or edit your symbol in the `TcHmiSrv` domain.

<a id="server_requests"></a>

### 7. Server Requests

There is some pretty comprehensive server documentation available for TwinCAT HMI. This was previously (1.12) installed with the engineering installation in the `C:\TwinCAT\Functions` directory, but it now (1.14) must be manually added via `tcpkg`.
- Open PowerShell terminal as administrator
- `tcpkg install TwinCAT.HMI.Server.Documentation`

Navigate here: 
```
C:\Program Files (x86)\Beckhoff\TwinCAT\Functions\TF2000-HMI-Server\Documentation\html
```
and run the `README.html` file.

Almost everything that can be accomplished from the configuration pages can also be programmatically configured via web socket requests. This documentation provides details and examples for each extension.

> Given that these are just basic web socket requests, the functions can be executed totally *outside* the front-end TcHmi framework. A custom client-side web application can leverage the powerful backend features of TwinCAT HMI server (e.g. connection and request pooling, ADS server, historizer, etc.) via a common open interface.
