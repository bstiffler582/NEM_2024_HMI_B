## HMI B

### Contents
1. [Intro | Setup](#setup)
2. [Server Requests](#server_requests)
3. [Framework Control](#fwcontrol)
4. [WTF is that?](#wtf)
5. [Automated Testing](#testing)

<a id="setup"></a>

### 1. Intro | Setup

This course will be mostly hands-on. We will explore some interesting components of the TcHmi framework, some tips and tricks to make life easier, and explore third-party tools that can enhance the developer experience.

Clone this repository, open the solution and add a fresh empty TwinCAT HMI project. No need to run through the Project Generator.

<a id="server_requests"></a>

### 2. Server Requests

There is some pretty comprehensive server documentation available for TwinCAT HMI. This was previously (1.12) installed with the engineering installation in the `C:\TwinCAT\Functions` directory, but it now (1.14) must be manually added via `tcpkg`.
- Open PowerShell terminal as administrator
- `tcpkg install TwinCAT.HMI.Server.Documentation`

Navigate here: 
```
C:\Program Files (x86)\Beckhoff\TwinCAT\Functions\TF2000-HMI-Server\Documentation\html
```
and run the `README.html` file.

Almost everything that can be accomplished from the configuration pages can also be programmatically configured via web socket requests. This documentation provides details and examples for each extension.

For instance, can programmatically map a PLC symbol via this interface with `TcHmiSrv.AddSymbol`. Or even better, we can map *all* the Auto-map symbols in a PLC project with one call to `TcHmiSrv.AddSymbols`. 

> Given that these are just basic web socket requests, the functions can be executed totally *outside* the front-end TcHmi framework. A custom client-side web application can leverage the powerful backend features of TwinCAT HMI server (e.g. connection and request pooling, ADS server, historizer, etc.) via a common, open interface.

#### Exercise: Server requests

Make sure the PLC project is running. It has some symbols with the `{ attribute TcHmiSymbol.AddSymbol }` pragma. Modify the `TcHmiSrv.AddSymbols` sample request to target our development server at `ws://localhost:3000/`, and send it. It will identify and return all the auto-map symbols. They should now show up as Mapped symbols in your HMI project.

Create a new CodeBehind file, and remove the function/namespace wrapper that is auto-generated. Add the following code:
```js
const ws = new WebSocket('ws://localhost:3000/');

ws.onopen = () => {
    const message = {
        commands: [
            {
                commandOptions: [],
                symbol: "ADS.PLC1.MAIN.nAttribute"
            }
        ],
        requestType: "Subscription",
        intervalTime: 1000
    };
    ws.send(JSON.stringify(message));
};

ws.onmessage = (message) => {
    if (message.data) {
        const data = JSON.parse(message.data);
        if (data.commands.length) {
            console.log(data);
        }
    }
};
```
We have mapped a symbol and subscribed to read its value on change, all without using any `TcHmi` calls. This is part of how the framework is interacting with the server behind the scenes. We can use the framework's wrapper functions to the same effect. Going back to the docs, copy the JavaScript sample request from the `TcHmiSrv.GetSchema` page. Then finesse it to point to one of our mapped symbols, and put it in a button action:
```js
TcHmi.Server.writeSymbol('GetSchema',
    {
        "symbol": "ADS.PLC1.MAIN.nAttribute"
    },
    data => {
        if (data.error !== TcHmi.Errors.NONE ||
            data.response.error ||
            data.response.commands[0].error) {
            // Handle error(s)...
            return;
        }
        // Handle result...
        console.log(data.response.commands[0].readValue);
    }
);
```
The `GetSchema` request returns some useful meta data about our symbol, like its type and even custom pragmas we may have applied in the PLC program. Customers can leverage this type of information to automate a lot of their HMI development.

If we open the network tab in our browser, refresh the page and filter for websocket traffic, we can click on the active connection (`127.0.0.1`) and see the websocket messages come in with a similar format to the message we manually created in the previous step. If we create a binding (e.g. TextBox control Text property to a PLC symbol), we can observe the framework creating its own "Subscription" request to poll the value.

<a id="fwcontrol"></a>

### 3. Framework Control

The TwinCAT HMI control architecture allows for near limitless extensibility. Leveraging open web technologies, we can write our own HTML, CSS and JavaScript/TypeScript and seamlessly integrate it with our project. We can even import third party libraries to leverage open source or community driven resources.

#### Exercise: Add a framework control and some third party references

Create a new "TwinCAT HMI Framework" project in your solution. A framework project can contain one or more custom controls. We are going to add a reference to the **BabylonJs** library, which is an open source JavaScript 3D rendering engine built and maintained by Microsoft.

An option for adding this reference would be to use the node package manager:
```ps
npm install @babylonjs/core @babylonjs/loaders
```
**but**, this would give us the full source, un-"minified" and with type definitions. This is especially useful for development and debugging, however the full source takes up a lot of extra space. Since we intend to compile and distribute our framework project (via nuget packages), we will want to keep dependencies to a minimum. We can simply create a 'Lib' folder in our framework project directory and copy in the minified versions of the Babylon dependencies from this repository.

Once the dependency files are added into our Framework project, we will want to make sure to reference them in the control's `Description.json` file:
```json
"dependencyFiles": [
  ...
  {
    "name": "../Lib/babylon.js",
    "type": "JavaScript",
    "description": "BabylonJs"
  },
  {
    "name": "../Lib/babylonjs.loaders.min.js",
    "type": "JavaScript",
    "description": "BabylonJs loaders"
  }
]
```
> If we were creating a TypeScript control, we would add the dependency in the TypeScript config file as well (`tsconfig.tpl.json`)
>```json
>"include": [
>    "$(Beckhoff.TwinCAT.HMI.Framework).InstallPath/TcHmi.d.ts",
>    "./Path/to/dependency/TypeScript_Declaration_File.d.ts"
>]
>```

Now to implement the library:
- Modify the `Template.html` file to include a canvas element with a unique identifier
```html
<canvas class="babylon" style="width: 100%; height: 100%"></canvas>
```

- In the control's JS file, we first need to grab a reference to the `<canvas>` element, then we can initialize the Babylon engine. Put this in the `__previnit()` method, before calling the base:
```js
this.__elementCanvas = this.__elementTemplateRoot.find('.babylon').get(0);
if (this.__elementCanvas) {
    this.__engine = new BABYLON.Engine(this.__elementCanvas, true);
}
```
Let's take a moment to declare some of these fields we are using. In a JS framework control, we do this in the `constructor` method:
```js
constructor(element, pcElement, attrs) {
    /** Call base class constructor */
    super(element, pcElement, attrs);

    // internal fields
    this.__engine = null;
    this.__scene = null;
}
```

- The next step is just some initialization logic to render the scene. We will create a new method to handle this. Put it underneath the `destroy()` method (same nesting level), and then call it from `__init()`.
```js
__createScene() {
    if (this.__engine) {
        // init scene
        const scene = new BABYLON.Scene(this.__engine);

        // camera and lighting
        scene.createDefaultCameraOrLight(true, true, true);

        // axis viewer
        new BABYLON.Debug.AxesViewer(scene, 0.25);

        // render loop
        this.__engine.runRenderLoop(function () {
            scene.render();
        });

        this.__scene = scene;
    }
}
```
```js
__init() {
    this.__createScene();
    super.__init();
}
```

- Build your Framework Control project, then add it as a project reference to the main HMI project. Once refreshed, we should be able to drop an instance of our new control on the view, just like we would any other control. You should have a 3D engine on your page, with visible X/Y/Z axes and typical mouse rotate/zoom/pan controls. You might notice that the rendering looks a little blurry. This is because the engine does not know the size of its parent `<canvas>` element. We can tell the engine to resize once the element has been *attached* to the DOM:
```js
__attach() {
    this.__engine?.resize();
    super.__attach();
}
```
That should look much better.
> **What** goes *where*?? How do we know where to put each piece of logic in the control's template? We have `previnit()`, `init()`, `attach()`, `detach()`... Each method has useful comments above to give you and idea of how the control lifecycle works, and there is also good documentation on [infosys](https://infosys.beckhoff.com/english.php?content=../content/1033/te2000_tc3_hmi_engineering/5004292747.html&id=9218520696118602432).

Now we are ready to start rendering some objects. The naive approach might be to manually import a model or statically define some shapes right in the body of our control. A better, more modular approach would be to have simple properties that allow the end user of our control to define their 3D objects. You might be wondering where to even start with this type of task, but we have a ton of reference material in the existing controls.

- Drop in an EventGrid control and take a look at the **Columns** property. Notice that we can specify multiple elements, each having their own unique property values. In this case, we can also specify different *types* of elements to be processed by the control in the same collection. A similar interface for our custom control would be ideal.

The type of this property is `TcHmi.Controls.Beckhoff.TcHmiEventGrid.ColumnList`. We can look in the EventGrid's Types.Schema.json file in the Schema directory to see the definition: 
`[buildOutput]\BeckhoffTwinCAT.HMI.Controls\TcHmiEventGrid\Schema`
```json
"TcHmi.Controls.Beckhoff.TcHmiEventGrid.ColumnList": {
    "title": "ColumnList",
    "type": "array",
    "items": {
        "anyOf": [
            {
...
```

Our own property will look similar. In Babylon, a 3D object either imported or drawn is typically called a "Mesh", so we will call our collection "MeshList". 
- In our own control's Types.Schema file, paste the following under **definitions**:
```json
"TcHmi.Controls.<ProjectName>.<ControlName>.MeshList": {
  "title": "MeshList",
  "type": "array",
  "items": {
    "anyOf": [
      {
        "title": "Import",
        "type": "object",
        "engineeringColumns": [ "meshName" ],
        "propertiesMeta": [
          {
            "name": "meshName",
            "displayName": "Mesh Name",
            "category": "General",
            "description": "Name of mesh",
            "defaultValue": "Mesh",
            "defaultValueInternal": null
          },
          {
            "name": "path",
            "displayName": "Path",
            "category": "General",
            "description": "Path to mesh file",
            "defaultValue": null,
            "defaultValueInternal": null
          },
          {
            "name": "position",
            "displayName": "Position",
            "category": "General",
            "description": "Mesh position offsets",
            "defaultValue": null,
            "defaultValueInternal": null
          }
        ],
        "properties": {
          "meshName": {
            "type": "string"
          },
          "path": {
            "$ref": "tchmi:framework#/definitions/Path"
          },
          "position": {
            "$ref": "tchmi:framework#/definitions/TcHmi.Controls.<ProjectName>.<ControlName>.MeshPosition"
          }
        },
        "additionalProperties": false,
        "required": [ "meshName", "path" ]
      },
      {
        "title": "Shape",
        "type": "object",
        "engineeringColumns": [ "meshName" ],
        "propertiesMeta": [
          {
            "name": "meshName",
            "displayName": "Mesh Name",
            "category": "General",
            "description": "Name of mesh",
            "defaultValue": "Shape",
            "defaultValueInternal": null
          },
          {
            "name": "shapeType",
            "displayName": "Type",
            "category": "General",
            "description": "Type of shape mesh",
            "defaultValue": "box",
            "defaultValueInternal": null
          },
          {
            "name": "position",
            "displayName": "Position",
            "category": "General",
            "description": "Mesh position offsets",
            "defaultValue": null,
            "defaultValueInternal": null
          }
        ],
        "properties": {
          "meshName": {
            "type": "string"
          },
          "shapeType": {
            "type": "string",
            "enum": [
              "box",
              "sphere",
              "cylinder"
            ]
          },
          "position": {
            "$ref": "tchmi:framework#/definitions/TcHmi.Controls.<ProjectName>.<ControlName>.MeshPosition"
          }
        },
        "additionalProperties": false,
        "required": [ "meshName", "shapeType" ]
      }
    ]
  }
}
```
Notice we are also referencing another type definition for the "position" property. We define that here as well:
```json
"TcHmi.Controls.<ProjectName>.<ControlName>.MeshPosition": {
  "title": "MeshPosition",
  "type": "object",
  "propertiesMeta": [
    {
      "name": "x",
      "displayName": "X",
      "description": "X position offset",
      "defaultValue": 0,
      "defaultValueInternal": 0
    },
    {
      "name": "y",
      "displayName": "Y",
      "description": "Y position offset",
      "defaultValue": 0,
      "defaultValueInternal": 0
    },
    {
      "name": "z",
      "displayName": "Z",
      "description": "Z position offset",
      "defaultValue": 0,
      "defaultValueInternal": 0
    }
  ],
  "properties": {
    "x": {
      "type": "number"
    },
    "y": {
      "type": "number"
    },
    "z": {
      "type": "number"
    }
  }
}
```
This will allow us to offset the position of the mesh rendering. Anything we define in this file will be added to the referencing project's framework definitions. You can view it in the project configuration under "Data Types".

>Imagine how we can also supply rotation and scaling properties to fully customize the behavior of the model. We can also go as far as to make these properties bindable so that meshes can respond to input and animate in real time.

Next, we create a new attribute in our control's `Description.json` file. This is where we specify control properties that can be assigned or bound in the editor.
- Create "Mesh List" property for the control:
```json
"attributes": [
  {
    "name": "data-tchmi-mesh-list",
    "displayName": "Mesh List",
    "type": "tchmi:framework#/definitions/TcHmi.Controls.<ProjectName>.<ControlName>.MeshList",
    "propertyGetterName": "getMeshList",
    "propertySetterName": "setMeshList",
    "propertyName": "meshList",
    "bindable": false,
    "category": "Common",
    "defaultValue": null,
    "defaultValueInternal": null,
    "description": "List of meshes"
  }
]
```
In the control's JS file, we need to add the getter and setter methods specified above. These are the entry points for reading and writing the property value. 
- Add below our `__createScene()` method:
```js
getMeshList() {
    return this.__meshList;
}

setMeshList(value) {
    if (value)
        this.__meshList = value;
}
```
- And finally, in the `constructor`, another field to hold the mesh list data:
```js
this.__meshList = [];
```
Upon a build, we are now able to select our control in the designer and specify a mesh list. We have a handy interface for providing the name, path or shape (based on mesh type), and position offsets.

We are almost ready to render. Included in this repository is a GLB model file we can test with. Extract the `model.zip` file and bring it into the `Imports` folder in the HMI project via "Add -> Existing Item...". Lastly, we just need the Babylon code to import the model and apply the offsets.
- Add the following `renderMeshes()` method (below `__createScene()`):
```js
__renderMeshes(meshes) {

    meshes.forEach(async mesh => {
        let m;
        if (mesh.path) {
            // import
            const imported = await BABYLON.SceneLoader.ImportMeshAsync(null, mesh.path);
            m = imported.meshes[0];
        } else if (mesh.shapeType) {
            // draw shape
            switch (mesh.shapeType) {
                case 'box':
                    m = BABYLON.MeshBuilder.CreateBox(mesh.meshName);
                    break;
                case 'sphere':
                    m = BABYLON.MeshBuilder.CreateSphere(mesh.meshName);
                    break;
                case 'cylinder':
                    m = BABYLON.MeshBuilder.CreateCylinder(mesh.meshName);
                    break;
            }
        }
        // apply position offsets
        if (m && mesh.position) {
            m.position.x = mesh.position.x;
            m.position.y = mesh.position.y;
            m.position.z = mesh.position.z;
        }
    });
}
```
And call it from within `__createScene()`, just above the render loop:
```js
...
// render mesh list
if (this.__meshList.length) {
    this.__renderMeshes(this.__meshList);
}
// render loop
...
```
Test the control by adding meshes to the Mesh List property, including the GLB file and some shapes. Think about how you might further extend the type schema, properties, methods and control to facilitate even more functionality.

<a id="wtf"></a>

### 4. WTF is that?

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

What is the result? Rest parameters rely on the spread operator (`...`) in the function declaration. Currently, the development environment does not automatically append this, so we need to do it manually:
```js
//...
MyFunction(...par1) {
  console.log(par1);
}
```
Now the log should show all the parameter values.

---
#### Buried Configs

Right-click on **Server** under the project name, and select both "Toggle Advanced" and "Toggle Advanced Settings". You will notice some additional extensions you can configure, as well as additional options within each extension. Some common ones:
- ADS
  - Limit ADS response size (for `Sum ADS read request failed` errors)
- TcHmiSrv -> Advanced
  - Symbol complexity limit
  - Maximum number of connections
  - Maximum connections per client
- TcHmiUserManagement
  - Password aging
  - Password complexity

---
#### Server symbols

Internal symbols are a great feature for storing data of any type or size in memory, but they are client-scoped and cannot be used to share information at the application level or between clients. An easy workaround for this might be to use the PLC; this is shared memory space with read/write access from any client. However, data that is unrelated to the PLC logic risks being removed by future maintainers that do not know the full context.

Luckily, we can create and use server symbols just like we do with internal symbols. The only difference is that they will live in the server's memory and be accessible from all clients.

In the HMI configuration page, right-click "Mapped Symbols" and "create new server symbol". You can select any type, default value, and even mark the symbol as persistent. Upon creation, you can view or edit your symbol in the `TcHmiSrv` domain.

<a id="testing"></a>

### 5. Automated Testing

We can leverage automated testing tools like [Jest](https://jestjs.io/) right within the TwinCAT HMI development environment. By default, we do not have access to the `TcHmi` framework from the test environment, so practical implementations would be reserved for business logic and helper functions. 
- Within the Framework project, create a new folder `Modules` and add a new JS file `Helpers.js`. Add the following code to `Helpers`:
```js
class Helpers {
    TestableMethod(a, b) {
        return a + b;
    }
}

// export for testing
try {
    if (process.env.NODE_ENV === 'test') {
        module.exports = Helpers;
    }
} catch (e) {
    // do nothing
}
```
- Now, add a test JS file to the same folder; `Helpers.test.js`:
```js
const Helpers = require('./Helpers');
const helpers = new Helpers();

test("Helper method", () => {
    expect(helpers.TestableMethod(1, 1)).toBe(2);
});
```
- To run the test, we must install Jest via the node package manager. Open a developer console (``` Ctrl + ` ```), make sure we are in the framework control's directory, and run the following command:
```ps
npm install jest
```
> The TwinCAT HMI's TypeScript build process relies on `node`, so any machine with TE2000 will already have `node` / `npm` installed and ready to use.

Adding any package via npm will create a `package.json` file in the installation root. 
- Add the `"scripts"` entry below to `packages.json` to tell npm to forward the `test` command to Jest:
```json
{
  "devDependencies": {
    ...
  },
  "scripts": {
    "test": "jest"
  }
}
```
- Finally, in the same developer console execute `npm test` to run the tests and see the results.

Furthermore, we can open the properties of our framework project and add the `npm test` command as a pre- or post-build event.
> The challenge with implementing popular test frameworks like Jest is that `TcHmi` does not use JavaScript *modules*, and instead relies on classes and prototypal inheritance. We intend to increase support for modules in future versions for TwinCAT HMI, which may expand testing capabilities.
