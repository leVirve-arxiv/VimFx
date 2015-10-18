<!--
This is part of the VimFx documentation.
Copyright Simon Lydell 2015.
See the file README.md for copying conditions.
-->

# Public API

VimFx has a public API. It is intended to be used by:

- Users who prefer to configure things using text files.
- Users who would like to add custom commands.
- Users who would like to set [special options].
- Users who would like to make site-specific customizations.
- Extension authors who would like to extend VimFx.

VimFx users who use the public API should write a so-called [config file].


## Getting the API

```js
let {classes: Cc, interfaces: Ci, utils: Cu} = Components
Cu.import('resource://gre/modules/Services.jsm')
let apiPref = 'extensions.VimFx.api_url'
let apiUrl = Services.prefs.getComplexValue(apiPref, Ci.nsISupportsString).data
Cu.import(apiUrl, {}).getAPI(vimfx => {

  // Do things with the `vimfx` object here.

})
```

You might also want to take a look at the [config file bootstrap.js
example][bootstrap.js].


## API

The following sub-sections assume that you store VimFx’s public API in a
variable called `vimfx`.

### `vimfx.get(pref)` and `vimfx.set(pref, value)`

Gets or sets the value of the VimFx pref `pref`.

You can see all prefs in [defaults.coffee], or by opening [about:config] and
filtering by `extensions.vimfx`. Note that you can also access the [special
options], which may not be accessed in [about:config], using `vimfx.get()` and
`vimfx.set()`—in fact, this is the _only_ way of accessing those options.

#### `vimfx.get(pref)`

Gets the value of the VimFx pref `pref`.

```js
// Get the value of the Hint chars option:
vimfx.get('hint_chars')
// Get all keyboard shortcuts (as a string) for the `f` command:
vimfx.get('modes.normal.follow')
```

#### `vimfx.set(pref, value)`

Sets the value of the VimFx pref `pref` to `value`.

```js
// Set the value of the Hint chars option:
vimfx.set('hint_chars', 'abcdefghijklmnopqrstuvwxyz')
// Add yet a keyboard shortcut for the `f` command:
vimfx.set('modes.normal.follow', vimfx.get('modes.normal.follow') + '  e')
```

Note: If you produce conflicting keyboard shortcuts, the order of your code does
not matter. The command that comes first in VimFx’s settings page in the Add-ons
Manager (and in the help dialog) gets the shortcut; the other one(s) do(es) not.
See the notes about order in [mode object], [category object] and [command
object] for more information about order.

```js
// Even though we set the shortcut for focusing the search bar last, the command
// for focusing the location bar “wins”, because it comes first in VimFx’s
// settings page in the Add-ons Manager.
vimfx.set('modes.normal.focus_location_bar', 'ö')
vimfx.set('modes.normal.focus_search_bar', 'ö')

// Swapping their orders also swaps the “winner”.
let {commands} = vimfx.modes.normal
;[commands.focus_location_bar.order, commands.focus_search_bar.order] =
  [commands.focus_search_bar.order, commands.focus_location_bar.order]
```

### `vimfx.addCommand(options, fn)`

Creates a new command.

**Note:** This should only be used by config file users, not by extension
authors who wish to extend VimFx. They should add commands manually to
[`vimfx.modes`] instead.

`options`:

- name: `String`. The name used when accessing the command via
  `vimfx.modes[options.mode].commands[options.name]`. It is also used for the
  pref used to store the shortcuts for the command:
  `` `custom.mode.${options.mode}.${options.name}` ``.
- description: `String`. Shown in the help dialog and VimFx’s settings page in
  the Add-ons Manager.
- mode: `String`. Defaults to `'normal'`. The mode to add the command to. The
  value has to be one of the keys of [`vimfx.modes`].
- category: `String`. Defaults to `'misc'` for Normal mode and `''`
  (uncategorized) otherwise. The category to add the command to. The
  value has to be one of the keys of [`vimfx.get('categories')`][categories].
- order: `Number`. Defaults to putting the command at the end of the category.
  The first of the default commands has the order `100` and then they increase
  by `100` per command. This allows to put new commands between two already
  existing ones.

`fn` is called when the command is activated. See the [onInput] documentation
below for more information.

Note that you have to give the new command a shortcut in VimFx’s settings page
in the Add-ons Manager or set one using `vimfx.set()` to able to use the new
command.

```js
vimfx.addCommand({
  name: 'hello',
  description: 'Log Hello World',
}, => {
  console.log('Hello World!')
})
// Optional:
vimfx.set('custom.mode.normal.hello', 'gö')
```

