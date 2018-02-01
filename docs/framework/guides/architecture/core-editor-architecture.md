---
category: framework-architecture
order: 20
---

# Core editor architecture

The [`@ckeditor/ckeditor5-core`](https://www.npmjs.com/package/@ckeditor/ckeditor5-core) package is relatively simple. It comes with just a handful of classes. The ones you need to know are presented below.

## Editor classes

The {@link module:core/editor/editor~Editor} class representing the base of the editor.

The editor is a root object, gluing all other components. It holds a couple of properties that you need to know:

* {@link module:core/editor/editor~Editor#config} &ndash; The configuration object.
* {@link module:core/editor/editor~Editor#plugins} and {@link module:core/editor/editor~Editor#commands} &ndash; The collection of loaded plugins and commands.
* {@link module:core/editor/editor~Editor#document} &ndash; The document. It is the editing engine's entry point.
* {@link module:core/editor/editor~Editor#data} &ndash; The data controller. It controls how data is retrieved from the document and set inside it.
* {@link module:core/editor/editor~Editor#editing} &ndash; The editing controller. It controls {@link module:engine/controller/editingcontroller~EditingController#document document} rendering, including selection handling.
* {@link module:core/editor/editor~Editor#keystrokes} &ndash; The keystroke handler. It allows to bind keystrokes to actions.

Besides that, the editor exposes a few of methods:

* {@link module:core/editor/editor~Editor.create `create()`} &ndash; The static `create()` method. Editor constructors are protected and you should create editors using this static method. It allows the initialization process to be asynchronous.
* {@link module:core/editor/editor~Editor#destroy `destroy()`} &ndash; Destroys the editor.
* {@link module:core/editor/editor~Editor#execute `execute()`} &ndash; Executes the given command.

You can also extend the editor interface using API interfaces:

* {@link module:core/editor/utils/elementapimixin~ElementApi} &ndash; A way to retrieve and set data from/to element on which the editor has been initialized.
* {@link module:core/editor/utils/dataapimixin~DataApi} &ndash; A way to retrieve data from the editor and set data in the editor. The data format is controlled by the {@link module:engine/controller/datacontroller~DataController#processor data controller's data processor} and it does not need to be a string (it can be e.g. JSON if you implement such a {@link module:engine/dataprocessor/dataprocessor~DataProcessor data processor}). See, for example, how to {@link features/markdown produce Markdown output}.

The editor class is a base to implement your own editors. CKEditor 5 Framework comes with a few editor types (for example, {@link module:editor-classic/classiceditor~ClassicEditor classic}, {@link module:editor-inline/inlineeditor~InlineEditor inline} and {@link module:editor-balloon/ballooneditor~BalloonEditor balloon}) but you can freely implement editors which work and look completely different. The only requirement is that you implement the {@link module:core/editor/editor~Editor} interface.

## Plugins

Plugins are a way to introduce editor features. In CKEditor 5 even {@link module:typing/typing~Typing typing} is a plugin. What is more, the {@link module:typing/typing~Typing} plugin requires {@link module:typing/input~Input} and {@link module:typing/delete~Delete} plugins which are responsible for handling, methods of inserting text and deleting content, respectively. At the same time, a couple of other plugins need to customize <kbd>Backspace</kbd> behavior in certain cases, which is handled by themselves. This leaves the base plugins free of any non-generic knowledge.

Another important aspect of how existing CKEditor 5 plugins are implemented is the split into engine and UI parts. For example, the {@link module:basic-styles/boldengine~BoldEngine} plugin introduces schema definition, mechanisms rendering `<strong>` tags, commands to apply and remove bold from text, while the {@link module:basic-styles/bold~Bold} plugin adds the UI of the feature (i.e. a button). This feature split is meant to allow for greater reuse (one can take the engine part and implement their own UI for a feature) as well as for running CKEditor 5 on the server side. At the same time, the feature split [is not perfect yet and will be improved](https://github.com/ckeditor/ckeditor5/issues/488).

The tl;dr of this is that:

* Every feature is implemented or at least enabled by a plugin.
* Plugins are highly granular.
* Plugins know everything about the editor.
* Plugins should know as little about other plugins as possible.

These are the rules based on which the official plugins were implemented. When implementing your own plugins, if you do not plan to publish them, you can reduce this list to the first point.

After this lengthy introduction (which is aimed at making it easier for you to digest the existing plugins), the plugin API can be explained.

All plugins need to implement the {@link module:core/plugin~PluginInterface}. The easiest way to do so is by inheriting from the {@link module:core/plugin~Plugin} class. The plugin initialization code should be located in the {@link module:core/plugin~PluginInterface#init `init()`} method (which can return a promise). If some piece of code needs to be executed after other plugins are initialized, you can put it in the {@link module:core/plugin~PluginInterface#afterInit `afterInit()`} method. The dependencies between plugins are implemented using the static {@link module:core/plugin~PluginInterface.requires} property.

```js
import MyDependency from 'some/other/plugin';

class MyPlugin extends Plugin {
	static get requires() {
		return [ MyDependency ];
	}

	init() {
		// Initialize your plugin here.

		this.editor; // The editor instance which loaded this plugin.
	}
}
```

You can see how to implement a simple plugin in the {@link framework/guides/quick-start Quick start} guide.

## Commands

A command is a combination of an action (a callback) and a state (a set of properties). For instance, the `bold` command applies or removes bold attribute from the selected text. If the text in which the selection is placed has bold applied already, the value of the command is `true`, `false` otherwise. If the `bold` command can be executed on the current selection, it is enabled. If not (because, for example, bold is not allowed in this place), it is disabled.

All commands need to inherit from the {@link module:core/command~Command} class. Commands need to be added to the editor's {@link module:core/editor/editor~Editor#commands command collection} so they can be executed by using the {@link module:core/editor/editor~Editor#execute `Editor#execute()`} method.

Take this example:

```js
class MyCommand extends Command {
	execute( message ) {
		console.log( message );
	}
}

class MyPlugin extends Plugin {
	init() {
		const editor = this.editor;

		editor.commands.add( 'myCommand', new MyCommand( editor ) );
	}
}
```

Calling `editor.execute( 'myCommand', 'Foo!' )` will log `Foo!` to the console.

To see how state management of a typical command like `bold` is implemented, have a look at some pieces of the {@link module:basic-styles/attributecommand~AttributeCommand} class on which `bold` is based.

First thing to notice is the {@link module:core/command~Command#refresh `refresh()`} method:

```js
refresh() {
	const doc = this.editor.document;

	this.value = doc.selection.hasAttribute( this.attributeKey );
	this.isEnabled = doc.schema.checkAttributeInSelection(
		doc.selection, this.attributeKey
	);
}
```

This method is automatically called (by the command itself) when {@link module:engine/model/document~Document#event:changesDone any changes are applied to the model}. This means that the command automatically refreshes its own state when anything changes in the editor.

The important thing about commands is that every change in their state as well as calling the `execute()` method fires an event (e.g. {@link module:core/command~Command#event:change:{attribute} `change:value`} or {@link module:core/command~Command#event:execute `execute`}).

These events make it possible to control the command from the outside. For instance, if you want to disable specific commands when some condition is true (for example, according to your application's logic, they should be temporarily disabled) and there is no other, cleaner way to do that, you can block the command manually:

```js
const command = editor.commands.get( 'someCommand' );

command.on( 'change:isEnabled', forceDisable, { priority: 'lowest' } );
command.isEnabled = false;

function forceDisabled() {
	this.isEnabled = false;
}
```

The command will now be disabled as long as you will not {@link module:utils/emittermixin~EmitterMixin#off off} this listener, regardless of how many times `someCommand.refresh()` is called.

## Event system and observables

CKEditor 5 has an event-based architecture so you can find {@link module:utils/emittermixin~EmitterMixin} and {@link module:utils/observablemixin~ObservableMixin} mixed all over the place. Both mechanisms allow for decoupling the code and make it extensible.

Most of the classes which were already mentioned are either emitters or observables (observable is an emitter too). Emitter can emit (fire events) as well as listen to them.

```js
class MyPlugin extends Plugin {
	init() {
		// Make MyPlugin listen to someCommand#execute.
		this.listenTo( someCommand, 'execute', () => {
			console.log( 'someCommand was executed' );
		} );

		// Make MyPlugin listen to someOtherCommand#execute and block it.
		// You listen with high priority to block the event before
		// someOtherCommand's execute() method is called.
		this.listenTo( someOtherCommand, 'execute', ( evt ) => {
			evt.stop();
		}, { priority: 'high' } );
	}

	// Inherited from Plugin:
	destroy() {
		// Removes all listeners added with this.listenTo();
		this.stopListening();
	}
}
```

The second listener to `'execute'` shows one of the very common practices in CKEditor 5 code. Basically, the default action of `'execute'` (which is calling the `execute()` method) is registered as a listener to that event with a default priority. Thanks to that, by listening to the event using `'low'` or `'high'` priorities you can execute some code before or after `execute()` is really called. If you stop the event, then the `execute()` method will not be called at all. In this particular case, the {@link module:core/command~Command#execute `Command#execute()`} method was decorated with the event using the {@link module:utils/observablemixin~ObservableMixin#decorate `ObservableMixin#decorate()`} function:

```js
import ObservableMixin from '@ckeditor/ckeditor5-utils/src/observablemixin';
import mix from '@ckeditor/ckeditor5-utils/src/mix';

class Command {
	constructor() {
		this.decorate( 'execute' );
	}

	// Will now fire the #execute event automatically.
	execute() {}
}

// Mix ObservableMixin into Command.
mix( Command, ObservableMixin );
```

Besides decorating methods with events, observables allow to observe their chosen properties. For instance, the `Command` class makes its `#value` and `#isEnabled` observable by calling {@link module:utils/observablemixin~ObservableMixin#set `set()`}:

```js
class Command {
	constructor() {
		this.set( 'value', undefined );
		this.set( 'isEnabled', undefined );
	}
}

mix( Command, ObservableMixin );

const command = new Command();

command.on( 'change:value', ( evt, propertyName, newValue, oldValue ) => {
	console.log(
		`${ propertyName } has changed from ${ oldValue } to ${ newValue }`
	);
} )

command.value = true; // -> 'value has changed from undefined to true'
```

<info-box>
	Observable properties are marked in API documentation strings with the `@observable` keyword but we do not mark them in {@link api/index API documentation} ([yet](https://github.com/ckeditor/ckeditor5-dev/issues/285)).
</info-box>

Observables have one more feature which is widely used by the editor (especially in the UI library) &mdash; the ability to bind the value of one object's property to the value of some other property or properties (of one or more objects). This, of course, can also be processed by callbacks.

Assuming that `target` and `source` are observables and that used properties are observable:

```js
target.bind( 'foo' ).to( source );

source.foo = 1;
target.foo; // -> 1

// Or:
target.bind( 'foo' ).to( source, 'bar' );

source.bar = 1;
target.foo; // -> 1
```

You can find more about bindings in the {@link framework/guides/architecture/ui-library UI library architecture} guide.

## Read next

Once you learnt how to create plugins and commands you can read how to implement real editing features in the {@link framework/guides/architecture/editing-engine Editing engine} guide.