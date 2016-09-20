[![Gem Version](https://badge.fury.io/rb/grape-swagger.svg)](http://badge.fury.io/rb/grape-swagger)
[![Build Status](https://travis-ci.org/ruby-grape/grape-swagger.svg?branch=master)](https://travis-ci.org/ruby-grape/grape-swagger)
[![Dependency Status](https://gemnasium.com/ruby-grape/grape-swagger.svg)](https://gemnasium.com/ruby-grape/grape-swagger)
[![Code Climate](https://codeclimate.com/github/ruby-grape/grape-swagger.svg)](https://codeclimate.com/github/ruby-grape/grape-swagger)

##### Table of Contents

* [What is grape-swagger?](#what)
* [Related Projects](#related)
* [Compatibility](#version)
* [Swagger-Spec](#swagger-spec)
* [Installation](#install)
* [Usage](#usage)
* [Model Parsers](#model_parsers)
* [Configure](#configure)
* [Routes Configuration](#routes)
* [Securing the Swagger UI](#oauth)
* [Markdown](#md_usage)
* [Response documentation](#response)
* [Extensions](#extensions)
* [Example](#example)
* [Rake Tasks](#rake)

<a name="what" />
## What is grape-swagger?

The grape-swagger gem provides an autogenerated documentation for your [Grape](https://github.com/ruby-grape/grape) API. The generated documentation is Swagger-compliant, meaning it can easily be discovered in [Swagger UI](https://github.com/wordnik/swagger-ui). You should be able to point [the petstore demo](http://petstore.swagger.io/) to your API.

![Demo Screenshot](example/swagger-example.png)

These screenshot is based on the [Hussars](https://github.com/LeFnord/hussars) sample app.

<a name="related" />
## Related Projects

* [Grape](https://github.com/ruby-grape/grape)
* [Grape Swagger Entity](https://github.com/ruby-grape/grape-swagger-entity)
  * [Grape Entity](https://github.com/ruby-grape/grape-entity)
* [Grape Swagger Representable](https://github.com/ruby-grape/grape-swagger-representable)
* [Swagger UI](https://github.com/wordnik/swagger-ui)


<a name="version" />
## Compatibility

The following versions of grape, grape-entity and grape-swagger can currently be used together.

grape-swagger | swagger spec | grape                   | grape-entity | representable |
--------------|--------------|-------------------------|--------------|---------------|
0.10.5        |     1.2      | >= 0.10.0 ... <= 0.14.0 |  < 0.5.0     | n/a           |
0.11.0        |     1.2      |               >= 0.16.2 |  < 0.5.0     | n/a           |
0.20.1        |     2.0      | >= 0.12.0 ... <= 0.14.0 | <= 0.5.1     | n/a           |
0.20.3        |     2.0      | >= 0.12.0 ... ~> 0.16.2 | ~> 0.5.1     | n/a           |
0.21.0        |     2.0      | >= 0.12.0 ... <= 0.16.2 | <= 0.5.1     | >= 2.4.1      |
0.23.0        |     2.0      | >= 0.12.0 ... <= 0.17.0 | <= 0.5.1     | >= 2.4.1      |

<a name="swagger-spec" />
## Swagger-Spec

Grape-swagger generates documentation per [Swagger / OpenAPI Spec 2.0](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md).

<!-- validating: http://bigstickcarpet.com/swagger-parser/www/index.html -->

<a name="install" />
## Installation

Add to your Gemfile:

```ruby
gem 'grape-swagger'
```

## Upgrade

Please see [UPGRADING](UPGRADING.md) when upgrading from a previous version.


<a name="usage" />
## Usage

Mount all your different APIs (with ```Grape::API``` superclass) on a root node. In the root class definition, include ```add_swagger_documentation```, this sets up the system and registers the documentation on '/swagger_doc'. See [example/config.ru](example/config.ru) for a simple demo.


```ruby
require 'grape-swagger'

module API
  class Root < Grape::API
    format :json
    mount API::Cats
    mount API::Dogs
    mount API::Pirates
    add_swagger_documentation
  end
end
```

To explore your API, either download [Swagger UI](https://github.com/wordnik/swagger-ui) and set it up yourself or go to the [online swagger demo](http://petstore.swagger.wordnik.com/) and enter your localhost url documentation root in the url field (probably something in the line of http://localhost:3000/swagger_doc).


<a name="model_parsers" />
## Model Parsers

Since 0.21.0, `Grape::Entity` is not a part of grape-swagger, you need to add `grape-swagger-entity` manually to your Gemfile.
Also added support for [representable](https://github.com/apotonick/representable) via `grape-swagger-representable`.

```ruby
# For Grape::Entity ( https://github.com/ruby-grape/grape-entity )
gem 'grape-swagger-entity'
# For representable ( https://github.com/apotonick/representable )
gem 'grape-swagger-representable'
```

If you are not using Rails, make sure to load the parser inside your application initialization logic, e.g., via `require 'grape-swagger/entity'` or `require 'grape-swagger/representable`.

### Custom Model Parsers

You can create your own model parser, for example for [roar](https://github.com/apotonick/roar).

```rb
module GrapeSwagger
  module Roar
    class Parser
      attr_reader :model
      attr_reader :endpoint

      def initialize(model, endpoint)
        @model = model
        @endpoint = endpoint
      end

      def call
        # Parse your model and return hash with model schema for swagger
      end
    end
  end
end
```

Then you should register your custom parser.

```rb
GrapeSwagger.model_parsers.register(GrapeSwagger::Roar::Parser, Roar::Decorator)
```

To control model parsers sequence, you can insert your parser before or after another parser.

#### insert_before

```rb
GrapeSwagger.model_parsers.insert_before(GrapeSwagger::Representable::Parser, GrapeSwagger::Roar::Parser, Roar::Decorator)
```

#### insert_after

```rb
GrapeSwagger.model_parsers.insert_after(GrapeSwagger::Roar::Parser, GrapeSwagger::Representable::Parser, Representable::Decorator)
```

As we know, `Roar::Decorator` uses as superclass `Representable::Decorator`, this allow to avoid problem when Roar objects will be processed by `GrapeSwagger::Representable::Parser`, instead `GrapeSwagger::Roar::Parser`.


### CORS

If you use the online demo, make sure your API supports foreign requests by enabling CORS in Grape, otherwise you'll see the API description, but requests on the API won't return. Use [rack-cors](https://github.com/cyu/rack-cors) to enable CORS.

````ruby
require 'rack/cors'
use Rack::Cors do
  allow do
    origins '*'
    resource '*', headers: :any, methods: [ :get, :post, :put, :delete, :options ]
  end
end
```

Alternatively you can set CORS headers in a Grape `before` block.

```ruby
before do
  header['Access-Control-Allow-Origin'] = '*'
  header['Access-Control-Request-Method'] = '*'
end
````


<a name="configure" />
## Configure

* [host](#host)
* [base_path](#base_path)
* [mount_path](#mount_path)
* [add_base_path](#add_base_path)
* [add_version](#add_version)
* [doc_version](#doc_version)
* [markdown](#markdown)
* [endpoint_auth_wrapper](#endpoint_auth_wrapper)
* [swagger_endpoint_guard](#swagger_endpoint_guard)
* [oauth_token](#oauth_token)
* [security_definitions](#security_definitions)
* [models](#models)
* [hide_documentation_path](#hide_documentation_path)
* [info](#info)


You can pass a hash with optional configuration settings to ```add_swagger_documentation```.
The examples shows the default value.


The `host` and `base_path` options also accept a `proc` or `lambda` to evaluate, which is passed a [request](http://www.rubydoc.info/github/rack/rack/Rack/Request) object:

```ruby
add_swagger_documentation \
  base_path: proc { |request| request.host =~ /^example/ ? '/api-example' : '/api' }
```

<a name="host" />
#### host:
Sets explicit the `host`, default would be taken from `request`.
```ruby
add_swagger_documentation \
   host: 'www.example.com'
```

<a name="base_path" />
#### base_path:
Base path of the API that's being exposed, default would be taken from `request`.
```ruby
add_swagger_documentation \
   base_path: nil
```

`host` and `base_path` are also accepting a `proc` or `lambda`

<a name="mount_path" />
#### mount_path:
The path where the API documentation is loaded, default is: `/swagger_doc`.
```ruby
add_swagger_documentation \
   mount_path: '/swagger_doc'
```

#### add_base_path:
Add `basePath` key to the documented path keys, default is: `false`.
```ruby
add_swagger_documentation \
   add_base_path: true # only if base_path given
```

#### add_version:
Add `version` key to the documented path keys, default is: `true`,
here the version is the API version, specified by `grape` in [`path`](https://github.com/ruby-grape/grape/#path)

```ruby
add_swagger_documentation \
   add_version: true
```

<a name="doc_version" />
#### doc_version:
Specify the version of the documentation at [info section](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#info-object), default is: `'0.0.1'`
```ruby
add_swagger_documentation \
   doc_version: '0.0.1'
```

<a name="markdown" />
#### markdown:
Allow markdown in `detail`, default is `false`. (disabled) See [below](#md_usage) for details.

```ruby
add_swagger_documentation \
  markdown: GrapeSwagger::Markdown::KramdownAdapter.new
```
or alternative
```ruby
add_swagger_documentation \
  markdown: GrapeSwagger::Markdown::RedcarpetAdapter.new
```

<a name="endpoint_auth_wrapper" />
#### endpoint_auth_wrapper:
Specify the middleware to use for securing endpoints.

```ruby
add_swagger_documentation \
   endpoint_auth_wrapper: WineBouncer::OAuth2
```

<a name="swagger_endpoint_guard" />
#### swagger_endpoint_guard:
Specify the method and auth scopes, used by the middleware for securing endpoints.

```ruby
add_swagger_documentation \
   swagger_endpoint_guard: 'oauth2 false'
```

<a name="oauth_token" />
#### oauth_token:
Specify the method to get the oauth_token, provided by the middleware.

```ruby
add_swagger_documentation \
   oauth_token: 'doorkeeper_access_token'
```

<a name="security_definitions" />
#### security_definitions:
Specify the [Security Definitions Object](https://github.com/OAI/OpenAPI-Specification/blob/master/versions/2.0.md#security-definitions-object)

```ruby
add_swagger_documentation \
  security_definitions: {
    api_key: {
      type: "apiKey",
      name: "api_key",
      in: "header"
    }
  }
```

<a name="models" />
#### models:
A list of entities to document. Combine with the [grape-entity](https://github.com/ruby-grape/grape-entity) gem.

These would be added to the definitions section of the swagger file.

```ruby
add_swagger_documentation \
   models: [
     TheApi::Entities::UseResponse,
     TheApi::Entities::ApiError
   ]
```

<a name="hide_documentation_path" />
#### hide_documentation_path: (default: `true`)
```ruby
add_swagger_documentation \
   hide_documentation_path: true
```

Don't show the `/swagger_doc` path in the generated swagger documentation.

<a name="info" />
#### info:
```ruby
add_swagger_documentation \
  info: {
    title: "The API title to be displayed on the API homepage.",
    description: "A description of the API.",
    contact_name: "Contact name",
    contact_email: "Contact@email.com",
    contact_url: "Contact URL",
    license: "The name of the license.",
    license_url: "www.The-URL-of-the-license.org",
    terms_of_service_url: "www.The-URL-of-the-terms-and-service.com",
  }
```

A hash merged into the `info` key of the JSON documentation.

<!-- #### *authorizations*:
This value is added to the `authorizations` key in the JSON documentation. -->

<!-- #### *api_documentation*:
Customize the Swagger API documentation route, typically contains a `desc` field. The default description is "Swagger compatible API description".

```ruby
add_swagger_documentation \
   api_documentation: { desc: 'Reticulated splines API swagger-compatible documentation.' }
```


#### *specific_api_documentation*:
Customize the Swagger API specific documentation route, typically contains a `desc` field. The default description is "Swagger compatible API description for specific API".

```ruby
add_swagger_documentation \
   specific_api_documentation: { desc: 'Reticulated splines API swagger-compatible endpoint documentation.' }
``` -->


<a name="routes" />
## Routes Configuration

* [Swagger Header Parameters](#headers)
* [Hiding an Endpoint](#hiding)
* [Overriding Auto-Generated Nicknames](#overriding-auto-generated-nicknames)
* [Defining an endpoint as array](#array)
* [Using an options hash](#options)
* [Specify endpoint details](#details)
* [Overriding param type](#overriding-param-type)
* [Overriding type](#overriding-type)
* [Multi types](#multi-types)
* [Hiding parameters](#hiding-parameters)
* [Response documentation](#response)


<a name="headers" />
#### Swagger Header Parameters <a name="headers" />

Swagger also supports the documentation of parameters passed in the header. Since grape's ```params[]``` doesn't return header parameters we can specify header parameters seperately in a block after the description.

```ruby
desc "Return super-secret information", {
  headers: {
    "XAuthToken" => {
      description: "Valdates your identity",
      required: true
    },
    "XOptionalHeader" => {
      description: "Not really needed",
      required: false
    }
  }
}
```


<a name="hiding" />
#### Hiding an Endpoint

You can hide an endpoint by adding ```hidden: true``` in the description of the endpoint:

```ruby
desc 'Hide this endpoint', hidden: true
```

Endpoints can be conditionally hidden by providing a callable object such as a lambda which evaluates to the desired
state:

```ruby
desc 'Conditionally hide this endpoint', hidden: lambda { ENV['EXPERIMENTAL'] != 'true' }
```


#### Overriding Auto-Generated Nicknames

You can specify a swagger nickname to use instead of the auto generated name by adding `:nickname 'string'``` in the description of the endpoint.

```ruby
desc 'Get a full list of pets', nickname: 'getAllPets'
```


<a name="array" />
#### Defining an endpoint as array

You can define an endpoint as array by adding `is_array` in the description:

```ruby
desc 'Get a full list of pets', is_array: true
```


<a name="options" />
#### Using an options hash

The Grape DSL supports either an options hash or a restricted block to pass settings. Passing the `nickname`, `hidden` and `is_array` options together with response codes is only possible when passing an options hash.
Since the syntax differs you'll need to adjust it accordingly:

```ruby
desc 'Get all kittens!', {
  hidden: true,
  is_array: true,
  nickname: 'getKittens',
  entity: Entities::Kitten, # or success
  http_codes: [[401, 'KittenBitesError', Entities::BadKitten]] # or failure
  # also explicit as hash: [{ code: 401, mssage: 'KittenBitesError', model: Entities::BadKitten }]
  produces: [ "array", "of", "mime_types" ],
  consumes: [ "array", "of", "mime_types" ]
  }
get '/kittens' do
```


<a name="details" />
#### Specify endpoint details

To specify further details for an endpoint, use the `detail` option within a block passed to `desc`:

```ruby
desc 'Get all kittens!' do
  detail 'this will expose all the kittens'
end
get '/kittens' do
```


#### Overriding param type

You can override paramType in POST|PUT methods to query, using the documentation hash.

```ruby
params do
  requires :action, type: Symbol, values: [:PAUSE, :RESUME, :STOP], documentation: { param_type: 'query' }
end
post :act do
  ...
end
```

#### Overriding type

You can override type, using the documentation hash.

```ruby
params do
  requires :input, type: String, documentation: { type: 'integer' }
end
post :act do
  ...
end
```

```json
{
  "in": "formData",
  "name": "input",
  "type": "integer",
  "format": "int32",
  "required": true
}
```

#### Array type

Array types are also supported.

```ruby
params do
  requires :action_ids, type: Array[Integer]
end
post :act do
  ...
end
```

```json
{
  "in": "formData",
  "name": "action_ids",
  "type": "array",
  "items": {
      "type": "integer"
  },
  "required": true
}
```

#### Collection format of arrays

You can set the collection format of an array, using the documentation hash.

Collection format determines the format of the array if type array is used. Possible values are:
*  csv - comma separated values foo,bar.
*  ssv - space separated values foo bar.
*  tsv - tab separated values foo\tbar.
*  pipes - pipe separated values foo|bar.
*  multi - corresponds to multiple parameter instances instead of multiple values for a single instance foo=bar&foo=baz. This is valid only for parameters in "query" or "formData".

```ruby
params do
  requires :statuses, type: Array[String], documentation: { collectionFormat: 'multi' }
end
post :act do
  ...
end
```

```json
{
  "in": "formData",
  "name": "statuses",
  "type": "array",
  "items": {
      "type": "string"
  },
  "collectionFormat": "multi",
  "required": true
}
```

#### Multi types

By default when you set multi types, the first type is selected as swagger type

```ruby
params do
  requires :action, types: [String, Integer]
end
post :act do
  ...
end
```

```json
{
  "in": "formData",
  "name": "action",
  "type": "string",
  "required": true
}
```

#### Hiding parameters

Exclude single optional parameter from the documentation

```ruby
params do
  optional :one, documentation: { hidden: true }
  optional :two, documentation: { hidden: -> { true } }
end
post :act do
  ...
end
```

#### Overriding the route summary

By default, the route summary is filled with the value supplied to `desc`.

```ruby
namespace 'order' do
  desc 'This will be your summary'
  get :order_id do
    ...
  end
end
```

To override the summary, add `summary: '[string]'` after the description.

```ruby
namespace 'order' do
  desc 'This will be your summary',
    summary: 'Now this is your summary!'
  get :order_id do
    ...
  end
end
```


#### Expose nested namespace as standalone route

Use the `nested: false` property in the `swagger` option to make nested namespaces appear as standalone resources.
This option can help to structure and keep the swagger schema simple.

```ruby
namespace 'store/order', desc: 'Order operations within a store', swagger: { nested: false } do
  get :order_id do
  	...
  end
end
```

All routes that belong to this namespace (here: the `GET /order_id`) will then be assigned to the `store_order` route instead of the `store` resource route.

It is also possible to expose a namespace within another already exposed namespace:

```ruby
namespace 'store/order', desc: 'Order operations within a store', swagger: { nested: false } do
  get :order_id do
  	...
  end
  namespace 'actions', desc: 'Order actions' do, nested: false
    get 'evaluate' do
      ...
    end
  end
end
```
Here, the `GET /order_id` appears as operation of the `store_order` resource and the `GET /evaluate` as operation of the `store_orders_actions` route.


##### With a custom name

Auto generated names for the standalone version of complex nested resource do not have a nice look.
You can set a custom name with the `name` property inside the `swagger` option, but only if the namespace gets exposed as standalone route.
The name should not contain whitespaces or any other special characters due to further issues within swagger-ui.

```ruby
namespace 'store/order', desc: 'Order operations within a store', swagger: { nested: false, name: 'Store-orders' } do
  get :order_id do
  	...
  end
end
```


<a name="additions" />
## Additional documentation

* [Markdown in Detail](#md_usage)
* [Response documentation](#response)

### Setting a Swagger defaultValue

Grape allows for an additional documentation hash to be passed to a parameter.

```ruby
params do
  requires :id, type: Integer, desc: 'Coffee ID'
  requires :temperature, type: Integer, desc: 'Temperature of the coffee in celcius', documentation: { default: 72 }
end
```

The example parameter will populate the Swagger UI with the example value, and can be used for optional or required parameters.

Grape uses the option `default` to set a default value for optional parameters. This is different in that Grape will set your parameter to the provided default if the parameter is omitted, whereas the example value above will only set the value in the UI itself. This will set the Swagger `defaultValue` to the provided value. Note that the example value will override the Grape default value.

```ruby
params do
  requires :id, type: Integer, desc: 'Coffee ID'
  optional :temperature, type: Integer, desc: 'Temperature of the coffee in celcius', default: 72
end
```


### Grape Entities

Add the [grape-entity](https://github.com/agileanimal/grape-entity) gem to your Gemfile.

The following example exposes statuses. And exposes statuses documentation adding :type and :desc.

```ruby
module API
  module Entities
    class Status < Grape::Entity
      expose :text, documentation: { type: 'string', desc: 'Status update text.' }
      expose :links, using: Link, documentation: { type: 'link', is_array: true }
      expose :numbers, documentation: { type: 'integer', desc: 'favourite number', values: [1,2,3,4] }
    end

    class Link < Grape::Entity
      expose :href, documentation: { type: 'url' }
      expose :rel, documentation: { type: 'string'}
    end
  end

  class Statuses < Grape::API
    version 'v1'

    desc 'Statuses index',
      entity: API::Entities::Status
    get '/statuses' do
      statuses = Status.all
      type = current_user.admin? ? :full : :default
      present statuses, with: API::Entities::Status, type: type
    end

    desc 'Creates a new status',
      entity: API::Entities::Status,
      params: API::Entities::Status.documentation
    post '/statuses' do
        ...
    end
  end
end
```


#### Relationships

You may safely omit `type` from relationships, as it can be inferred. However, if you need to specify or override it, use the full name of the class leaving out any modules named `Entities` or `Entity`.


##### 1xN

```ruby
module API
  module Entities
    class Client < Grape::Entity
      expose :name, documentation: { type: 'string', desc: 'Name' }
      expose :addresses, using: Entities::Address,
        documentation: { type: 'API::Address', desc: 'Addresses.', param_type: 'body', is_array: true }
    end

    class Address < Grape::Entity
      expose :street, documentation: { type: 'string', desc: 'Street.' }
    end
  end

  class Clients < Grape::API
    version 'v1'

    desc 'Clients index', params: Entities::Client.documentation
    get '/clients' do
      ...
    end
  end

  add_swagger_documentation models: [Entities::Client, Entities::Address]
end
```


##### 1x1

Note: `is_array` is `false` by default.

```ruby
module API
  module Entities
    class Client < Grape::Entity
      expose :name, documentation: { type: 'string', desc: 'Name' }
      expose :address, using: Entities::Address,
        documentation: { type: 'API::Address', desc: 'Addresses.', param_type: 'body', is_array: false }
    end

    class Address < Grape::Entity
      expose :street, documentation: { type: 'string', desc: 'Street' }
    end
  end

  class Clients < Grape::API
    version 'v1'

    desc 'Clients index', params: Entities::Client.documentation
    get '/clients' do
      ...
    end
  end

  add_swagger_documentation models: [Entities::Client, Entities::Address]
end
```

<a name="oauth" />
## Securing the Swagger UI


The Swagger UI on Grape could be secured from unauthorized access using any middleware, which provides certain methods:

- a *before* method to be run in the Grape controller for authorization purpose;
- some guard method, which could receive as argument a string or an array of authorization scopes;
- a method which processes and returns the access token received in the HTTP request headers (usually in the 'HTTP_AUTHORIZATION' header).

Below are some examples of securing the Swagger UI on Grape installed along with Ruby on Rails:

- The WineBouncer and Doorkeeper gems are used in the examples;
- 'rails' and 'wine_bouncer' gems should be required prior to 'grape-swagger' in boot.rb;
- This works with a fresh PR to WineBouncer which is yet unmerged - [WineBouncer PR](https://github.com/antek-drzewiecki/wine_bouncer/pull/64).

This is how to configure the grape_swagger documentation:

```ruby
  add_swagger_documentation base_path: '/',
                            title: 'My API',
                            doc_version: '0.0.1',
                            hide_documentation_path: true,
                            hide_format: true,
                            endpoint_auth_wrapper: WineBouncer::OAuth2, # This is the middleware for securing the Swagger UI
                            swagger_endpoint_guard: 'oauth2 false',     # this is the guard method and scope
                            oauth_token: 'doorkeeper_access_token'      # This is the method returning the access_token
```

The guard method should inject the Security Requirement Object into the endpoint's route settings (see Grape::DSL::Settings.route_setting method).

The 'oauth2 false' added to swagger_documentation is making the main Swagger endpoint protected with OAuth, i.e. it
is retreiving the access_token from the HTTP request, but the 'false' scope is for skipping authorization and showing
 the UI for everyone. If the scope would be set to something else, like 'oauth2 admin', for example, than the UI
 wouldn't be displayed at all to unauthorized users.  

Further on, the guard could be used, where necessary, for endpoint access protection. Put it prior to the endpoint's method:

```ruby
  resource :users do
    oauth2 'read, write'
    get do
      render_users
    end

    oauth2 'admin'
    post do
      User.create!...
    end
  end
```

And, finally, if you want to not only restrict the access, but to completely hide the endpoint from unauthorized
users, you could pass a lambda to the :hidden key of a endpoint's description:

```ruby
  not_admins = lambda { |token=nil| token.nil? || !User.find(token.resource_owner_id).admin? }

  resource :users do
    desc 'Create user', hidden: not_admins
    oauth2 'admin'
    post do
      User.create!...
    end
  end
```

The lambda is checking whether the user is authenticated (if not, the token is nil by default), and has the admin
role - only admins can see this endpoint.

<a name="md_usage" />
## Markdown in Detail

The grape-swagger gem allows you to add an explanation in markdown in the detail field. Which would result in proper formatted markdown in Swagger UI.
Grape-swagger uses adapters for several markdown formatters. It includes adapters for [kramdown](http://kramdown.rubyforge.org) (kramdown [syntax](http://kramdown.rubyforge.org/syntax.html)) and [redcarpet](https://github.com/vmg/redcarpet).
The adapters are packed in the GrapeSwagger::Markdown modules. We do not include the markdown gems in our gemfile, so be sure to include or install the depended gems.

To use it, add a new instance of the adapter to the markdown options of `add_swagger_documentation`, such as:
```ruby
add_swagger_documentation \
  markdown: GrapeSwagger::Markdown::KramdownAdapter.new(options)
```
and write your route details in GFM, examples could be find in [details spec](blob/master/spec/swagger_v2/api_swagger_v2_detail_spec.rb)


#### Kramdown
If you want to use kramdown as markdown formatter, you need to add kramdown to your gemfile.

```ruby
gem 'kramdown'
```

Configure your api documentation route with:
```ruby
add_swagger_documentation \
  markdown: GrapeSwagger::Markdown::KramdownAdapter.new(options)
```


#### Redcarpet
As alternative you can use [redcarpet](https://github.com/vmg/redcarpet) as formatter, you need to include redcarpet in your gemspec. If you also want to use [rouge](https://github.com/jneen/rouge) as syntax highlighter you also need to include it.

```ruby
gem 'redcarpet'
gem 'rouge'
```

Configure your api documentation route with:

```ruby
add_swagger_documentation(
  markdown: GrapeSwagger::Markdown::RedcarpetAdapter.new(render_options: { highlighter: :rouge })
)
```

Alternatively you can disable rouge by adding `:none` as highlighter option. You can add redcarpet extensions and render options trough the `extenstions:` and `render_options:` parameters.


#### Custom markdown formatter

You can also add your custom adapter for your favourite markdown formatter, as long it responds to the method `markdown(text)` and it formats the given text.


```ruby
module API

  class FancyAdapter
   attr_reader :adapter

   def initialize(options)
    require 'superbmarkdownformatter'
    @adapter = SuperbMarkdownFormatter.new options
   end

   def markdown(text)
      @adapter.render_supreme(text)
   end
  end

  add_swagger_documentation markdown: FancyAdapter.new(no_links: true)
end
```


<a name="response" />
## Response documentation

You can also document the HTTP status codes with a description and a specified model, as ref in the schema to the definitions, that your API returns with one of the following syntax.

In the following cases, the schema ref would be taken from route.

```ruby
desc 'thing', http_codes: [ { code: 400, message: "Invalid parameter entry" } ]
get '/thing' do
  ...
end
```

```ruby
desc 'thing' do
  params Entities::Something.documentation
  http_codes [ { code: 400, message: "Invalid parameter entry" } ]
end
get '/thing' do
  ...
end
```

```ruby
get '/thing', http_codes: [
  { code: 200, message: 'Ok' },
  { code: 400, message: "Invalid parameter entry" }
] do
  ...
end
```

By adding a `model` key, e.g. this would be taken.
```ruby
get '/thing', http_codes: [
  { code: 200, message: 'Ok' },
  { code: 422, message: "Invalid parameter entry", model: Entities::ApiError }
] do
  ...
end
```
If no status code is defined [defaults](/lib/grape-swagger/endpoint.rb#L121) would be taken.

The result is then something like following:

```json
"responses": {
  "200": {
    "description": "get Horses",
    "schema": {
      "$ref": "#/definitions/Thing"
    }
  },
  "401": {
    "description": "HorsesOutError",
    "schema": {
      "$ref": "#/definitions/ApiError"
    }
  }
},
```

<a name="extensions" />
## Extensions

Swagger spec2.0 supports extensions on different levels, for the moment,
the documentation on `verb`, `path` and `definition` level would be supported.
The documented key would be generated from the `x` + `-` + key of the submitted hash,
for possibilities refer to the [extensions spec](spec/lib/extensions_spec.rb).
To get an overview *how* the extensions would be defined on grape level, see the following examples:

- `verb` extension, add a `x` key to the `desc` hash:
```ruby
desc 'This returns something with extension on verb level',
  x: { some: 'stuff' }
```
this would generate:
```json
"/path":{
  "get":{
    "…":"…",
    "x-some":"stuff"
  }
}
```

- `path` extension, by setting via route settings:
```ruby
route_setting :x_path, { some: 'stuff' }
```
this would generate:
```json
"/path":{
  "x-some":"stuff",
  "get":{
    "…":"…",
  }
}
```

- `definition` extension, again by setting via route settings,
here the status code must be provided, for which definition the extensions should be:
```ruby
route_setting :x_def, { for: 422, other: 'stuff' }
```
this would generate:
```json
"/definitions":{
  "ApiError":{
    "x-other":"stuff",
    "…":"…",
  }
}
```
or, for more definitions:
```ruby
route_setting :x_def, [{ for: 422, other: 'stuff' }, { for: 200, some: 'stuff' }]
```

<a="example" />
## Example

Go into example directory and run it: `$ bundle exec rackup`
go to: `http://localhost:9292/swagger_doc` to get it

For request examples load the [postman file]()

#### Grouping the API list using Namespace

Use namespace for grouping APIs

![grape-swagger-v2-new-corrected](https://cloud.githubusercontent.com/assets/1027590/13516020/979cfefa-e1f9-11e5-9624-f4a6b17a3c8a.png)

#### Example Code

```ruby
class NamespaceApi < Grape::API
  namespace :hudson do
    desc 'Document root'
    get '/' do
    end
  end

  namespace :hudson do
    desc 'This gets something.',
      notes: '_test_'

    get '/simple' do
      { bla: 'something' }
    end
  end

  namespace :colorado do
    desc 'This gets something for URL using - separator.',
      notes: '_test_'

    get '/simple-test' do
      { bla: 'something' }
    end
  end
end
  …

```


<a name="rake" />
## Rake Tasks

Add these lines to your Rakefile, and initialize the Task class with your Api class – be sure your Api class is available.

```ruby
require 'grape-swagger/rake/oapi_tasks'
GrapeSwagger::Rake::OapiTasks.new(::Api::Base)
```

#### OpenApi/Swagger Documentation

```
rake oapi:fetch
params:
- store={ true | file_name } – save as JSON (optional)
- resource=resource_name     – get only for this one (optional)
```

#### OpenApi/Swagger Validation

**requires**: `npm` and `swagger-cli` to be installed


```
rake oapi:validate
params:
- resource=resource_name – get only for this one (optional)
```


## Contributing to grape-swagger

See [CONTRIBUTING](CONTRIBUTING.md).

## Copyright and License

Copyright (c) 2012-2014 Tim Vandecasteele and contributors. See [LICENSE.txt](LICENSE.txt) for details.
