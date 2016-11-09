# Building an API

## Designing an API

- When designing an API, it is important to map out the flow of data and to pay attention to RESTful practices.
- API design usually consists of mapping RESTful routes to specific actions for your users to take.
- A standard pen and paper will usually do, but there are tools out there to help you with this process. A popular one is [Apiary](https://apiary.io/).

## Implementing Functionality as an API

- Instead of the traditional intercept a request, issue a response in HTML, CSS and JS, the flow of API is more basic.
- A request is intercepted, processing happens on the server-side, and nothing but a JSON response comes back to the end user.
- Let's take a simple example of user records that are requested:

##### Request:

```
http://mysite.com/users
```

##### Response:

```javascript
[
	{
		firstname: "Bob",
		lastname: "Jones",
		age: 34,
		username: "bjones"
	},
	{
		firstname: "Katie",
		lastname: "Blount",
		age: 21,
		username: "kblount"
	}
]
```

##### Request:

```
http://mysite.com/users/1
```

##### Response:

```javascript
{
	firstname: "Bob",
	lastname: "Jones",
	age: 34,
	username: "bjones"
}
```

##### users_controller.rb

```ruby
def index
	render :json => User.all, status: 200
end

def show
	render :json => User.find(params[:id]), status: 200
end

def create
	user = User.create(user_params)
	
	if user.valid?
		head 201
	else
		head 400
	end
end
```

> Notice that each response (render) has a status code attached with it. This is very crucial to API design. A list of applicable status codes can be found [here](http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html).

## Implementing CORS

- Cross Origin Resource Sharing (CORS) is a set of restrictions that limit the ability of resources to be requested beyond the domain from which the resource originated.
- A more detailed explanation of the topic can be found here: https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS
- The bulk of API requests will be from another domain, so we have to have a way to lift this restriction with Rails.
- Essentially it boils down to configuring the server to send over a header named Access-Control-Allow-Origin that will allow specific domains.
- The Access-Control-Allow-Origin header can also be set to "*", which means all domains.
- An easy implementation of this is through a gem called [rack-cors](https://github.com/cyu/rack-cors).

## Namespaced Routes

- Namespacing is a technique to append a prefix to routes that meet a certain use case.
- This is commonly seen in building APIs, since your API functionality may differ from the rest of your application.
- Here is how we can create a namespace within Rails:

```ruby
namespace :api do
	resources :users
end
```

- Namespaces also make assumptions as to how the application is set up in terms of controllers.
- Namespaces will automatically look for a controller within the scope of your namespace:

```
GET /api/users(.:format) api/users#index
```

- As a result, it is important to change the way you create controllers with the Rails generator:

```
rails generate controller api/users
```

- This will give us:

```ruby
class Api::UsersController < ApplicationController
	# Your code here
end
```

## Rails as an API

- Rails does great as an API, but as you already know, wraps in a lot in the way of additional components meant for the browser.
- That includes functionality like views, assets, etc.
- There is a gem called [rails-api](https://github.com/rails-api/rails-api) that allows you to create a stripped-down project to be used as an api only.
- This gem is now a part of Rails core. You can read more about it here: http://edgeguides.rubyonrails.org/api_app.html

## CSRF with APIs

- As you remember, CSRF tokens help us to prevent forgery when dealing with forms.
- The problem though is that with an API there is no CSRF token. We can easily fix this in application_controller.rb:

```ruby
protect_from_forgery with: :null_session
```

## API Security

- There are times when you want a completely open API for the world to use, and there are times that you want to lock an API down to specific users.
- When you need security you have a few options. If you need to provide third-party access to user content, you can [implement your own OAuth strategy](https://github.com/doorkeeper-gem/doorkeeper).
- If you need to restrict access to users who are registered with an account for example you can use token-based authentication.
- We will be implementing token authentication for our user manager application using the built-in `authenticate_or_request_with_http_token` method.

##### application_controller.rb

```ruby
class ApplicationController < ActionController::Base
	protect_from_forgery with: :null_session

private

	def authenticate
		authenticate_or_request_with_http_token do |token, options|
			@auth_user = User.find_by(auth_token: token)
		end
	end
end
```

##### models/user.rb

```ruby
class User < ApplicationRecord
	before_create :set_auth_token

private

	def set_auth_token
		if auth_token.present?
			return
		else
			self.auth_token = generate_auth_token
		end
	end

	def generate_auth_token
		return SecureRandom.uuid.gsub(/\-/, '')
	end

end
```

##### users_controller.rb

```ruby
class UsersController < ApplicationController
	before_action :authenticate
	
	def index
		render :json => User.all, status: 200
	end
	
	def show
		render :json => User.find(params[:id]), status: 200
	end
	
	def create
		user = User.create(user_params)
		
		if user.valid?
			head 201
		else
			head 400
		end
	end
end
```

## Writing Tests for API Endpoints

- Testing API endpoints is made easy by RSpec.
- After following the [installation instructions for RSpec](https://github.com/rspec/rspec-rails) you will simply add a folder to /spec called "requests".
- Any specs that you place into this folder will have access to RSpec's built-in HTTP request methods.
- Let's take an example:

spec/requests/users_spec.rb

```ruby
describe "Testing Users API" do
	it "Should return a 200 status code" do
		get "/api/users"
		
		expect(response).to have_http_status(200)
	end
end
```

## Build an API Lab

- In this lab we will be taking a front end that is already developed, and build an API to power it using Rails.
- You will be working on the [Chirp! application located here](https://profstream.com/projects/26).
- The app front end only has two views - index and edit. You will have to add two views - signup and login.
- Your goal is to plan your API endpoints, document them, and then develop them with Rails using token authorization.
- A good idea is to write your request specs before creating the endpoints, and then test them with Postman.
- For this lab, use [Devise](https://github.com/plataformatec/devise) for authentication.
- Since you will be implementing login and signup with Devise through AJAX, a small tweak must be performed to get Devise to work with AJAX requests:

##### controllers/sessions_controller.rb

```ruby
class SessionsController < Devise::SessionsController  
	respond_to :json
end
```

##### controller/registrations_controller.rb

```ruby
class RegistrationsController < Devise::RegistrationsController
	respond_to :json

private

    def sign_up_params
        params.require(:user).permit(:firstname, :lastname, :email, :password, :password_confirmation, :username)
    end

    def account_update_params
        params.require(:user).permit(:firstname, :lastname, :email, :password, :password_confirmation, :current_password, :username)
    end

end
```

##### config/routes.rb

```ruby
devise_for :users, :controllers => { registrations: "registrations", sessions: "sessions" }
```