# What is Flux and Why using it ?

If you're familiar with ReactJS components, and you're developing highly interactive components, you'll run into problem with `method` hell. By `method` hell, i mean the following pseudocode:

    <Button onPress={this.onPress}>My Button</Button>
    
`this.onPress` method had the following pattern:

    onPress: function(){
        // do something with state
        // or do something with server
        // or do something to local storage
    }
    
Imagine you had a lot of that `button`, you will see some drawbacks of this approach:
    - The code is not scale well: If you want to add a new JSON request on many components, you will have to duplicate in each component.
    - The code is hard to read and maintain. Too many codes mean it is hard to follow, debug and test.
    - The component is no longer portable as it was before. When you introduces too many high-coupled code in one component, you will have problem with render it to other component (as child or parent). 
    - This approach violates the Single Of Truth pattern. Remember when you design layered components for a backend ? You have the View layer, you have the Business model layer, then you have the DBAccess layer. That DBAccess layer is the Single place of Truth, when it is the only place for you to access and manipulate your database.
    
6 months before, when Flux pattern was not known and public to the wild, i've faced all problems above. It leaded me to unmaintainable architecture for scaling. I am sure your project is really an awesome project from beginning, and you will want it to be updated from time to time, to deliver more features and more value to your customers. So please `Do not` do this approach from beginning. Use `Flux` instead.

### From React to Flux

I will show you how to get benefit from Flux from an example application: Todo Application. I will make step by step.

- I use ReactJS for mockup
- Step by step transition to add React components
- Step by step add Flux components

