## Introduction to React.js

React.js is the new popular guy around the "JavaScript Frameworks" block, and it shines for its simplicity. Where other frameworks implement a complete _MVC_ framework, we could say React only implements the _V_ (in fact, some people replace their framework's _V_ with React). React applications are built over 2 main principles: _Components_ and _States_. _Components_ can be made of other smaller components, built-in or custom; the _State_ drives what the guys at Facebook call _one-way reactive data flow_, meaning that our UI will react to every change of state.

One of the good things about React is that it doesn't require any additional dependencies, making it pluggable with virtually any other JS library. Taking advantage of this feature, we are going to include it into our Rails stack to build a frontend-powered application, or you might say, a Rails view on steroids.

## A mock expense tracking app

For this guide, we are building a small application from scratch to keep track of our expenses; each record will consist of a date, a title and an amount. A record will be treated as _Credit_ if its amount is greater than zero, otherwise it will be treated as _Debit_. Here is a mockup of the project:

![Mockup](//i.imgur.com/QB8S7ap.png)

Summarizing, the application will behave as follows:

* When the user creates a new record through the horizontal form, it will be appended to the records table
* The user will be able to inline-edit any existing record
* Clicking on any _Delete_ button will remove the associated record from the table
* Adding, editing or removing an existing record will update the amount boxes at the top of the page

## Initializing our React.js on Rails project

First things first, we need to start our brand new Rails project, we are naming it `Accounts`:

```
  rails new accounts
```

For this project's UI, we'll be using Twitter Bootstrap. The installation process is out of the scope of this post, but you can install the `bootstrap-sass` official gem following the instructions from the [official github repo](https://github.com/twbs/bootstrap-sass).

Once our project has been initialized, we proceed to include __React__. For this post I decided to include it via the official gem [react-rails](https://github.com/reactjs/react-rails) because we are going to take advantage of some cool features included in this gem, but there are other ways to achieve this task, like using [Rails assets](https://rails-assets.org/) or even downloading the source package from [the official page](https://facebook.github.io/react/) and pasting it into our `javascripts` folder.

If you have been developing Rails apps, you know how easy it is to install a gem: Add `react-rails` to your _Gemfile_.

```ruby
  gem 'react-rails', '~> 1.0'
```

Then, (kindly) tell Rails to install the new gems:

```
  bundle install
```

`react-rails` comes with an installation script, which will create a `components.js` file and a `components` directory under `app/assets/javascripts` where our React components will live.

```
  rails g react:install
```

If you take a look at your `application.js` file after running the installer, you will notice three new lines:

```javascript
  //= require react
  //= require react_ujs
  //= require components
```

Basically, it includes the actual __react__ library, the `components` manifest file and a kind of familiar file ended in _ujs_. As you might have guessed for the file's name, _react-rails_ includes an unobtrusive JavaScript driver which will help us to mount our React components and will also handle _Turbolinks_ events.

## Creating the Resource

We are going to build a `Record` resource, which will include a `date`, a `title`, and an `amount`. Instead of using the `scaffold` generator, we are going to use the `resource` generator, as we are not going to be using all of the files and methods created by the `scaffold`generator. Another option might be running the `scaffold` generator and then proceed to delete the unused files/methods, but our project can turn a little messy after this. Inside your project, run the following command:

```
  rails g resource Record title date:date amount:float
```

After some magic, we will end up with a new `Record` model, controller, and routes. We just need to create our database and run pending migrations.

```
  rake db:create db:migrate
```

As a plus, you can create a couple of records through `rails console`:

```
  Record.create title: 'Record 1', date: Date.today, amount: 500
  Record.create title: 'Record 2', date: Date.today, amount: -100
```

Don't forget to start your server with `rails s`.

Done! We're ready to write some code.

## Nesting Components: Listing Records

For our first task, we need to render any existing record inside a table. First of all, we need to create an `index` action inside of our `RecordsController`:

```ruby
  # app/controllers/records_controller.rb

  class RecordsController < ApplicationController
    def index
      @records = Record.all
    end
  end
```

Next, we need to create a new file `index.html.erb` under `apps/views/records/`, this file will act as a bridge between our Rails app and our React Components. To achieve this task, we will use the helper method `react_component`, which receives the name of the React component we want to render along with the data we want to pass into it.

```erb
  <%# app/views/records/index.html.erb %>

  <%= react_component 'Records', { data: @records } %>
```

It is worth mentioning this helper is provided by the `react-rails` gem, if you decide to use other React integration method, this helper will not be available.

You can now navigate to `localhost:3000/records`. Obviously, this won't work yet because of the lack of a `Records` React component, but if we take a look at the generated HTML inside the browser window, we can spot something like the following code

```markup
  <div data-react-class="Records" data-react-props="{...}">
  </div>
```

With this markup present, `react_ujs` will detect we are trying to render a React component and will instantiate it, including the properties we sent through `react_component`, in our case, the contents of `@records`.

The time has come for us to build our First React component, inside the `javascripts/components` directory, create a new file called `records.js.coffee`, this file will contain our `Records` component.

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    render: ->
      React.DOM.div
        className: 'records'
        React.DOM.h2
          className: 'title'
          'Records'
```

Each component requires a `render` method, which will be in charge of rendering the component itself. The render method should return an instance of `ReactComponent`, this way, when React executes a re-render, it will be performed in an optimal way (as React detects the existence of new nodes through building a virtual DOM in memory). In the snippet above we created an instance of `h2`, a built-in `ReactComponent`.

NOTE: Another way to instantiate ReactComponents inside the render method is through `JSX` syntax. The following snippet is equivalent to the previous one:

```coffeescript
  render: ->
    `<div className="records">
      <h2 className="title"> Records </h2>
    </div>`
```

Personally, when I am working with CoffeeScript, I prefer using the `React.DOM` syntax over `JSX` because the code will arrange in a hierarchical structure by itself, similar to _HAML_. On the other hand, if you are trying to integrate React into an existing application built with _erb_, you have the option to reuse your existing _erb_ code and convert it into _JSX_.

You can refresh your browser now.

![Records component](//i.imgur.com/jrehS5B.png)

Perfect! We have rendered our first React Component. Now, it's time to display our records.

Besides the `render` method, React components rely on the use of _properties_ to communicate with other components and _states_ to detect whether a re-render is required or not. We need to initialize our component's state and properties with the desired values:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    getInitialState: ->
      records: @props.data
    getDefaultProps: ->
      records: []
    render: ->
      ...
```

The method `getDefaultProps` will initialize our component's properties in case we forget to send any data when instantiating it, and the `getInitialState` method will generate the initial state of our component. Now we need to actually display the records provided by our Rails view.

It looks like we are going to need a helper method to format _amount_ strings, we can implement a simple string formatter and make it accesible to all of our _coffee_ files. Create a new `utils.js.coffee` file under `javascripts/` with the following contents:

```coffeescript
  # app/assets/javascripts/utils.js.coffee

  @amountFormat = (amount) ->
    '$ ' + Number(amount).toLocaleString()
```

We need to create a new `Record` component to display each individual record, create a new file `record.js.coffee` under the `javascripts/components` directory and insert the following contents:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    render: ->
      React.DOM.tr null,
        React.DOM.td null, @props.record.date
        React.DOM.td null, @props.record.title
        React.DOM.td null, amountFormat(@props.record.amount)
```

The `Record` component will display a table row containing table cells for each record attribute. Don't worry about those `null`s in the `React.DOM.*` calls, it means we are not sending attributes to the components. Now update the `render` method inside the `Records` component with the following code:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    render: ->
      React.DOM.div
        className: 'records'
        React.DOM.h2
          className: 'title'
          'Records'
        React.DOM.table
          className: 'table table-bordered'
          React.DOM.thead null,
            React.DOM.tr null,
              React.DOM.th null, 'Date'
              React.DOM.th null, 'Title'
              React.DOM.th null, 'Amount'
          React.DOM.tbody null,
            for record in @state.records
              React.createElement Record, key: record.id, record: record
```

Did you see what just happened? We created a table with a header row, and inside of the body table we are creating a `Record` element for each existing record. In other words, we are nesting built-in/custom React components. Pretty cool, huh?

When we handle dynamic children (in this case, records) we need to provide a `key` property to the dynamically generated elements so React won't have a hard time refreshing our UI, that's why we send `key: record.id` along with the actual record when creating `Record` elements. If we don't do so, we will receive a warning message in the browser's JS console (and probably some headaches in the near future).

![Records component](//i.imgur.com/vETiEeF.png)

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/bf1d80cf3d23a9a5e4aa48c86368262b7a7bd809), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/bf1d80cf3d23a9a5e4aa48c86368262b7a7bd809).

## Parent-Child communication: Creating Records

Now that we are displaying all the existing records, it would be nice to include a form to create new records, let's add this new feature to our React/Rails application.

First, we need to add the `create` method to our Rails controller (don't forget to use _strong_params_):

```ruby
  # app/controllers/records_controller.rb

  class RecordsController < ApplicationController
    ...

    def create
      @record = Record.new(record_params)

      if @record.save
        render json: @record
      else
        render json: @record.errors, status: :unprocessable_entity
      end
    end

    private

      def record_params
        params.require(:record).permit(:title, :amount, :date)
      end
  end
```

Next, we need to build a React component to handle the creation of new records. The component will have its own _state_ to store `date`, `title` and `amount`. Create a new `record_form.js.coffee` file under `javascripts/components` with the following code:

```coffeescript
  # app/assets/javascripts/components/record_form.js.coffee

  @RecordForm = React.createClass
    getInitialState: ->
      title: ''
      date: ''
      amount: ''
    render: ->
      React.DOM.form
        className: 'form-inline'
        React.DOM.div
          className: 'form-group'
          React.DOM.input
            type: 'text'
            className: 'form-control'
            placeholder: 'Date'
            name: 'date'
            value: @state.date
            onChange: @handleChange
        React.DOM.div
          className: 'form-group'
          React.DOM.input
            type: 'text'
            className: 'form-control'
            placeholder: 'Title'
            name: 'title'
            value: @state.title
            onChange: @handleChange
        React.DOM.div
          className: 'form-group'
          React.DOM.input
            type: 'number'
            className: 'form-control'
            placeholder: 'Amount'
            name: 'amount'
            value: @state.amount
            onChange: @handleChange
        React.DOM.button
          type: 'submit'
          className: 'btn btn-primary'
          disabled: !@valid()
          'Create record'
```

Nothing too fancy, just a regular Bootstrap inline form. Notice how we are defining the `value` attribute to set the input's value and the `onChange` attribute to attach a handler method which will be called on every keystroke; the `handleChange` handler method will use the `name` attribute to detect which input triggered the event and update the related `state` value:

```coffeescript
  # app/assets/javascripts/components/record_form.js.coffee

  @RecordForm = React.createClass
    ...
    handleChange: (e) ->
      name = e.target.name
      @setState "#{ name }": e.target.value
    ...

```

We are just using string interpolation to dynamically define object keys, equivalent to `@setState title: e.target.value` when `name` equals `title`. But why do we have to use `@setState`? Why can't we just set the desired value of `@state` as we usually do in regular JS Objects? Because `@setState` will perform 2 actions, it:

1. Updates the component's `state`
2. Schedules a UI verification/refresh based on the new state

It is very important to have this information in mind every time we use `state` inside our components.

Lets take a look at the _submit_ button, just at the very end of the `render` method:

```coffeescript
  # app/assets/javascripts/components/record_form.js.coffee

  @RecordForm = React.createClass
    ...
    render: ->
      ...
      React.DOM.form
        ...
        React.DOM.button
          type: 'submit'
          className: 'btn btn-primary'
          disabled: !@valid()
          'Create record'

```

We defined a `disabled` attribute with the value of `!@valid()`, meaning that we are going to implement a `valid` method to evaluate if the data provided by the user is correct.

```coffeescript
  # app/assets/javascripts/components/record_form.js.coffee

  @RecordForm = React.createClass
    ...
    valid: ->
      @state.title && @state.date && @state.amount
    ...

```

For the sake of simplicity we are only validating `@state` attributes against empty strings. This way, every time the state gets updated, the _Create record_ button is enabled/disabled depending on the validity of the data.

![New record form](//i.imgur.com/EuPQ64m.png)
![New record form](//i.imgur.com/ZsPiUxq.png)

Now that we have our controller and form in place, it's time to submit our new record to the server. We need to handle the form's `submit` event. To achieve this task, we need to add an `onSubmit` attribute to our form and a new `handleSubmit` method (the same way we handled `onChange` events before):

```coffeescript
  # app/assets/javascripts/components/record_form.js.coffee

  @RecordForm = React.createClass
    ...
    handleSubmit: (e) ->
      e.preventDefault()
      $.post '', { record: @state }, (data) =>
        @props.handleNewRecord data
        @setState @getInitialState()
      , 'JSON'

    render: ->
      React.DOM.form
        className: 'form-inline'
        onSubmit: @handleSubmit
      ...
```

Let's review the new method line by line:

1. Prevent the form's HTTP submit
2. POST the new `record` information to the current URL
3. Success callback

The `success` callback is the key of this process, after successfully creating the new record _someone_ will be notified about this action and the `state` is restored to its initial value. Do you remember early in the post when I mentioned that components communicate with other components through properties (or _@props_)? Well, this is it. Our current component sends data back to the parent component through `@props.handleNewRecord` to notify it about the existence of a new record.

As you might have guessed, wherever we create our `RecordForm` element, we need to pass a `handleNewRecord` property with a method reference into it, something like `React.createElement RecordForm, handleNewRecord: @addRecord`. Well, the parent `Records` component is the "wherever", as it has a _state_ with all of the existing records, we need to update its state with the newly created record.

Add the new `addRecord` method inside `records.js.coffee` and create the new `RecordForm` element, just after the `h2` title (inside the `render` method).

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    addRecord: (record) ->
      records = @state.records.slice()
      records.push record
      @setState records: records
    render: ->
      React.DOM.div
        className: 'records'
        React.DOM.h2
          className: 'title'
          'Records'
        React.createElement RecordForm, handleNewRecord: @addRecord
        React.DOM.hr null
      ...
```

Refresh your browser, fill in the form with a new record, click _Create record_... No suspense this time, the record was added almost immediately and the form gets cleaned after submit, refresh again just to make sure the backend has stored the new data.

![New record](//i.imgur.com/635Wuv7.png)

If you have used other JS frameworks along with Rails (for example, AngularJS) to build similar features, you might have run into problems because your `POST` requests don't include the `CSRF` token required by Rails, so, why didn't we run into this same issue? Easy, because we are using `jQuery` to interact with our backend, and Rails' `jquery_ujs` unobtrusive driver will include the `CSRF` token on every _AJAX_ request for us. Cool!

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/f4708e19f8be929471bc0c8c2bda93f36b9a7f23), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/f4708e19f8be929471bc0c8c2bda93f36b9a7f23).

## Reusable Components: Amount Indicators

What would an application be without some (nice) indicators? Let's add some boxes at the top of our window with some useful information. Our goal for this section is to show 3 values: Total credit amount, total debit amount and Balance. This looks like a job for 3 components, or maybe just one with properties?

We can build a new `AmountBox` component which will receive three properties: `amount`, `text` and `type`. Create a new file called `amount_box.js.coffee` under `javascripts/components/` and paste the following code:

```coffeescript
  # app/assets/javascripts/components/amount_box.js.coffee

  @AmountBox = React.createClass
    render: ->
      React.DOM.div
        className: 'col-md-4'
        React.DOM.div
          className: "panel panel-#{ @props.type }"
          React.DOM.div
            className: 'panel-heading'
            @props.text
          React.DOM.div
            className: 'panel-body'
            amountFormat(@props.amount)
```

We are just using Bootstrap's `panel` element to display the information in a "blocky" way, and setting the color through the `type` property. We have also included a really simple amount formatter method called `amountFormat` which reads the `amount` property and displays it in currency format.

In order to have a complete solution, we need to create this element (3 times) inside of our main component, sending the required properties depending on the data we want to display. Let's build the calculator methods first, open the `Records` component and add the following methods:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    credits: ->
      credits = @state.records.filter (val) -> val.amount >= 0
      credits.reduce ((prev, curr) ->
        prev + parseFloat(curr.amount)
      ), 0
    debits: ->
      debits = @state.records.filter (val) -> val.amount < 0
      debits.reduce ((prev, curr) ->
        prev + parseFloat(curr.amount)
      ), 0
    balance: ->
      @debits() + @credits()
    ...
```

`credits` sums all the records with an amount greater than 0, `debits` sums all the records with an amount lesser than 0 and balance is self-explanatory. Now that we have the calculator methods in place, we just need to create the `AmountBox` elements inside the `render` method (just above the `RecordForm` component):

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    render: ->
      React.DOM.div
        className: 'records'
        React.DOM.h2
          className: 'title'
          'Records'
        React.DOM.div
          className: 'row'
          React.createElement AmountBox, type: 'success', amount: @credits(), text: 'Credit'
          React.createElement AmountBox, type: 'danger', amount: @debits(), text: 'Debit'
          React.createElement AmountBox, type: 'info', amount: @balance(), text: 'Balance'
        React.createElement RecordForm, handleNewRecord: @addRecord
    ...
```

We are done with this feature! Refresh your browser, you should see three boxes displaying the amounts we've calculated earlier. But wait! There's more! Create a new record and see the magic work...

![Indicators](//i.imgur.com/YWEcQHh.png)

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/8d6f0a4fb62f2a9abd5d34d502461388863302cb), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/8d6f0a4fb62f2a9abd5d34d502461388863302cb).

## setState/replaceState: Deleting Records

The next feature in our list is the ability to delete records, we need a new `Actions` column in our records table, this column will have a `Delete` button for each record, pretty standard UI. As in our previous example, we need to create the `destroy` method in our Rails controller:

```ruby
  # app/controllers/records_controller.rb

  class RecordsController < ApplicationController
    ...

    def destroy
      @record = Record.find(params[:id])
      @record.destroy
      head :no_content
    end

    ...
  end
```

That is all the server-side code we will need for this feature. Now, open your `Records` React component and add the _Actions_ column at the rightmost position of the table header:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    render: ->
      ...
      # almost at the bottom of the render method
      React.DOM.table
        React.DOM.thead null,
          React.DOM.tr null,
            React.DOM.th null, 'Date'
            React.DOM.th null, 'Title'
            React.DOM.th null, 'Amount'
            React.DOM.th null, 'Actions'
        React.DOM.tbody null,
          for record in @state.records
            React.createElement Record, key: record.id, record: record
```

And finally, open the `Record` component and add an extra column with a _Delete_ link:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    render: ->
      React.DOM.tr null,
        React.DOM.td null, @props.record.date
        React.DOM.td null, @props.record.title
        React.DOM.td null, amountFormat(@props.record.amount)
        React.DOM.td null,
          React.DOM.a
            className: 'btn btn-danger'
            'Delete'
```

Save your file, refresh your browser and... We have a useless button with no events attached to it!

![Delete records](//i.imgur.com/JV1KVNK.png)

Let's add some functionality to it. As we learned from our `RecordForm` component, the way to go here is:

1. Detect an event inside the child `Record` component (_onClick_)
2. Perform an action (send a _DELETE_ request to the server in this case)
3. Notify the parent `Records` component about this action (sending/receiving a handler method through _props_)
4. Update the `Record` component's state

To implement _step 1_, we can add a handler for `onClick` to `Record` the same way we added a handler for `onSubmit` to `RecordForm` to create new records. Fortunately for us, React implements most of the common browser events in a normalized way, so we don't have to worry about cross-browser compatibility (you can take a look at the complete events list [here](http://facebook.github.io/react/docs/events.html#supported-events)).

Re-open the `Record` component, add a new `handleDelete` method and an `onClick` attribute to our "useless" delete button as follows:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    handleDelete: (e) ->
      e.preventDefault()
      # yeah... jQuery doesn't have a $.delete shortcut method
      $.ajax
        method: 'DELETE'
        url: "/records/#{ @props.record.id }"
        dataType: 'JSON'
        success: () =>
          @props.handleDeleteRecord @props.record
    render: ->
      React.DOM.tr null,
        React.DOM.td null, @props.record.date
        React.DOM.td null, @props.record.title
        React.DOM.td null, amountFormat(@props.record.amount)
        React.DOM.td null,
          React.DOM.a
            className: 'btn btn-danger'
            onClick: @handleDelete
            'Delete'
```

When the _delete_ button gets clicked, `handleDelete` sends an AJAX request to the server to delete the record in the backend and, after this, it notifies the parent component about this action through the `handleDeleteRecord` handler available through _props_, this means we need to adjust the creation of `Record` elements in the parent component to include the extra property `handleDeleteRecord`, and also implement the actual handler method in the parent:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    deleteRecord: (record) ->
      records = @state.records.slice()
      index = records.indexOf record
      records.splice index, 1
      @replaceState records: records
    render: ->
      ...
      # almost at the bottom of the render method
      React.DOM.table
        React.DOM.thead null,
          React.DOM.tr null,
            React.DOM.th null, 'Date'
            React.DOM.th null, 'Title'
            React.DOM.th null, 'Amount'
            React.DOM.th null, 'Actions'
        React.DOM.tbody null,
          for record in @state.records
            React.createElement Record, key: record.id, record: record, handleDeleteRecord: @deleteRecord
```

Basically, our `deleteRecord` method copies the current component's `records` state, performs an index search of the record to be deleted, splices it from the array and updates the component's state, pretty standard JavaScript operations.

We introduced a new way of interacting with the _state_, `replaceState`; the main difference between `setState` and `replaceState` is that the first one will only update one key of the _state_ object, the second one will completely __override__ the current state of the component with whatever new object we send.

After updating this last bit of code, refresh your browser window and try to delete a record, a couple of things should happen:

1. The records should disappear from the table and...
2. The indicators should update the amounts instantly, no additional code is required

![Delete records](//i.imgur.com/fjoCUYN.png)

We are almost done with our application, but before implementing our last feature, we can apply a small refactor and, at the same time, introduce a new React feature.

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/1a4dc646e53fecebc821c709347aae774e9ef170), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/1a4dc646e53fecebc821c709347aae774e9ef170).

## Refactor: State Helpers

At this point, we have a couple of methods where the state gets updated without any difficulty as our data is not what you might call "complex", but imagine a more complex application with a multi-level JSON state, you can picture yourself performing deep copies and juggling with your state data. React includes some fancy _state helpers_ to help you with some of the heavy lifting, no matter how deep your _state_ is, these helpers will let you manipulate it with more freedom using a kind-of MongoDB's query language (or at least that's what [React's documentation says](https://facebook.github.io/react/docs/update.html)).

Before using these helpers, first we need to configure our Rails application to include them. Open your project's `config/application.rb` file and add `config.react.addons = true` at the bottom of the Application block:

```ruby
  # config/application.rb

  ...
  module Accounts
    class Application < Rails::Application
      ...
      config.react.addons = true
    end
  end
```

For the changes to take effect, restart your rails server, I repeat, __restart your rails server__. Now we have access to the state helpers through `React.addons.update`, which will process our _state_ object (or any other object we send to it) and apply the provided _commands_. The two commands we will be using are `$push` and `$splice` (I'm borrowing the explanation of these commands from the official [React documentation](https://facebook.github.io/react/docs/update.html#available-commands)):

* `{$push: array}` `push()` all the items in `array` on the target.
* `{$splice: array of arrays}` for each item in `arrays` call `splice()` on the target with the parameters provided by the item.

We're about to simplify `addRecord` and `deleteRecord` from the `Record` component using these helpers, as follows:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    addRecord: (record) ->
      records = React.addons.update(@state.records, { $push: [record] })
      @setState records: records
    deleteRecord: (record) ->
      index = @state.records.indexOf record
      records = React.addons.update(@state.records, { $splice: [[index, 1]] })
      @replaceState records: records
```

Shorter, more elegant and with the same results, feel free to reload your browser and ensure nothing got broken.

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/d19127f40ae2f795a30b7de6470cde95d3734eee), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/d19127f40ae2f795a30b7de6470cde95d3734eee).

## Reactive Data Flow: Editing Records

For the final feature, we are adding an extra _Edit_ button, next to each _Delete_ button in our records table. When this _Edit_ button gets clicked, it will toggle the entire row from a read-only _state_ (wink wink) to an editable state, revealing an inline form where the user can update the record's content. After submitting the updated content or canceling the action, the record's row will return to its original read-only _state_.

As you might have guessed from the previous paragraph, we need to handle _mutable_ data to toggle each record's state inside of our `Record` component. This is a use case of what React calls _reactive data flow_. Let's add an `edit` flag and a `handleToggle` method to `record.js.coffee`:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    getInitialState: ->
      edit: false
    handleToggle: (e) ->
      e.preventDefault()
      @setState edit: !@state.edit
    ...
```

The `edit` flag will default to `false`, and `handleToggle` will change `edit` from false to true and vice versa, we just need to trigger `handleToggle` from a user `onClick` event.

Now, we need to handle two row versions (read-only and form) and display them conditionally depending on `edit`. Luckily for us, as long as our `render` method returns a React element, we are free to perform any actions in it; we can define a couple of helper methods `recordRow` and `recordForm` and call them conditionally inside of `render` depending on the contents of `@state.edit`.

We already have an initial version of `recordRow`, it's our current `render` method. Let's move the contents of `render` to our brand new `recordRow` method and add some additional code to it:


```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    ...
    recordRow: ->
      React.DOM.tr null,
        React.DOM.td null, @props.record.date
        React.DOM.td null, @props.record.title
        React.DOM.td null, amountFormat(@props.record.amount)
        React.DOM.td null,
          React.DOM.a
            className: 'btn btn-default'
            onClick: @handleToggle
            'Edit'
          React.DOM.a
            className: 'btn btn-danger'
            onClick: @handleDelete
            'Delete'
    ...
```

We only added an additional `React.DOM.a` element which listens to `onClick` events to call `handleToggle`.

Moving forward, the implementation of `recordForm` will follow a similar structure, but with input fields in each cell. We are going to use a new `ref` attribute for our inputs to make them accessible; as this component doesn't handle a _state_, this new attribute will let our component read the data provided by the user through `@refs`:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    ...
    recordForm: ->
      React.DOM.tr null,
        React.DOM.td null,
          React.DOM.input
            className: 'form-control'
            type: 'text'
            defaultValue: @props.record.date
            ref: 'date'
        React.DOM.td null,
          React.DOM.input
            className: 'form-control'
            type: 'text'
            defaultValue: @props.record.title
            ref: 'title'
        React.DOM.td null,
          React.DOM.input
            className: 'form-control'
            type: 'number'
            defaultValue: @props.record.amount
            ref: 'amount'
        React.DOM.td null,
          React.DOM.a
            className: 'btn btn-default'
            onClick: @handleEdit
            'Update'
          React.DOM.a
            className: 'btn btn-danger'
            onClick: @handleToggle
            'Cancel'
    ...
```

Do not be afraid, this method might look big, but it is just our HAML-like syntax. Notice we are calling `@handleEdit` when the user clicks on the _Update_ button, we are about to use a similar flow as the one implemented to delete records.

Do you notice something different on how `React.DOM.input`s are being created? We are using `defaultValue` instead of `value` to set the initial input values, this is because __using just `value` without `onChange` will end up creating read-only inputs__.

Finally, the `render` method boils down to the following code:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    ...
    render: ->
      if @state.edit
        @recordForm()
      else
        @recordRow()
```

You can refresh your browser to play around with the new toggle behavior, but don't submit any changes yet as we haven't implemented the actual _update_ functionality. 

![Toggle row](//i.imgur.com/dwNuuXw.png)
![Toggle row](//i.imgur.com/jaQ1rId.png)

To handle record updates, we need to add the `update` method to our Rails controller:

```ruby
  # app/controllers/records_controller.rb

  class RecordsController < ApplicationController
    ...
    def update
      @record = Record.find(params[:id])
      if @record.update(record_params)
        render json: @record
      else
        render json: @record.errors, status: :unprocessable_entity
      end
    end
    ...
  end
```

Back to our `Record` component, we need to implement the `handleEdit` method which will send an _AJAX_ request to the server with the updated `record` information, then it will notify the parent component by sending the updated version of the record via the `handleEditRecord` method, this method will be received through `@props`, the same way we did it before when deleting records:

```coffeescript
  # app/assets/javascripts/components/record.js.coffee

  @Record = React.createClass
    ...
    handleEdit: (e) ->
      e.preventDefault()
      data =
        title: React.findDOMNode(@refs.title).value
        date: React.findDOMNode(@refs.date).value
        amount: React.findDOMNode(@refs.amount).value
      # jQuery doesn't have a $.put shortcut method either
      $.ajax
        method: 'PUT'
        url: "/records/#{ @props.record.id }"
        dataType: 'JSON'
        data:
          record: data
        success: (data) =>
          @setState edit: false
          @props.handleEditRecord @props.record, data
    ...
```

For the sake of simplicity, we are not validating user data, we just read it through `React.findDOMNode(@refs.fieldName).value` and sending it verbatim to the backend. Updating the state to toggle _edit_ mode on `success` is not mandatory, but the user will definitely thank us for that.

Last, but not least, we just need to update the state on the `Records` component to overwrite the former record with the newer version of the child record and let React perform its magic. The implementation might look like this:

```coffeescript
  # app/assets/javascripts/components/records.js.coffee

  @Records = React.createClass
    ...
    updateRecord: (record, data) ->
      index = @state.records.indexOf record
      records = React.addons.update(@state.records, { $splice: [[index, 1, data]] })
      @replaceState records: records
    ...
    render: ->
      ...
      # almost at the bottom of the render method
      React.DOM.table
        React.DOM.thead null,
          React.DOM.tr null,
            React.DOM.th null, 'Date'
            React.DOM.th null, 'Title'
            React.DOM.th null, 'Amount'
            React.DOM.th null, 'Actions'
        React.DOM.tbody null,
          for record in @state.records
            React.createElement Record, key: record.id, record: record, handleDeleteRecord: @deleteRecord, handleEditRecord: @updateRecord
```

As we have learned on the previous section, using `React.addons.update` to change our state might result on more concrete methods. The final link between `Records` and `Record` is the method `@updateRecord` sent through the `handleEditRecord` property.

Refresh your browser for the last time and try updating some existing records, notice how the amount boxes at the top of the page keep track of every record you change.

![Edit record](//i.imgur.com/qLPVSlU.png)

We are done! Smile, we have just built a small Rails + React application from scratch!

You can take a look at the resulting code of this section [here](https://github.com/fervisa/accounts-react-rails/tree/1a62dd0e48b31aa55659e0035e754cee1776aa61), or just the changes introduced by this section [here](https://github.com/fervisa/accounts-react-rails/commit/1a62dd0e48b31aa55659e0035e754cee1776aa61).

## Closing thoughts: React.js Simplicity & Flexibility 

We have examined some of React's functionalities and we learned that it barely introduces new concepts. I have heard comments of people saying X or Y JavaScript framework has a steep learning curve because of all the new concepts introduced, this is not React's case; it implements core JavaScript concepts such as event handlers and bindings, making it easy to adopt and learn. Again, one of its strengths is its simplicity.

We also learned by the example how to integrate it into the Rails' assets pipeline and how well it plays along with CoffeeScript, jQuery, Turbolinks, and the rest of the Rails' orchestra. But this is not the only way of achieving the desired results. For example, if you don't use Turbolinks (hence, you don't need `react_ujs`) you can use [Rails Assets](https://rails-assets.org/) instead of the `react-rails` gem, you could use [Jbuilder](https://github.com/rails/jbuilder) to build more complex _JSON_ responses instead of rendering _JSON_ objects, and so on; you would still be able to get the same wonderful results.

React will definitely boost your frontend abilities, making it a great library to have under your Rails' toobelt.
