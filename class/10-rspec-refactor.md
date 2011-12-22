Refactoring with RSpec
======================

[RSpec](http://relishapp.com/rspec) is a Behaviour-Driven Development tool for Ruby programmers. BDD is an approach to software development that combines Test-Driven Development, Domain Driven Design, and Acceptance Test-Driven Planning. RSpec helps you do the TDD part of that equation, focusing on the documentation and design aspects of TDD.

RSpec lets you focus on testing vs. theorizing about testing.

Gemfile
-------

Gems from the test group aren't installed in production.

    group :test do
      gem "rspec", "~> 2.7"
      gem "rspec-rails", "~> 2.7"
      gem "rspec-core", "~> 2.7"
      gem "rspec-expectations", "~> 2.7"
      gem "rspec-mocks", "~> 2.7"
    end

You can also have a `:development` group for your favorite tools.

Spec Helper
-----------

A helper can be included with every spec. It defines RSpec behavior, such as what to do before every test or after the test suite finished.

    require 'rubygems'

    ENV["RAILS_ENV"] ||= 'test'

    require "rails/application"
    require File.expand_path("../../config/environment", __FILE__)
    require 'rspec/rails'

    RSpec.configure do |config|
      config.mock_with :rspec
      config.expect_with :rspec
      config.after(:all) do
        p "All tests finished."
      end
    end

A Model Spec
------------

A simple spec can execute model validations and ensure they perform as expected. Create `spec/models/thing_spec.rb`.

    require 'spec_helper'

    describe Thing do
      it "can be created with a name" do
        Thing.new({name: "thing"}).should be_valid
      end
      it "cannot be created without a name" do
        Thing.new.should_not be_valid
      end
    end

Mock Objects vs. Real Data
--------------------------

Sometimes we'll want to use a real database for our testing instead of relying on mock objects. This means we need a clean copy of the database for every test. There's a gem for that.

    gem "database_cleaner"

We don't need to re-create a schema every time, just truncate tables.

    RSpec.configure do |config|
      config.before(:suite) do
        DatabaseCleaner.strategy = :truncation
      end
      config.before(:each) do
        DatabaseCleaner.clean
      end
    end

Ensure that you have a schema for tests.

    rake db:test:prepare

Run tests.

    rspec spec

Exercise
--------

Add [fuubar](https://github.com/jeffkreeftmeijer/fuubar) progress bar.

Exercise
--------

You must have noticed that Rails takes a while to load. Rails is a large codebase that is interpreted every time you run `rspec`. Add [spork](https://github.com/sporkrb/spork) to the `:test` group to avoid reloading Rails every time you run tests and [rails-dev-boost](https://github.com/thedarkone/rails-dev-boost) to the `:development` group that brings an optimization from a future version of Rails that prevents reloading unchanged models when editing code behind a running Rails server.

Controller Specs w/ Stubs
-------------------------

The implementation of `Thing.method` may be non-trivial, a mock replaces it with a simple return of a value. An RSpec `mock_model` also defines a unique model identity, methods such as `:new_record?`, etc. The first parameter to `mock_model` is a model class, and the second one is a hash of stubs.

We can add a helper that creates a mock *Thing* the first time we need it. 

    def mock_thing(stubs={})
      @mock_thing ||= mock_model(Thing, stubs).as_null_object
    end

For the `:index` method we want to ensure that `@things` is assigned the collection of things. Because we're working with a mock object we have to stub `Thing.all`.

    describe "GET index" do
      it "assigns all things to @things" do
        Thing.stub!(:all).and_return [ mock_thing ]
        get :index
        assigns(:things).should eq [ mock_thing ]
      end
    end

Exercise
--------

Implement specs for the remaining methods.

* GET new
* GET edit
* POST create
* PUT update
* DELETE destroy

Controller Specs w/ Fabricators
-------------------------------

Mock objects work well for trivial models. Many Rubyists prefer to use real objects that are written to the database in tests, especially for higher level integration tests. Manufacturing these objects can be done by calling `Thing.create!`, but it doesn't provide good defaults or unique values. A *fabricator* from a gem called [fabrication](https://github.com/paulelliott/fabrication) can do that.

    gem "fabrication"

Add a fabricator for *Thing* in `spec/fabricators/thing_fabricator.rb`.

    Fabricator(:thing) do
      name { Fabricate.sequence(:name) { |i| "Thing Number #{i}" } }
    end

We can now use a real *Thing* for testing.

    describe "PUT update" do
      before(:each) do
        @thing = Fabricate(:thing)
      end
      it "updates thing" do
        put :update, :id => @thing.id.to_s, :thing => { 'name' => 'updated' }
        @thing.reload.name.should == 'updated'
      end
    end

Exercise
--------

Implement the remainder of the Thing controller tests with a Fabricator.

View Specs
----------

Controllers are not executed in a normal view spec. You must assign variables explicitly. The following spec is `spec/views/things/index.html.haml_spec.rb`.

    describe "things/index.html.haml" do
      before(:each) do
        @thing = Fabricate(:thing)
        assign(:things, Thing.all)
      end
      it "renders a list of things" do
        render
        assert_select "tr>td", :text => @thing.name, :count => 1
      end
    end

Acceptance Tests
----------------

While we can test individual components, we also want to ensure that the entire *Thing* feature is working. This can be done with an acceptance test that that is going to execute real user actions with a browser with [capybara](https://github.com/jnicklas/capybara) and [selenium](http://seleniumhq.org/).

    require 'spec_helper'

    feature "Things", :driver => :selenium do
      scenario "are displayed in a table" do
        thing = Fabricate(:thing)
        visit "/things"
        page.should have_css "td", text: thing.name
      end
      scenario "can be destroyed" do
        thing = Fabricate(:thing)
        visit "/things"
        page.evaluate_script('window.confirm = function() { return true; }')
        click_link "Destroy"
        Thing.count.should == 0
      end
    end

Refactor in Confidence
----------------------

Controllers support filters that avoid copy-pasting. This is called *DRYing* a controller. *DRY* stands for *Don't Repeat Yourself*. Now that we have specs that cover the application, we can refactor in confidence. 

    class ThingsController < ApplicationController
      before_filter :get_thing, :only => [ :edit, :show, :update, :destroy ]
      def get_thing
        @thing = Thing.find(params[:id])
      end
    end

[Other filters](http://rails.rubyonrails.org/classes/ActionController/Filters/ClassMethods.html) include `:after_filter`, `:around_filter`, etc.

Exercise
--------

Modify the application to redirect to a *404 Not Found* page when a user requests a *Thing* that doesn't exist. Exercise your TDD - write two failing controller and acceptance tests first.