### `vimfx.addOptionOverrides(...rules)` and `vimfx.addKeyOverrides(...rules)`

These methods take any number of arguments. Each argument is a rule. The rules
are added in order. The methods may be run multiple times.

A rule is an `Array` of length 2:

1. The first item is a function that returns `true` if the rule should be
   applied and `false` if not. This is called the matching function.
2. The second item is the value that should be used if the rule is applied. This
   is called the override.

The rules are tried in the same order they were added. When a matching rule is
found it is applied. No more rules will be applied.

#### `vimfx.addOptionOverrides(...rules)`

The rules are matched any time the value of a VimFx pref is needed.

The matching function receives a [location object].

The override is an object whose keys are VimFx pref names and whose values
override the pref in question. The values should be formatted as in an [options
object].

```js
vimfx.addOptionOverrides(
  [ ({hostname, pathname, hash}) =>
    `${hostname}${pathname}${hash}` === 'google.com/',
    {prevent_autofocus: false}
  ]
)
```

#### `vimfx.addKeyOverrides(...rules)`

The rules are matched any time you press a key that is not part of the tail of a
multi-key shortcut.

The matching function receives a [location object] as well as the current
mode name (one of the keys of [`vimfx.modes`]).

The override is an array of keys which should not activate VimFx commands but be
sent to the page.

This allows to disable commands on specific sites. To _add_ commands on specific
sites, add them globally and then disable them on all _other_ sites.

```js
vimfx.addKeyOverrides(
  [ location => location.hostname === 'facebook.com',
    ['j', 'k']
  ]
)
```

### `vimfx.on(eventName, listener)`

Runs `listener(data)` when `eventName` is fired.

#### The `locationChange` event

Occurs when opening a new tab or navigating to a new URL causing a full page
load. The data passed to listeners is an object with the following properties:

- vim: The current [vim object].
- location: A [location object].

This can be used to enter a different mode by default on some pages (which can
be used to replace the blacklist option).

```js
vimfx.on('load', ({vim, location}) => {
  if (location.hostname === 'example.com') {
    vim.enterMode('ignore')
  }
})
```

#### The `modeChange` event

Occurs whenever the current mode in any tab changes. The initial entering of the
default mode in new tabs also counts as a mode change. The data passed to
listeners is the current [vim object].

```js
vimfx.on('modeChange', vim => {
  let mode = vimfx.modes[vim.mode].name()
  vim.notify(`Entering mode: ${mode}`)
})
```

#### The `TabSelect` event

Occurs whenever any tab in any window is selected. This is also fired when
Firefox starts for the currently selected tab. The data passed to listeners is
the `event` object passed to the standard Firefox [TabSelect] event.

### `vimfx.refresh()`

If you make changes to [`vimfx.modes`] directly you need to call
`vimfx.refresh()` for your changes to take effect.

### `vimfx.modes`

An object whose keys are mode names and whose values are [mode object]s.

This is a very low-level part of the API. It allows to:

- Access all commands and run them. This is the only thing that a config file
  user needs it for.

  ```js
    let {commands} = vimfx.modes.normal
    // Inside a custom command:
    commands.tab_new.run(args)
  ```

- Adding new commands. This is intended to be used by extension authors who wish
  to extend VimFx, not config file users. They should use the
  `vimfx.addCommand()` helper instead.

  ```js
  vimfx.modes.normal.commands.new_command = {
    pref: 'extensions.my_extension.mode.normal.new_command',
    category: 'misc',
    order: 10000,
    description: () => translate('mode.normal.new_command'),
    run: args => console.log('New command! args:', args)
  }
  ```

- Adding new modes. This is intended to be used by extension authors who wish to
  extend VimFx, not config file users.

  ```js
  vimfx.modes.new_mode = {
    name: () => translate('mode.new_mode'),
    order: 10000,
    commands: {},
    onEnter(args) {},
    onLeave(args) {},
    onInput(args, match) {
      if (match.type === 'full') {
        match.command.run(args)
      }
      return (match.type !== 'none')
    },
  }
  ```

When you’re done modifying `vimfx.modes` directly, you need to call
`vimfx.refresh()`. (That’s taken care of automatically in the
`vimfx.addCommand()` helper.)

Have a look at [modes.coffee] and [commands.coffee] for more information.

### `vimfx.get('categories')`

An object whose keys are category names and whose values are [category object]s.

```js
let categories = vimfx.get('categories')

// Add a new category.
categories.custom = {
  name: () => 'Custom commands',
  order: 10000,
}

// Swap the order of the Location and Tabs categories.
;[commands.focus_location_bar.order, categories.tabs.order] =
  [categories.tabs.order, commands.focus_location_bar.order]
```

