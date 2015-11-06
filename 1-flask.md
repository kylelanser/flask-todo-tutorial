The goal of this and subsequent articles will be to build our very own Todo app in EmberJS backed by the wonderful Flask Python-based web framework.  This is a living documents which has a [home on Github](https://github.com/drwlrsn/flask-todo-tutorial).  *I know there must be mistakes in here!* So please, if you find any (and you will) please leave a comment on Github or submit a pull request.  The [final source code](https://github.com/drwlrsn/flask-todo) is also on Github.

## Setup the Python environment

Install virtualenv because it's pretty neat.

<pre>$ sudo aptitude install python-virtualenv</pre>
or
<pre>$ sudo pip install virtualenv</pre>

I chose to use the package manager, aptitude, because pip was not installed on my vagrant virtual machine by default.

Now that we have python, virtualenv and pip installed let's set up a directory for our project.
<pre>
$ mkdir todo
$ cd todo
$ virtualenv env
New python executable in env/bin/python
Installing distribute…….[a lot of dots]..done.
$ source env/bin/activate
(env)$
</pre>

We created a directory called, 'todo' we moved into and then we had virtualenv create a virtual environment for our new Flask project. At the very end we activated this virtual environment so we can work within it.  Notice it changed your prompt to include (env).

<pre>
(env)$ pip install Flask
</pre>

And some downloading and unpacking you should get:

<pre>
Successfully installed Flask Jinja2 Werkzeug
Cleaning up…
</pre>

## Hello, World it's me Drew

We now have Flask and are ready to start the first bit of our app. I'm going to use the example provided in the Flask Quickstart documentation with a couple of changes to make our lives easier:

    from flask import Flask
    app = Flask(__name__)
    
    @app.route('/')
    def hello_world():
        return 'Hello World!'
    
    app.debug = True
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0')

The first change was setting Flask into debug mode with starts Flask with a code reloader. Any changes we make to the app will be loaded on the fly so we can see them right away. The second was changing the host so our Flask app will listen on all devices. You will need to do this if you are using a virtual machine like Vagrant.

Save this as todo.py in your app directory. Now you should be able to run it:

<pre>
(env)$ python todo.py
 * Running on http://0.0.0.0:5000/
 * Restarting with reloader
</pre>

Point your web browser at http://localhost:5000/ and you should see the fruits of your labour. To stop your Flask app, press ctrl+c.

That is pretty damn boring and cliché so let's move on really quick. If you require a more in depth explanation the Quickstart guide does a great job of explaining exactly what each line does.

## Install a bunch of Flask extensions

Since I want to use ember-data we need a data store. For this, we are going to use sqlalchemy. Not because SQL databases are the greatest thing ever, but because I don't want to set up another piece of software. That means we are going to use sqlite! Exciting!

Let's now install flask-sqlalchemy:

<pre>
(env)$ pip install flask-sqlalchemy
</pre>

### Models, flasks and databases

Now it's time to make some changes to our app to incorporate SQLAlchemy. We want a Todo object that has a unique ID, a title that could be anything and a state so we can strike something. So now the code looks like this:
    
    from flask import Flask
    import flask.ext.sqlalchemy
    
    app = Flask(__name__)

    # Create the Flask-SQLAlchemy object and an SQLite database
    app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///todo.db'
    db = flask.ext.sqlalchemy.SQLAlchemy(app)

    class Todo(db.Model):
        id = db.Column(db.Integer, primary_key=True)
        title = db.Column(db.Text, unique=False)
        is_completed = db.Column(db.Boolean)
   
        # When we create a new Todo it should be incomplete and have a title
        def __init__(self, title, is_completed=False):
            self.title = title
            self.is_completed = is_completed
    
    # Create the database tables.
    db.create_all()
    
    @app.route('/')
    def hello_world():
        return 'Hello World!'
    
    app.debug = True
    
    if __name__ == '__main__':
        app.run(host='0.0.0.0')'

Run that to make sure we haven't made any mistakes. After it successfully runs, quit and now let's play with it in the python interactive interpreter to make sure our Todo class makes sense and works as we imagined:

<pre>
(env)$ python -i
Python 2.6.6 (r266:84292, Dec 27 2010, 00:02:40)
[GCC 4.4.5] on linux2
Type “help”, “copyright”, “credits” or “license” for more information.
&gt;&gt;&gt; from todo import Todo, db
&gt;&gt;&gt; todo1 = Todo('Hey does this work')
&gt;&gt;&gt; print todo1.title
Hey does this work
&gt;&gt;&gt;print todo1.is_completed
False
&gt;&gt;&gt; print todo1.id
None
</pre>

Notice our Todo object doesn't have an id. That is only because we haven't added it to the database. This is where we are going to use the db object:

<pre>
&gt;&gt;&gt; db.session.add(todo1)
&gt;&gt;&gt; db.session.commit()
&gt;&gt;&gt; print todo1.id
1
</pre>

Hey that's pretty cool. We added our new Todo object to the database and now it has an id of 1. We can use this id to find it in the database:

<pre>
&gt;&gt;&gt; todo_db = Todo.query.get(1)
&gt;&gt;&gt; print todo_db.title
Hey does this work
</pre>

## Flask-Restless the hard way

Awesome. Look like we can now save new Todos in a database. Now that we have one Todo in the ol' database it's time to make our data available via rest. For this, we will use Flask-Restless. BUT and this a big but, we need a version of Flask-Restless at 0.10.0-dev (at the time of writing, this is the current version of master branch on Github) because of a [bug](https://github.com/jfinkels/flask-restless/issues/146).  So before going any further make sure pip is installing version >= 0.10.0 or clone the repo from Github and build it.

### Build from source

Make sure you have git installed! The following will work with Debian and Ubuntu and other Debian-based Linux distributions.  If you want to make this distribution independent, please help me via [Github](https://github.com/drwlrsn/flask-todo-tutorial)!

<pre>
(env)$ sudo aptitude install git
</pre>

<pre>
(env)$ git clone https://github.com/jfinkels/flask-restless.git
(env)$ cd flask-restless/
(env)$ python setup.py build
(env)$ python setup.py install
</pre>

Because we still have the virtualenv activated `python setup.py install` installs to the virtual environment we have setup.

### Install using pip

<pre>
(env)$ pip install Flask-Restless
</pre>

Now that we have Flask-Restless installed we can use it in our app.

	from flask import Flask
	import flask.ext.sqlalchemy
	import flask.ext.restless
	
	app = Flask(__name__)

	# Create the Flask-SQLAlchemy object and an SQLite database
	app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///todo.db'
	db = flask.ext.sqlalchemy.SQLAlchemy(app)
	
	class Todo(db.Model):
	    id = db.Column(db.Integer, primary_key=True)
	    title = db.Column(db.Text, unique=False)
	    is_completed = db.Column(db.Boolean)
	    
	    # When we create a new Todo it should be incomplete and have a title
	    def __init__(self, title, is_completed=False):
	        self.title = title
	        self.is_completed = is_completed
	
	# Create the database tables.
	db.create_all()
	
	# Create the Flask-Restless API manager.
	restless_manager = flask.ext.restless.APIManager(app, flask_sqlalchemy_db=db)
	
	# Create API endpoints, which will be available at /api/<tablename> by
	# default. Allowed HTTP methods can be specified as well.
	restless_manager.create_api(Todo, methods=['GET', 'POST', 'DELETE'])
	
	@app.route('/')
	def hello_world():
	    return 'Hello World!'
	
	app.debug = True
	
	
	if __name__ == '__main__':
	    app.run(host='0.0.0.0')

Now we can point our web browser to http://localhost:5000/api/todo we should get the following:

	{
	  "total_pages": 1, 
	  "objects": [ 
	    {
	      "is_completed": false, 
	      "id": 1, 
	      "title": "Hey does this work"
	    }
	  ], 
	  "num_results": 1, 
	  "page": 1
	}

### Sometimes Rubyists have too many opinions

Hey that's some sweet sweet JSON. Unfortunately that isn't the format EmberJS wants so we are going to create some [(pre and post) processors](https://flask-restless.readthedocs.org/en/latest/customizing.html#request-preprocessors-and-postprocessors) for Flask-Restless so we generate and accept JSON that EmberJS expects.

	import flask.ext.restless
	import flask.ext.sqlalchemy
	from flask import Flask, render_template
	
	
	app = Flask(__name__)
	
	
	# Create the Flask-SQLAlchemy object and an SQLite database
	app.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///todo.db'
	db = flask.ext.sqlalchemy.SQLAlchemy(app)
	
	
	class Todo(db.Model):
	    id = db.Column(db.Integer, primary_key=True)
	    title = db.Column(db.Text, unique=False)
	    is_completed = db.Column(db.Boolean)
	
	    # When we create a new Todo it should be incomplete and have a title
	    def __init__(self, title, is_completed=False):
	        self.title = title
	        self.is_completed = is_completed
	
	# Create the database tables.
	db.create_all()
	
	# Create the Flask-Restless API manager.
	restless_manager = flask.ext.restless.APIManager(app, flask_sqlalchemy_db=db)
	
	
	def ember_formatter(results):
	    return {'todos': results['objects']} if 'page' in results else results
	
	
	def pre_ember_formatter(results):
	    return results['todo']
	
	
	def pre_patch_ember_formatter(instid, results):
	    return results['todo']
	
	# Create API endpoint, which will be available at /api/todos
	restless_manager.create_api(
	    Todo,
	    methods=['GET', 'POST', 'DELETE', 'PUT', 'PATCH'],
	    url_prefix='/api',
	    collection_name='todos',
	    results_per_page=-1,
	    postprocessors={
	        'GET_MANY': [ember_formatter]
	    },
	    preprocessors={
	        'POST': [pre_ember_formatter],
	        'PUT_SINGLE': [pre_patch_ember_formatter]
	    }
	)
	
	app.debug = True
	
	
	@app.route('/')
	def index():
	    return render_template('index.html')
	
	if __name__ == '__main__':
	    app.run(host='0.0.0.0')

## Building ember-data… because RUBY

Next up is we need to setup ember-data so we have an interface between the EmberJS datastore and the API provided by our Flask app. First thing we need to do is create a directory structure for our templates and associated static files. There we are going to change directory back to our home directory and install Ruby so we can build ember-data.

<pre>
(env)$ mkdir -p static/js
(env)$ mkdir templates
(env)$ cd ~
(env)$ \curl -#L https://get.rvm.io | bash -s stable --ruby
(env)$ source /home/vagrant/.rvm/scripts/rvm
</pre>

Next is to clone and build ember-data so we can use it with our Flask-based Todo API

<pre>
(env)$ git clone https://github.com/emberjs/data.git ember-data
(env)$ cd ember-data/
(env)$ bundle && rake dist
</pre>

### Nothing is ever easy

If you are like me and forgot to install the bundler gem, you will get the error `ERROR: Gem bundler is not installed, run gem install bundler first.` This just means you have to install bundler first with the command `gem install bundler`. And now you might get another error:

<pre>
(env)$ gem install bundler
ERROR:  Loading command: install (LoadError)
	cannot load such file -- openssl
ERROR:  While executing gem ... (NoMethodError)
    undefined method `invoke_with_build_args' for nil:NilClass
</pre>

To fix this you need to install the openssl pkg for rvm with `rvm pkg install openssl`. Then I got another error (sweet Jesus):

<pre>
Fetching openssl-1.0.1c.tar.gz to /home/vagrant/.rvm/archives
######################################################################## 100.0%
Extracting openssl to /home/vagrant/.rvm/src/openssl-1.0.1c
Configuring openssl in /home/vagrant/.rvm/src/openssl-1.0.1c.
Compiling openssl in /home/vagrant/.rvm/src/openssl-1.0.1c.
Error running 'make', please read /home/vagrant/.rvm/log/openssl/make.log

Please note that it's required to reinstall all rubies:

    rvm reinstall all --force

Updating openssl certificates
Error running 'update_openssl_certs', please read /home/vagrant/.rvm/log/openssl.certs.log
</pre>

After looking at the make.log there were a bunch of zlib errors so it looks like I needed the zlib development library. It can be installed with `$ sudo aptitude install zlib1g` on Debian. After you can try reinstalling the openssl dvm package. I ran `$ rvm reinstall all --force` because it told me. This will take some time because it will reinstall/rebuild ruby and all of its stuff – like rubygems. After all of that compiling (probably) you can go ahead and install bundler with `gem install bundler`. And once bundler is installed we can build ember-data: `$ bundle && rake dist`. But after `rake dist` there is yet another error!

### The second time is not for me, but Javascript is

<pre>
Building Ember Data...
rake aborted!
Could not find a JavaScript runtime. See https://github.com/sstephenson/execjs for a list of available runtimes.
</pre>

So now we need to install a JavaScript runtime. I chose therubyracer because it uses Google's V8 and because science.

<pre>
(env)$ gem install therubyracer
</pre>

This did not fix the error so I had to append `gem therubyracer` to the Gemfile included with ember-data

<pre>
(env)$ echo "gem 'therubyracer'" >> Gemfile
</pre>

### The third time is charming

Now you should be able to `build dist` ember-data! You should now find ember-data.js, etc in dist/.

<pre>
(env)$ ls dist/
ember-data.js  ember-data.min.js  ember-data.prod.js  ember-spade.js  ember-tests.js  modules
</pre>

Next we can copy that minified version over so we can start using it with our project!

<pre>
(env)$ cp dist/ember-data.js ~/todo/static/js/
(env)$ cd ~/todo
</pre>

Now we need to get ember.js and handlebars.js so complete the dependencies.

<pre>
(env)$ cd static/js
(env)$ curl https://raw.github.com/emberjs/ember.js/release-builds/ember-1.0.0-rc.1.min.js -o ember.min.js
(env)$ curl https://raw.github.com/wycats/handlebars.js/1.0.0-rc.3/dist/handlebars.js -o handlebars.js
(env)$ cd ..
</pre>

## Using that fancy Javascript in a template

After we have setup now let's make an index.html in our templates directory. So far in our Jinja2 template we are just using [`url_for()`](http://flask.pocoo.org/docs/quickstart/#static-files) to get our static assets like CSS and all those javascript files we downloaded and created.

	<!doctype html>
	<html lang="en">
	    <head>
	        <meta charset="utf-8">
	        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
	        <title>ember.js • TodoMVC</title>
	        <link rel="stylesheet" href="{{ url_for('static', filename='css/base.css') }}">
	    </head>
	    <body>
	        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
	        <script src="{{ url_for('static', filename='js/handlebars.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/ember.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/ember-data.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/app.js') }}"></script>
	        </body>
	</html>



Next create `app.js` residing in `static/js` which will be super simple to make sure we didn't mess anything up.

	$(document).ready(function() {
	
    	window.App = Ember.Application.create();
	
	});

Pretty boring but you should see in your web browser's console:

<pre>
DEBUG: ------------------------------- 
DEBUG: Ember.VERSION : 1.0.0-rc.1
DEBUG: Handlebars.VERSION : 1.0.0-rc.3
DEBUG: jQuery.VERSION : 1.9.1
DEBUG: ------------------------------- 
</pre>

## The part where we stand on the shoulders of giants

This is the portion of the tutorial where I borrow some excellent code from the [TodoMVC](http://todomvc.com/) project. The goal of TodoMVC is to compare Javascript MVC frameworks using a standard app – in this case, a todo app. To begin, I am going to take [their code](https://github.com/addyosmani/todomvc/tree/gh-pages/architecture-examples/emberjs) for and EmberJS todo app and squeeze it into one file (which is not really a recommended way to organising an application). I'm doing this to make it easier to type out (or copy and paste) and also to help explain line-by-line what is going on. After the body, we are going to create a [Moustache](http://mustache.github.com/) template within our Jinja2 template. To accomplish this, we are going to use the `{% raw %}{% endraw %}` Tags so Moustache can work its magic.

	<html lang="en">
	    <head>
	        <meta charset="utf-8">
	        <meta http-equiv="X-UA-Compatible" content="IE=edge,chrome=1">
	        <title>ember.js • TodoMVC</title>
	        <link rel="stylesheet" href="{{ url_for('static', filename='css/base.css') }}">
	    </head>
	    <body>
	        {% raw %}
	        <script type="text/x-handlebars" data-template-name="todos">
	            <section id="todoapp">
	                <header id="header">
	                    <h1>todos</h1>
	                    {{view Ember.TextField id="new-todo" placeholder="What needs to be done?"
	                                 valueBinding="newTitle" action="createTodo"}}
	                </header>
	                {{#if length}}
	                    <section id="main">
	                        <ul id="todo-list">
	                            {{#each filteredTodos itemController="todo"}}
	                                <li {{bindAttr class="isCompleted:completed isEditing:editing"}}>
	                                    {{#if isEditing}}
	                                        {{view Todos.EditTodoView todoBinding="this"}}
	                                    {{else}}
	                                        {{view Ember.Checkbox checkedBinding="isCompleted" class="toggle"}}
	                                        <label {{action "editTodo" on="doubleClick"}}>{{title}}</label>
	                                        <button {{action "removeTodo"}} class="destroy"></button>
	                                    {{/if}}
	                                    </li>
	                            {{/each}}
	                        </ul>
	                        {{view Ember.Checkbox id="toggle-all" checkedBinding="allAreDone"}}
	                    </section>
	                    <footer id="footer">
	                        <span id="todo-count">{{{remainingFormatted}}}</span>
	                        <ul id="filters">
	                            <li>
	                                {{#linkTo todos.index activeClass="selected"}}All{{/linkTo}}
	                            </li>
	                            <li>
	                                {{#linkTo todos.active activeClass="selected"}}Active{{/linkTo}}
	                            </li>
	                            <li>
	                                {{#linkTo todos.completed activeClass="selected"}}Completed{{/linkTo}}
	                            </li>
	                        </ul>
	                        {{#if hasCompleted}}
	                            <button id="clear-completed" {{action "clearCompleted"}} {{bindAttr class="buttonClass:hidden"}}>
	                                Clear completed ({{completed}})
	                            </button>
	                        {{/if}}
	                    </footer>
	                {{/if}}
	            </section>
	            <footer id="info">
	                <p>Double-click to edit a todo</p>
	                <p>
	                    Created by
	                    <a href="http://github.com/tomdale">Tom Dale</a>,
	                    <a href="http://github.com/addyosmani">Addy Osmani</a>
	                </p>
	                <p>Part of <a href="http://todomvc.com">TodoMVC</a></p>
	            </footer>
	        </script>
	        {% endraw %}
	        <script src="//ajax.googleapis.com/ajax/libs/jquery/1.9.1/jquery.min.js"></script>
	        <script src="{{ url_for('static', filename='js/handlebars.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/ember.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/ember-data.js') }}"></script>
	        <script src="{{ url_for('static', filename='js/app.js') }}"></script>
	        </body>
	</html>

The first line after the `{% raw %}` tag: `<script type="text/x-handlebars" data-template-name="todos">` starts the Moustache template. EmberJS uses Handlebars and it can be pretty similar to Jinja2 templates at times. So the conditional statements should be pretty obvious.

The next snippet which bears explaining is:

	{{view Ember.TextField id="new-todo" placeholder="What needs to be done?"
	                                 valueBinding="newTitle" action="createTodo"}}

This is a built in view	 that creates an HTML `<input placeholder="What needs to be done" type=text>` that you would expect, but does a couple of other things. The first is that `action="createTodo"` will call a function that is defined in our Todos controller (which we will examine shortly) within that function we can access the value of the `Ember.TextField` using the [get](http://emberjs.com/api/classes/Ember.TextField.html#method_get) method which returns the value of `<input>`. 

Before going on any further, let's fill out our `app.js` so we are all on the same page and we can better examine how the template interacts with the todo app.

	'use strict';
	
	jQuery(document).ready(function($){
	     window.Todos = Ember.Application.create();
	
	    Todos.Store = DS.Store.extend({
	        revision: 12,
	        adapter: DS.RESTAdapter.create({
	            namespace: 'api'
	        })
	    });
	
	    Todos.Todo = DS.Model.extend({
	         title: DS.attr('string'),
	         isCompleted: DS.attr('boolean'),
	
	        todoDidChange: function () {
	            Ember.run.once(this, function () {
	                this.get('store').commit();
	            });
	        }.observes('isCompleted', 'title')
	     });
	
	    /* ------------------- Todos Views ------------------- */
	
	   Todos.TodoView = Ember.View.extend({
	        tagName: 'li',
	        classNameBindings: [
	            'todo.isCompleted:completed',
	            'isEditing:editing'
	        ],
	        doubleClick: function () {
	            this.set('isEditing', true);
	        }
	    });
	
	   Todos.EditTodoView = Ember.TextField.extend({
	        classNames: ['edit'],
	
	        valueBinding: 'todo.title',
	
	        change: function () {
	            var value = this.get('value');
	
	            if (Ember.isEmpty(value)) {
	                this.get('controller').removeTodo();
	            }
	        },
	
	        focusOut: function () {
	            this.set('controller.isEditing', false);
	        },
	
	        insertNewline: function () {
	            this.set('controller.isEditing', false);
	        },
	
	        didInsertElement: function () {
	            this.$().focus();
	        }
	    });
	
	/* ------------------- Todos Router ------------------- */
	
	    Todos.Router.map(function () {
	        this.resource('todos', { path: '/' }, function () {
	            this.route('active');
	            this.route('completed');
	        });
	    });
	
	    Todos.TodosRoute = Ember.Route.extend({
	        model: function () {
	            return Todos.Todo.find();
	        }
	    });
	
	    Todos.TodosIndexRoute = Ember.Route.extend({
	        setupController: function () {
	            var todos = Todos.Todo.find();
	            this.controllerFor('todos').set('filteredTodos', todos);
	        }
	    });
	
	    Todos.TodosActiveRoute = Ember.Route.extend({
	        setupController: function () {
	            var todos = Todos.Todo.filter(function (todo) {
	                if (!todo.get('isCompleted')) {
	                    return true;
	                }
	            });
	
	            this.controllerFor('todos').set('filteredTodos', todos);
	        }
	    });
	
	    Todos.TodosCompletedRoute = Ember.Route.extend({
	        setupController: function () {
	            var todos = Todos.Todo.filter(function (todo) {
	                if (todo.get('isCompleted')) {
	                    return true;
	                }
	            });
	
	            this.controllerFor('todos').set('filteredTodos', todos);
	        }
	    });
	
	/* ------------------- Todos Controller ------------------- */
	    Todos.TodosController = Ember.ArrayController.extend({
	        createTodo: function () {
	            // Get the todo title set by the "New Todo" text field
	            var title = this.get('newTitle');
	            if (!title.trim()) {
	                return;
	            }
	
	            // Create the new Todo model
	            Todos.Todo.createRecord({
	                title: title,
	                isCompleted: false
	            });
	
	            // Clear the "New Todo" text field
	            this.set('newTitle', '');
	
	            // Save the new model
	            this.get('store').commit();
	        },
	
	        clearCompleted: function () {
	            var completed = this.filterProperty('isCompleted', true);
	            completed.invoke('deleteRecord');
	
	            this.get('store').commit();
	        },
	
	        remaining: function () {
	            return this.filterProperty('isCompleted', false).get('length');
	        }.property('@each.isCompleted'),
	
	        remainingFormatted: function () {
	            var remaining = this.get('remaining');
	            var plural = remaining === 1 ? 'item' : 'items';
	            return '<strong>%@</strong> %@ left'.fmt(remaining, plural);
	        }.property('remaining'),
	
	        completed: function () {
	            return this.filterProperty('isCompleted', true).get('length');
	        }.property('@each.isCompleted'),
	
	        hasCompleted: function () {
	            return this.get('completed') > 0;
	        }.property('completed'),
	
	        allAreDone: function (key, value) {
	            if (value !== undefined) {
	                this.setEach('isCompleted', value);
	                return value;
	            } else {
	                return !!this.get('length') &&
	                    this.everyProperty('isCompleted', true);
	            }
	        }.property('@each.isCompleted')
	    });
	
	    Todos.TodoController = Ember.ObjectController.extend({
	        isEditing: false,
	
	        editTodo: function () {
	            this.set('isEditing', true);
	        },
	
	        removeTodo: function () {
	            var todo = this.get('model');
	
	            todo.deleteRecord();
	            todo.get('store').commit();
	        }
	    });
	
	});
	
## Conclusion

This tutorial is now getting pretty damn long and maybe boring. So I'm going to leave it there. In the next instalment we'll examine line-by-line what is exactly happening with our EmberJS app and how we can add a couple more features to get little more comfortable with the Javascript portion. Leaving it here should at least give you some positive pay-off and inspire some exploration with EmberJS.

** [Part 2](http://b.drwlrsn.com/post/47056659519/flask-todo-emberjs-pt2) ** is now here. Explaining how it all fits together.

**NB I probably messed up something.** Please correct me. Comments and pull requests are welcome at the [Github home for this document](https://github.com/drwlrsn/flask-todo-tutorial).

[Source code for this tutorial](https://github.com/drwlrsn/flask-todo) also can be found on Github. I probably messed something up there too so comments and pull requests welcome!
