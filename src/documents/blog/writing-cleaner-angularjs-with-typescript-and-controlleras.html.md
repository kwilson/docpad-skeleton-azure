---
title: Writing Cleaner AngularJS with TypeScript and ControllerAs
date: 2015-01-07 10:00
type: blog
layout: post
tags: ['AngularJS','Code', 'Jasmine', 'JavaScript', 'TypeScript', 'Web Development']
---

Our team made the move to TypeScript and Angular at the tail end of last year. I'd had a look at Angular a year or so ago but struggled to get my head around the excessive usage of **$scope** and the nesting of **$parent** items that needed to be traversed.

Since then (with version 1.2.0), Angular now supports [Controller As](https://docs.angularjs.org/api/ng/directive/ngController) which lets us define properties and methods on the controller as a class rather than in a shared scope object.

It's easier to see with an example. Here's a controller for a to-do list using **$scope**:

<pre class="brush: javascript">
function toDoListController($scope) {
    $scope.name = "My List";
    $scope.listItems = [];
    $scope.newItemName = "";

    $scope.save = function() { }
    $scope.toggle = function(listItem) { }
}
</pre>

This could then be used with a view such as:

<pre class="brush: html">
<div ng-controller="toDoListController">
    <h2>{{ name }}</h2>

    <form ng-submit="save()">
        <input type="text" 
               ng-model="newItemName" 
               placeholder="Task name"/>
        <button type="submit">Add Item</button>
    </form>

    <ul>
        <li ng-repeat="item in listItems" 
            ng-class="{ complete: item.isComplete }">
            <label>
                <input type="checkbox" 
                       ng-click="toggle(item)"
                       ng-checked="item.isComplete" />
                <p>{{ item.name }}</p>
            </label>
        </li>
    </ul>
</div>
</pre>

There's nothing badly wrong with this. **$scope** only really starts to become a problem when you start nesting them.

<pre class="brush: html">
<div ng-controller="myController">
	This name: {{ name }}

	<div ng-controller="childController">
		This name:   {{ name }}
		Parent name: {{ $parent.name }}

		<div ng-controller="anotherChildController">
			This name:        {{ name }}
			Parent name:      {{ $parent.name }}
			Grandparent name: {{ $parent.$parent.name }}
		</div>
	</div>
</div>
</pre>

The above is a very contrived example but you can see where the scope hierarchies start to become problematic.

This is where Controller As comes in.

##Controller As
The [Controller As](https://docs.angularjs.org/api/ng/directive/ngController) syntax allows us to use the controller as an instance and bind to and interract with properties and methods directly on it.

Using our example from above, our to-do list controller can drop all references to **$scope** and would look something like this:

<pre class="brush: javascript">
function ToDoListController() {
    this.name = "My List";
    this.listItems = [];
    this.newItemName = "";
}

ToDoListController.prototype.save = function() {};
ToDoListController.prototype.toggle = function(listItem) {};
</pre>

Note the change (following convention) from *toDoListController* to *ToDoListController* as we are now treating the controller as a class.

Our view can now be changed to this:

<pre class="brush: html">
<div ng-controller="toDoListController as ctrl">
    <h2>{{ ctrl.name }}</h2>

    <form ng-submit="ctrl.save()">
        <input type="text" 
               ng-model="ctrl.newItemName" 
               placeholder="Task name"/>
        <button type="submit">Add Item</button>
    </form>

    <ul>
        <li ng-repeat="item in ctrl.listItems" 
            ng-class="{ complete: item.isComplete }">
            <label>
                <input type="checkbox" 
                       ng-click="ctrl.toggle(item)"
                       ng-checked="item.isComplete" />
                <p>{{ item.name }}</p>
            </label>
        </li>
    </ul>
</div>
</pre>

This is a little more verbose than using **$scope** everywhere but is more explicit with references. Have a look at our contrived example to see just how much more readible it becomes:

<pre class="brush: html">
<div ng-controller="myController as ctrl">
	This name: {{ ctrl.name }}

	<div ng-controller="childController as childCtrl">
		This name:   {{ childCtrl.name }}
		Parent name: {{ ctrl.name }}

		<div ng-controller="anotherChildController as anotherChildCtrl">
			This name:        {{ anotherChildCtrl.name }}
			Parent name:      {{ childCtrl.name }}
			Grandparent name: {{ ctrl.name }}
		</div>
	</div>
</div>
</pre>

That's the first piece of our structure.

## TypeScript
As written on the [TypeScript site](http://www.typescriptlang.org/):
> TypeScript lets you write JavaScript the way you really want to.
> TypeScript is a typed superset of JavaScript that compiles to plain JavaScript.
> Any browser. Any host. Any OS. Open Source.

One of the main benfits from using TypeScipt is that it allows you to use ES6 features such as [Classes](http://www.typescriptlang.org/Handbook#classes) right now.

### EC5 'Class'
<pre class="brush: javascript">
function Greeter(message) {
    this.message = message;
}

Greeter.prototype.greet = function () {
    console.log("Hello, " + this.message);
};
</pre>

### TypeScript Class
<pre class="brush: javascript">
class Greeter {
    constructor(public message: string) { }

    greet() {
        console.log("Hello, " + this.message);
    }
}
</pre>

As we saw above, using **Controller As** allows us to access the Controller as an instance. With TypeScript, we can make that instance an explicit class.

<pre class="brush: javascript">
class ToDoListController {
    name: string;
    listItems: any[];

    newItemName: string;

    constructor() {
        this.name = "";
        this.listItems = [];
    }

    save() {
    }

    toggle(listItem: ListItem){
    }
}
</pre>

Now we have an overview of how this works, let's build out a simple To-Do app and see how everything fits together.

## The To-Do Angular/TypeScript App
To start with, we're going to need to set out a file structure. The *scripts* folder in a Visual Studio solution quickly becomes a no-man's-land of libraries from NuGet, so we'll stay out of there. Under the root, we'll create an *app* folder and our files will live under there:

<pre>
app
-- controllers/
-- directives/
-- models/
-- views/
-- main.ts
</pre>

First of all we'll need a model for a To-Do item to appear in the list, so we'll create that under the ./models folder. TypeScript also gives us support for module definition (very similar to namespacing in the .NET world) so we'll use that to keep from polluting the global namespace.

Modules also make registering components with Angular a lot simpler, but we'll cover that later.

### models/list-item.ts
<pre class="brush: javascript">
module TypeScriptAndAngular {
    export class ListItem {

        constructor(
            public name: string,
            public isComplete: boolean = false) {
        }
    }
} 
</pre>

Here we're passing the task name through the constructor and also setting the *isComplete* flag (which defaults to *false*).

We could manage our to-do lists directly on the page using controller instances but, to make them more easily re-usable, let's wrap them up as an [Angular Directive](https://docs.angularjs.org/api/ng/directive).

For our to-do list, we want to be able to pass in a name for the list as we create it on the page. We'll wrap this up in an interface matching the scope definition (line 10) then use that with the directive's controller.

Each directive can have its own controller instance and its own *isolate scope*. The *isolate scope* (as it sounds) keeps everything in the directive scope isolated from the main app scope. Although we're not going to be using the **$scope** directly, we still need to use the directive's scope binding to pull in data from its usage on the page (passing in the list name as an HTML attribute).

### directives/to-do-list.ts
<pre class="brush: javascript; highlight: [10,12,13,14]">
module TypeScriptAndAngular.Directives {
    export interface IToDoListScope {
        name: string
    }

    export function toDoList(): ng.IDirective {
         return {
            restrict: "E",
            scope: {
                name: "@"
            },
            controller: Controllers.ToDoListController,
            controllerAs: "vm",
            templateUrl: "./app/views/to-do-list.html",
            replace: true
        }
     }
}
</pre>

In line 13 we use the *Controller As* syntax to give our controller instance an alias for use with the template defined on line 14.

Our controller (referenced on line 12) is very similar to the example code earlier:

### controllers/to-do-list-controller.ts
<pre class="brush: javascript; highlight: [9,12]">
module TypeScriptAndAngular.Controllers {

    export class ToDoListController {
        name: string;
        listItems: ListItem[];

        newItemName: string;

        static $inject = [
            "$scope"
        ];
        constructor(isolateScope: Directives.IToDoListScope) {
            this.name = isolateScope.name;
            this.listItems = [];
        }

        save() {
            if (this.newItemName && this.newItemName.length > 0) {
                var newItem = new ListItem(this.newItemName);
                this.listItems.push(newItem);

                this.newItemName = null;
            }
        }

        toggle(listItem: ListItem): boolean {
            listItem.isComplete = !listItem.isComplete;
            return listItem.isComplete;
        }
    }
}
</pre>

So that the file will still function when compressed, we use the static **$inject** method on line 9 to specify what parameter Angular should inject in the class constructor. This instance (line 12) is the *isolate scope* of the directive.

The last part of our directive is the view. This uses the controller alias from our directive definition and lists the to-do items along with a form for adding new ones:

### views/to-do-list.html
<pre class="brush: html">
<div class="toDoList">
    <h2>{{ vm.name }}</h2>

    <form ng-submit="vm.save()">
        <input type="text" ng-model="vm.newItemName" placeholder="Task name"/>
        <button type="submit">Add Item</button>
    </form>

    <ul>
        <li ng-repeat="item in vm.listItems" ng-class="{ complete: item.isComplete }">
            <label>
                <input type="checkbox" ng-click="vm.toggle(item)" ng-checked="item.isComplete" />
                <p>{{ item.name }}</p>
            </label>
        </li>
    </ul>
</div>
</pre>

All references to our controller are made using the *vm* alias (vm = view model).

We now only need our *main.ts* file and our HTML page to pull everything together:

### main.ts
<pre class="brush: javascript">
module TypeScriptAndAngular {
    angular.module("tsAngularApp", [])
        .controller(TypeScriptAndAngular.Controllers)
        .directive(TypeScriptAndAngular.Directives);
}
</pre>

Since we're using modules, we only need to reference the module name rather than each individual item when registering our Angular components. This can greatly simplify a large app as we can maintain a structure of one class per file while not having to update the app definition every time a new one is added. By using the module in the registration, all items within that module will be registered at once.

### default.html
<pre class="brush: html">
<body ng-app="tsAngularApp">
    <h1>To-Do</h1>
    <to-do-list name="My Test List"></to-do-list>
    <to-do-list name="My Second Test List"></to-do-list>
</body>
</pre>

## Testing
Since we have no dependency on **$scope**, testing of our controller becomes very easy since we no longer need to pull in any references to Angular at all. Our controller is simply a class that is expecting *something* that implements the *IToDoListScope* interface in the constructor. So this becomes easy to mock.

Our full tests for the controller (using [Jasmine](jasmine.github.io)) then look like this:

<pre class="brush: javascript">
module TypeScriptAndAngular.Controllers.Tests {
    describe("ToDoListController Tests", () => {
        var listScopeMock: Directives.IToDoListScope;

        describe("Constructor Tests", () => {

            it("Constructor sets defaults as expected", () => {
                // Arrange
                var name = "A List Name";
                listScopeMock = {
                    name: name
                }

                // Act
                var ctrl = new Controllers.ToDoListController(listScopeMock);

                // Assert
                expect(ctrl.name).toEqual(name);
                expect(ctrl.listItems).toBeDefined();
                expect(ctrl.newItemName).toBeUndefined();
                expect(ctrl.listItems.length).toBe(0);
            });
        });

        describe("Save Tests", () => {
            it("Save does nothing if no task name has been entered", () => {
                // Arrange
                var ctrl = new Controllers.ToDoListController(listScopeMock);

                // Act
                ctrl.save();

                // Assert
                expect(ctrl.listItems.length).toBe(0);
            });

            it("Save does nothing if task name is empty string", () => {
                // Arrange
                var ctrl = new Controllers.ToDoListController(listScopeMock);
                ctrl.newItemName = "";

                // Act
                ctrl.save();

                // Assert
                expect(ctrl.listItems.length).toBe(0);
            });

            it("Save adds a new item with the specified name", () => {
                // Arrange
                var taskName = "A new task";
                var ctrl = new Controllers.ToDoListController(listScopeMock);
                ctrl.newItemName = taskName;

                // Act
                ctrl.save();

                // Assert
                expect(ctrl.listItems.length).toBe(1);
                expect(ctrl.listItems[0].name).toBe(taskName);
            });
        });

        describe("Toggle Tests", () => {
            it("Toggle sets complete to FALSE if it was originally TRUE", () => {
                // Arrange
                var item = new ListItem("A new item", true);
                var ctrl = new Controllers.ToDoListController(listScopeMock);

                // Act
                ctrl.toggle(item);

                // Assert
                expect(item.isComplete).toBe(false);
            });

            it("Toggle sets complete to TRUE if it was originally FALSE", () => {
                // Arrange
                var item = new ListItem("A new item", false);
                var ctrl = new Controllers.ToDoListController(listScopeMock);

                // Act
                ctrl.toggle(item);

                // Assert
                expect(item.isComplete).toBe(true);
            });
        });
    });
}
</pre>

And that's about it. This is still an evolving project so we may discover better ways to structure or organise some of the code here. If you have any suggested improvements, please let me know.

All code used above is [available on GitHub](https://github.com/kwilson/TypeScriptAndAngular).