### Mode object

A mode is an object with the follwing properties:

- name(): `Function`. Returns a human readable name of the mode used in the help
  dialog and VimFx’s settings page in the Add-ons Manager.
- order: `Number`. The first of the default modes has the order `100` and then
  they increase by `100` per mode. This allows to put new modes between two
  already existing ones.
- commands: `Object`. The keys are command names and the values are [command
  object]s.
- onEnter(data, ...args): `Function`. Called when the mode is entered.
- onLeave(data): `Function`. Called when the mode is left.
- onInput(data, match): `Function`. Called when a key is pressed.

#### onEnter, onLeave and onInput

These methods are called with an object (called `data` above) with the following
properties:

- vim: The current [vim object].
- storage: An object unique to the current [vim object] and to the current mode.
  Allows to share things between commands of the same mode by getting and
  setting keys on it.

##### onEnter

This method is called with an object as mentioned above, and after that there
may be any number of arguments (`args` in `vim.enterMode(modeName, ...args)`)
that the mode is free to do whatever it wants with.

##### onInput

The object passed to this method (see above) also has the following properties:

- isFrameEvent: `Boolean`. `true` if the event occured in web page content,
  `false` otherwise (if the event occured in the browser UI).
- count: `Number`. The count for the command. `undefined` if no count. (This is
  simply a copy of `match.count`. `match` is defined below.)

The above object should be passed to commands when running them. The mode is
free to do whatever it wants with the return value (if any) of the commands it
runs.

It also receives a [match object] as the second argument.

`onInput` should return `true` if the current keypress should not be passed on
to the browser and web pages, and `false` otherwise.

### Category object

A category is an object with the follwing properties:

- name(): `Function`. Returns a human readable name of the category used in the
  help dialog and VimFx’s settings page in the Add-ons Manager. Config file
  users adding custom categories could simply return a string; extension authors
  are encouraged to look up the name from a locale file.
- order: `Number`. The first of the default categories is the “uncategorized”
  category. It has the order `100` and then they increase by `100` per category.
  This allows to put new categories between two already existing ones.

### Command object

A command is an object with the following properties:

- pref: `String`. The pref used to store the shortcuts for the command.
- run(args): `Function`. Called when the command is activated.
- description(): `Function`. Returns a description of the command (as a string),
  shown in the help dialog and VimFx’s settings page in the Add-ons Manager.
- category: `String`. The category to add the command to. The value has to be
  one of the keys of [`vimfx.get('categories')`][categories].
- order: `Number`. The first of the default commands has the order `100` and
  then they increase by `100` per command. This allows to put new commands
  between two already existing ones.

### Match object

A `match` object has the following properties:

- type: `String`. It has one of the following values:

  - `'full'`: The current keypress, together with previous keypresses, fully
    matches a command shortcut.
  - `'partial'`: The current keypress, together with previous keypresses,
    partially matches a command shortcut.
  - `'count'`: The current keypress is not part of a command shortcut, but is a
    digit and contributes to the count of a future matched command.
  - `'none'`: The current keypress is not part of a command shortcut and does
    not contribute to a count.

- focus: `String` or `null`. The type of currently focused _element_ plus
  current pressed _key_ combo. You might not want to run commands and suppress
  the event if this value is anything other than null. It has one of the
  following values, depending on what kind of _element_ is focused and which
  _key_ was pressed:

  - `'editable'`: element: a text input or a `contenteditable` element.
    key: any pressed key.
  - `'activatable'`: element: an “activatable” element (link or button).
    key: see the [`activatable_element_keys`] option.
  - `'adjustable'`: element: an “adjustable” element (form control or video
    player). key: see the [`adjustable_element_keys`] option.
  - `'other'`: element: some other kind of element that can receive keystrokes,
    for example an element in fullscreen mode. key: any pressed key.

  If none of the above criteria is met, the value is `null`, which means that
  the currently focused element does not appear to respond to keystrokes in any
  special way.

- command: `null` unless `type` is `'full'`. Then it is the matched command (a
  [command object]).

  The matched command should usually be run at this point. It is suitable to
  pass on the object passed to [onInput] to the command. Some modes might choose
  to add extra properties to the object first. (That is favored over passing
  several arguments, since it makes it easier for the command to in turn pass
  the same data it got on to another command, if needed.)

  Usually the return value of the command isn’t used, but that’s up to the mode.

- count: `Number`. The count for the command. `undefined` if no count.

