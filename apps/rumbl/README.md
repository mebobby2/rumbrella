# Rumbl

# Set up
* mix deps.get
* mix ecto.create
* mix phoenix.server

# Phoenix Notes

## Phoenix plugs

Web applications in Phoenix are pipelines of plugs. The basic flow of traditional applications is endpoint, router, pipeline, controller.

## Plugs

Plugs are just functions, whether it's a function plug or a module plug (modules is just a collection of functions). A module plug must have two functions, init and call. init will happen at compile time. Plug uses the result of init as the second argument to call. Because init is called at compilation time, it’s the perfect place to validate options and prepare some of the work. That way, call can be as fast as possible. Since call is the workhorse, we want it to do as little work as possible.

## Forms not backed by changesets
Usually, the first arguement to a form_for is a changeset.
```
form_for @changeset...
```
But if you have a form that is not backed by a changeset, such as a login or search form, you can pass in %Plug.Conn{} struct instead.
```
form_for @conn...
```

## Phoenix MVC
You already know a bit about the differences between traditional MVC and Phoenix’s tweak from the perspective of controllers. More explicitly, we’d like to keep functions with side effects—the ones that change the world around us—in the controller while the model and view layers remain side effect free. Since Ecto splits the responsibilities between the repository and its data API, it fits our world view perfectly.

## Prefer db references and indexes

Constraints allow us to use underlying relational database features to help us maintain database integrity. For example, let’s validate our categories. When we create a video, we need to make sure that our category exists. We might be tempted to solve this problem by simply performing a query, but such an approach would be unsafe due to race conditions. In most cases, we would expect it to work like this:

1. The user sends a category ID through the form.
2. We perform a query to check if the category ID exists in the database.
3. If the category ID does exist in the database, we add the video with the category ID to the database.

However, someone could delete the category between steps 2 and 3, allowing us to ultimately insert a video without an existing category in the database. In any sufficiently busy application, that approach will lead to inconsistent data over time. Ecto has relentlessly pushed us to define references and indexes in our database because sometimes, doing a query won’t be enough and we’ll need to rely on database constraints.

Ecto will throw Ecto.ConstraintError if any of the db contraints fail. You can use unique_constraint, assoc_constraint etc inside the changesets to convert Ecto.ConstraintError into changeset errors. That way, your application does not crash on Ecto.ContraintErrors.

the *_constraint changeset functions are useful when the constraint being mapped is triggered by external data, often as part of the user request. Using changeset constraints only makes sense if the error message can be something the user can take action on.

Example:

When we added a foreign_key_constraint to the video belongs_to :category relationship, we knew we wanted to allow the user to choose the video category later on. If a category is removed at some point between the user loading the page and submitting the request to publish the video, setting the changeset constraint allows us to show a nice error message telling the user to pick something else.

This isn’t so uncommon. Maybe you’ve started to publish a new video on Friday at 5:00 p.m. but decide to finish the process next Monday. Someone has the whole weekend to remove a category, making your form data outdated.

On the other hand, let’s take the user has_many :videos relationship. Our application is the one responsible for setting up the relationship between videos and users. If a constraint is violated, it can only be a bug in our application or a data-integrity issue.

In such cases, the user can do nothing to fix the error, so crashing is the best option. Something unexpected really happened. But that’s OK. We know Elixir was designed to handle failures, and Phoenix allows us to convert them into nice status pages. Furthermore, we also recommend setting up a notification system that aggregates and emails errors coming from your application, so you can discover and act on potential bugs when your software is running in production.

## Use correct isolation in testing

For example, in your page_controller_test, we called our controller with get conn, "/" rather than calling the index action on our controller directly. This practice gives us the right level of isolation because we’re using the controller the same way Phoenix does.

## web/static/assets

We put everything in assets that doesn’t need to be transformed by Brunch. The build tool will simply copy those assets just as they are to priv/static, where they’ll be served by Phoenix.Static in our endpoint.

## web/static/js

Phoenix wraps the contents for each JavaScript file you add to web/static/js in a function and collects them into priv/static/js/app.js. That’s the file loaded by browsers at the end of web/templates/layout.html.eex when we call static_path(@conn, "/js/app.js").

Since each file is wrapped in a function, it won’t be automatically executed by browsers unless you explicitly import it in your app.js file. In this way, the app.js file is like a manifest. It’s where you import and wire up your JavaScript dependencies.

## Ecto Changesets

Because Ecto separates changesets from the definition of a given record, we can have a separate change policy for each type of change. We could easily add a JSON API that creates videos, including the slug field, for example.

Changesets can validate data—for example, the length or the format of a field—on the fly, but validations that depend on data integrity are left to the database in the shape of constraints.

## Channels

Whereas request/response interactions are stateless, conversations in a long-running process can be stateful. This means that for more-sophisticated user interactions like interactive pages or multiplayer games, you don’t have to work so hard to keep track of the conversation by using cookies, databases, or the like. Each call to a channel simply picks up where the last one left off.

## simple_one_for_one strategy

This strategy doesn’t start any children. Instead, it waits for us to explicitly ask it to start a child process. Example:

inside lib/rumbl.ex
```
children = [
  ...,
  supervisor(Rumbl.InfoSys.Supervisor, []) # adds the Info Sys supervisor to our main supervisor
]

Supervisor.start_link(children, opts) # starts up the all the workers and sub supervisors. The sub supervisors will in turn start up all their workers and sub supervisors
```

inside lib/rumbl/info_sys/supervisor.ex
```
def start_link do
  Supervisor.start_link(__MODULE__, [], name: __MODULE__) # ref-name-point
end

def init(_opts) do
  children = [
    worker(Rumbl.InfoSys, [], restart: :temporary)
  ]
  supervise children, strategy: :simple_one_for_one

  # Using the simple_one_for_one, means this supervisor will not start up any of its children, it's left to the programmer to manually start a child using the start_child function
end
```

inside lib/rumbl/info_sys.ex
```
defp spawn_query(backend, query, limit) do
  query_ref = make_ref()
  opts = [backend, query, query_ref, self(), limit]
  # We ask the Rumbl.InfoSys.Supervisor to start one child Rumbl.InfoSys process using the start_child function.
  # The first argument to start_child is just an atom. Modules in Elixir are just atoms, i.e. Rumbl.InfoSys.Supervisor
  # is just an atom. From ref-name-point above, you can see we use the module name as the supervisor name, hence when
  # using start_child, we have to use the same module name.
  # If instead ref-name-point is Supervisor.start_link(__MODULE__, [], name: :hello), then we would use start_child
  # as start_child(:hello, opts).
  # opts passed into the supervisor are already passed to the supervisor's worker processes when it initializes them.
  {:ok, pid} = Supervisor.start_child(Rumbl.InfoSys.Supervisor, opts)
  {pid, query_ref}
end
```

## Timeout of 0
```
def test do
  receive do
    :hello ->
      IO.puts "hello, world!" # will never be executed
  after
    0 ->
      IO.puts "kill ya!"
  end
end
```
If we use a after timeout of 0 seconds, no code in the receive blocks will ever be executed as it will jump straight into the after block when we enter the receive block.

## Tasks vs Agents

Tasks wrap behavior and agents encapsulate state.