![flow](http://i.gyazo.com/9f715680daed0116cbb06a401db26264.png)



Let's open console, and type the following commands:

    mkdir todoapp
    cd todoapp
    npm install -g browserify
    npm install --save reactify flux
    
After running above commands, you `package.json` file will have the following content:

*package.json*

    {
      "name": "todoapp",
      "version": "1.0.0",
      "description": "",
      "main": "index.js",
      "scripts": {
        "test": "echo \"Error: no test specified\" && exit 1",
        "build": "browserify -t reactify main.js > build/main.js"
      },
      "author": "",
      "license": "ISC",
      "dependencies": {
        "react": "^0.13.1",
        "reactify": "^1.1.0"
      }
    }
Create the following file structure:

![structure](http://i.gyazo.com/c88ff6a440c9f102104ad4a476875237.png)

Add the `index.html` to project folder, it's just the simple skeleton for our mockup.

*index.html*

    <!DOCTYPE html>
    <html>
      <head>
        <meta charset="UTF-8">
      </head>
      <body>
        <div id="example"></div>
        <script src="build/main.js"></script>
      </body>
    </html>

In `main.js`, we implement the most basic template for a `Todo` application, which includes an `input` to add `todo item`, a `button` to add, and the `todo item list`.

![todoapp](http://i.gyazo.com/48f3cc94bad454686504013eb8508cb4.png)

*main.js*

    'use strict'
    var React = require('react');
    var TodoList = require('./components/TodoList');
    var TodoApp = React.createClass({
    	getInitialState: function(){
    		return {
    			todos: []
    		}
    	},
    	componentDidMount: function(){
    		var todos = [
    			{id: 1, content: 'todo 1'},
    			{id: 2, content: 'todo 2'},
    			{id: 3, content: 'todo 3'}
    		];
    		this.setState({todos: todos});
    	},
    	render: function(){
    		return (
    			<div>
    				<input type="text" ref="todo" />
    				<button onClick={this.addTodo}>Add</button>
    				<h2>Todos:</h2>
    				<TodoList items={this.state.todos} />
    			</div>
    		)
    	}
    });
    
    React.render(<TodoApp />, document.getElementById('example'));

The `TodoList` component retuns a `ul` html component, which includes an array of `li TodoItem component`:
    
*./components/TodoList.js*

    'use strict'
    var React = require('react');
    var TodoItem =  require('./TodoItem');
    var TodoList = React.createClass({
    	render: function(){
    		var todoItems = this.props.items.map(function(item){
    			return <TodoItem item={item} key={item.id} />
    		})
    		return (
    			<ul>
    				{todoItems}
    			</ul>
    		)
    	}
    });
    
    module.exports = TodoList;

*./components/TodoItem.js*

    'use strict'
    var React = require('react');
    var TodoItem = React.createClass({
    	getInitialState: function(){
    		return {edit: false, content: this.props.item.content}
    	},
    	render: function(){
    		if (this.state.edit === false){
    			return (
    				<li>
    					{this.props.item.content}&nbsp;<button onClick={() => this.setState({edit: true})}>Edit</button>
    				</li>
    			)
    		} else{
    			return (
    				<li>
    					<input type="text" value={this.state.content} onChange={(event) => this.setState({content: event.target.value})} />
    				</li>
    			)
    		}
    		
    	}
    });
    
    module.exports = TodoItem;
    
Run `npm run build`, and we got our ReactJS mockup like this:

![todoapp](http://i.gyazo.com/5aa90244a0f77895f45b4a2c28a55aa2.png)
    
### FLux

If you go to the official Flux website, you could see the following picture:

![flux](https://facebook.github.io/flux/img/flux-simple-f8-diagram-explained-1300w.png)

Let's convert the `ReactJS Todo` to `Flux Todo`:

Create the following Dir file structure:

![structure](http://i.gyazo.com/5d8effc21a0cf39fe7df4466c40965f9.png)

In short, we create a `TodoStore`, a `TodoStoreAction`, and two `Dispatcher`.

    npm install object-assign es6-promise --save

*dispatcher/Dispatcher.js*

    var Promise = require('es6-promise').Promise;
    var assign = require('object-assign');
    
    var _callbacks = [];
    var _promises = [];
    
    var Dispatcher = function() {};
    Dispatcher.prototype = assign({}, Dispatcher.prototype, {
    
      /**
       * Register a Store's callback so that it may be invoked by an action.
       * @param {function} callback The callback to be registered.
       * @return {number} The index of the callback within the _callbacks array.
       */
      register: function(callback) {
        _callbacks.push(callback);
        return _callbacks.length - 1; // index
      },
    
      /**
       * dispatch
       * @param  {object} payload The data from the action.
       */
      dispatch: function(payload) {
        // First create array of promises for callbacks to reference.
        var resolves = [];
        var rejects = [];
        _promises = _callbacks.map(function(_, i) {
          return new Promise(function(resolve, reject) {
            resolves[i] = resolve;
            rejects[i] = reject;
          });
        });
        // Dispatch to callbacks and resolve/reject promises.
        _callbacks.forEach(function(callback, i) {
          // Callback can return an obj, to resolve, or a promise, to chain.
          // See waitFor() for why this might be useful.
          Promise.resolve(callback(payload)).then(function() {
            resolves[i](payload);
          }, function() {
            rejects[i](new Error('Dispatcher callback unsuccessful'));
          });
        });
        _promises = [];
      }
    });
    
    module.exports = Dispatcher;
    
*dispatcher/AppDispatcher.js*

    var Dispatcher = require('./Dispatcher');
    var assign = require('object-assign');
    
    var AppDispatcher = assign({}, Dispatcher.prototype, {	
      handleViewAction: function(action){	
    	  this.dispatch({
    	    source: 'VIEW_ACTION',
    	    action: action
    	  });
      }
    });
    
    module.exports = AppDispatcher;
    
*actions/TodoStoreActions.js*

    var AppDispatcher = require('../dispatcher/AppDispatcher');
    var TodoStoreActions = {
    
      loadTodos: function() {
      	var todos = [
      		{id: 1, content: 'todo 1'},
      		{id: 2, content: 'todo 2'},
      		{id: 3, content: 'todo 3'}
    	   ];
    	AppDispatcher.handleViewAction({
    		actionType: "LOAD_TODOS",
    		data: {todos: todos}
    	})
      }
    
    };
    
    module.exports = TodoStoreActions;
    
*stores/TodoStore.js*

    'use strict'
    var AppDispatcher = require('../dispatcher/AppDispatcher');
    var EventEmitter = require('events').EventEmitter;
    var assign = require('object-assign');
    
    // Internal object of todos
    var _todos = [];
    
    // Method to load todos from action data
    function loadTodos(data) {
      _todos = data.todos;
    }
    
    // Merge our store with Node's Event Emitter
    var TodoStore = assign({}, EventEmitter.prototype, {
    
      // Returns all todos
      getTodos: function() {
        return _todos;
      },
    
      emitChange: function() {
        this.emit('change');
      },
    
      addChangeListener: function(callback) {
        this.on('change', callback);
      },
    
      removeChangeListener: function(callback) {
        this.removeListener('change', callback);
      }
    
    });
    
    // Register dispatcher callback
    AppDispatcher.register(function(payload) {
      var action = payload.action;
      // Define what to do for certain actions
      switch(action.actionType) {
        case "LOAD_TODOS":
          // Call internal method based upon dispatched action
          loadTodos(action.data);
          break;
    
        default:
          return true;
      }
      
      // If action was acted upon, emit change event
      TodoStore.emitChange();
    
      return true;
    
    });
    
    module.exports = TodoStore;
    
*main.js*

    'use strict'
    var React = require('react');
    var TodoList = require('./components/TodoList');
    var TodoStoreActions = require('./actions/TodoStoreActions');
    var TodoStore = require('./stores/TodoStore')
    // Method to retrieve application state from store
    function getAppState() {
        return {
    	   todos: TodoStore.getTodos()
    	};
    }
    var TodoApp = React.createClass({
    	getInitialState: function(){
    		TodoStoreActions.loadTodos();
    		return getAppState();
    	},
    	componentDidMount: function(){
    		TodoStore.addChangeListener(this._onChange);
    	},
    	// Unbind change listener
    	componentWillUnmount: function() {
    	  TodoStore.removeChangeListener(this._onChange);
    	},
    	addTodo: function(){
    
    	},
    	render: function(){
    		return (
    			<div>
    				<input type="text" ref="todo" />
    				<button onClick={this.addTodo}>Add</button>
    				<h2>Todos:</h2>
    				<TodoList items={this.state.todos} />
    			</div>
    		)
    	},
    	_onChange: function() {
    	  this.setState(getAppState());
    	}
    });
    
    React.render(<TodoApp />, document.getElementById('example'));
    
Such a lengthy page of code!    
I want to put all the code here for you to `copy` and `paste` to really see what it looks like before really understand it.
Let me explain:

- In funciton getInitialState of TodoApp, you see the following code:


    getInitialState: function(){
		TodoStoreActions.loadTodos();
		return getAppState();
	},
	
You understand that we always use `Action` to do `command` or `query` that originates from the `View`.
The helper `getAppState` method is an exception here, you use it to directly get the `in-memory` data from `TodoStore`. Because `TodoStore` is the only place in your application that mutates data, it keeps the current snapshot of your application state, so everytime the `TodoStore` `emit changes`, the `View` knows it to render everything:
    
    componentDidMount: function(){
		TodoStore.addChangeListener(this._onChange);
	},
	// Unbind change listener
	componentWillUnmount: function() {
	  TodoStore.removeChangeListener(this._onChange);
	},
	_onChange: function() {
	  this.setState(getAppState());
	}
	
Whenever the `TodoStore` emit changes, it calls the registered callback in the `View`, in this case it's the `_onChange` method. The `TodoStore` inherits from `EventEmitter` so it can allow other `object` to listen to its events.

- Now let me explain the Dispatcher side.

After the `View` call the `TodoStoreActions` method, in turn the `TodoStoreActions` will call `dispatch`. `Dispatcher` will route the `action method` to call the corresponding `store method`:

After a call in `TodoStoreActions`:

    AppDispatcher.handleViewAction({
  		actionType: "LOAD_TODOS",
  		data: {todos: todos}
  	})
  	
The `AppDispatcher` will automatically calls the registered callback:  	

    AppDispatcher.register(function(payload) {
      var action = payload.action;
      var text;
      // Define what to do for certain actions
      switch(action.actionType) {
        case "LOAD_TODOS":
          // Call internal method based upon dispatched action
          loadTodos(action.data);
          break;
    
        default:
          return true;
      }
      // If action was acted upon, emit change event
      TodoStore.emitChange();
      return true;
    });

After the `TodoStore` `emit changes`, it's the `View` turn to render everything again.

So we can tell shortly that:


#### React View ==[originate]==> Action ==[dispatch]==> Dispatcher ==[call registered callback]==> Store ==[emit change]==> React View [render the state again]

This finishes the introduction to Flux Application Architecture.
In the next chapter, we will convert this mockup HTML/Javascript code to React Native code.

#### Excercise:
- Please complete the AddTodo method, that is, after clicking the Add Todo button, the new todo item will append to the TodoList.
- Do the above function in both ReactJS and Flux versions.
- What do you prefer to use? ReactJS or Flux?