- specialKeys: `Object`. The keys may be any of the following:

  - `<force>`
  - `<late>`

  If a key exists, its value is always `true`. The keys that exist indicate the
  [special keys] for the sequence used for the matched command (if any).

- keyStr: `String`. The current keypress represented as a string.

- unmodifiedKey: `String`. `keyStr` without modifiers.

- toplevel: `Boolean`. Whether or not the match was a toplevel match in the
  shortcut key tree. This is `true` unless the match is part of the tail of a
  multi-key shortcut.

### Vim object

There is one `vim` object per tab.

A `vim` object has the following properties:

- window: [`Window`]. The current Firefox window object. Most commands
  interacting with Firefox’s UI use this.

- browser: [`Browser`]. The `browser` that this vim object handles.

- options: `Object`. Provides access to all of VimFx’s options. It is an
  [options object].

- mode: `String`. The current mode name.

- enterMode(modeName, ...args): `Function`. Enter mode `modeName`, passing
  `...args` to the mode. It is up to every mode to do whatever it wants to with
  `...args`.

- isFrameEvent(event): `Function`. Returns `true` if `event` occurred in web
  page content, and `false` otherwise (if it occurred in Firefox’s UI).

- isCurrent(): `Function`. Returns whether this vim object is the currently used
  one. In other words, if the tab that this vim object manages is the currently
  selected tab in the current Firefox window. Note, though, that the current
  Firefox window is not necessarily the current window of your operating system.

- notify(title, options = {}): `Function`. Display a notification with the title
  `title` (a `String`). If you need more text than a title, use `options.body`.
  See [`Notification`] for more information.

- markPageInteraction(): `Function`. Marks that the user has interacted with the
  page. After that [autofocus prevention] is not done anymore. Commands
  interacting with web page content might want to do this.

**Warning:** There are also properties starting with an underscore on `vim`
objects. They are private, and not supposed to be used outside of VimFx’s own
source code. They may change at any time.

### Options object

An `options` object provides access to all of VimFx’s options. It is an object
whose keys are VimFx pref names.

Note that the values are not just simply `vimfx.get(pref)` for the `pref` in
question; they are _parsed_ (`parse(vimfx.get(pref))`):

- Space-separated prefs are parsed into arrays of strings.

- `black_list` and `{prev,next}_patterns` are parsed into arrays of regular
  expressions.

(See [parse-prefs.coffee] for all details.)

Any [option overrides] are automatically taken into account when getting an
option value.

The [special options] are also available on this object.


### Location object

A location object is very similar to [`window.location`] in web pages.
Technically, it is a [`URL`] instance. You can experient with the current
location object by opening the [web console] and entering `location`.


## Stability

The public API is currently **experimental** and therefore **unstable.** Things
might break with new VimFx versions.

As soon as VimFx 1.0.0 is released backwards compatibility will be a priority
and won’t be broken until VimFx 2.0.0.

[option overrides]: #vimfxaddOptionOverridesrules
[categories]: #vimfxgetcategories
[`vimfx.modes`]: #vimfxmodes
[onInput]: #oninput
[mode object]: #mode-object
[category object]: #category-object
[command object]: #command-object
[match object]: #match-object
[vim object]: #vim-object
[options object]: #options-object
[location object]: #location-object

[blacklisted]: options.md#blacklist
[special options]: options.md#special-options
[config file]: config-file.md
[bootstrap.js]: config-file.md#bootstrapjs
[autofocus prevention]: options.md#prevent-autofocus
[`activatable_element_keys`]: options.md#activatable_element_keys
[`adjustable_element_keys`]: options.md#adjustable_element_keys

[special keys]: TODO

[defaults.coffee]: ../extension/lib/defaults.coffee
[parse-prefs.coffee]: ../extension/lib/parse-prefs.coffee
[modes.coffee]: ../extension/lib/modes.coffee
[commands.coffee]: ../extension/lib/commands.coffee
[vim.coffee]: ../extension/lib/vim.coffee

[`Window`]: https://developer.mozilla.org/en-US/docs/Web/API/Window
[`Browser`]: https://developer.mozilla.org/en-US/docs/Mozilla/Tech/XUL/browser
[`Notification`]: https://developer.mozilla.org/en-US/docs/Web/API/Notification
[`window.location`]: https://developer.mozilla.org/en-US/docs/Web/API/Location
[`URL`]: https://developer.mozilla.org/en-US/docs/Web/API/URL
[TabSelect]: https://developer.mozilla.org/en-US/docs/Web/Events/TabSelect
[web console]: https://developer.mozilla.org/en-US/docs/Tools/Web_Console
[about:config]: http://kb.mozillazine.org/About:config
