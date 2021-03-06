# MiniApivore

MiniApivore is an adaptation of apivore gem for mini-test instead of rspec. 

Original project: https://github.com/westfieldlabs/apivore

So base credits are for the apivore authors, this is 60% copy/paste of original project. 
Rem: didn't forked cause didn't expect it to be a relatively small changes.

Code-Test-Document, the idea of how things are need to be done: https://medium.com/@leshchuk/code-test-document-9b79921307a5

## What's new/different
* Swagger schema can be loaded from a file or directly. There can be one schema per MiniTestClass. 
* Removed all dependencies of active support and rails. See tests as an example on how 
  to use a mini-apivore outside a rails 
* Didn't implement a custom schema validator ( but I keeped an original schema and code from apivore in case of a future need )
* Test for untested routes now added by default at the end of all runnable_methods
* Much simplified tests against original project, rspec is replaced with minitest
* Removed all rspec dependencies and usage.

## Installation

Add this line to your application's Gemfile:

```ruby
gem 'mini-apivore'
```

And then execute:

    $ bundle

Or install it yourself as:

    $ gem install mini-apivore

## Usage

To start testing routes with mini_apivore you need: 

* ```require 'mini_apivore' ``` in you MiniTest class file
* ```include MiniApivore``` in you MiniTest class 
* ```init_swagger('/apidocs.json')``` init swagger-schema against which yourtest are gonna run
* Run ```check_route( :get, '/cards.json', OK )``` against all routes in your swagger schema

You can see example in test/mini_apivore/mini_apivore/api_schemas_test.rb

Here another complete example of testing simple internal REST api for Card model 
with devise integration as authentication framework

```ruby
#mini_apivore_helper.rb
require 'mini_apivore'

# this is simple intermediate class for apivore tes classes
class MiniApivoreTest < ActionDispatch::IntegrationTest
  include Devise::Test::IntegrationHelpers
  include MiniApivore

  # swagger checker initialized once per class, but since we using one definition
  # for all we can just redefine original swagger_checker
  def swagger_checker;
    SWAGGER_CHECKERS[MiniApivoreTest]
  end
  
end
```

```ruby
#cards_api_test.rb
require 'test_helper'
require 'mini_apivore_helper'

class CardsApiTest < MiniApivoreTest

  test 'cards unauthorized' do
    card = cards(:valid_card_1)
    check_route( :get, '/cards.json', NOT_AUTHORIZED )
    # check_route using to_param inside
    check_route( :get, '/cards/{id}.json', NOT_AUTHORIZED, id: card )
    check_route( :patch, '/cards/{id}.json', NOT_AUTHORIZED, id: card,
                 _data: { card: { title: '1' } } )
    check_route( :post, '/cards.json', NOT_AUTHORIZED,  _data: { card: { title: '1' } } )
    check_route( :delete, '/cards/{id}.json', NOT_AUTHORIZED, id: card )
  end

  test 'cards forbidden' do
    sign_in( users(:first_user) )
    # card with restricted privileges 
    card = cards(:restricted_card)

    check_route( :get, '/cards/{id}.json', FORBIDDEN, id: card )
    check_route( :patch, '/cards/{id}.json', FORBIDDEN, id: card,
                 _data: { card: { title: '1' } } )

    # this may be added if not all users can create cards 
    # check_route( :post, '/cards.json', FORBIDDEN,  _data: { card: { title: '1' } } )

    check_route( :delete, '/cards/{id}.json', FORBIDDEN, id: card )
  end


  test 'cards not_found' do
    sign_in( users(:first_user) )
    check_route( :get, '/cards/{id}.json', NOT_FOUND, id: -1 )
    check_route( :patch, '/cards/{id}.json', NOT_FOUND, id: -1 )
    check_route( :delete, '/cards/{id}.json', NOT_FOUND, id: -1 )
  end


  test 'cards REST authorized' do
    sign_in( users(:first_user) )
    check_route( :get, '/cards.json', OK )
    check_route( :get, '/cards/{id}.json', OK, id: cards(:valid_card_1) )
    

    assert_difference( -> { Card.count } ) do
      check_route( :post, '/cards.json', OK,
                   _data: {
                     card: { title: 'test card creation', 
                     card_preview_img_attributes: {
                                upload: fixture_file_upload( Rails.root.join('test', 'fixtures', 'files', 'test.png') ,'image/png')
                              }
                     } } )
    end
    assert( 'test card creation' == Card.last.title )

    check_route( :patch, '/cards/{id}.json', OK,
                 _data: { card: { title: 'Nothing' } }, id: Card.last )

    assert(Card.last.title == 'Nothing' )

    assert_difference( -> { Card.count }, -1 ) do
      check_route( :delete, '/cards/{id}.json', OK, id:  Card.last )
    end

  end
end
```

## Development

After checking out the repo, run `bin/setup` to install dependencies. Then, run `rake test` to run the tests. You can also run `bin/console` for an interactive prompt that will allow you to experiment.

To install this gem onto your local machine, run `bundle exec rake install`. To release a new version, update the version number in `version.rb`, and then run `bundle exec rake release`, which will create a git tag for the version, push git commits and tags, and push the `.gem` file to [rubygems.org](https://rubygems.org).

## Contributing

Bug reports and pull requests are welcome on GitHub at https://github.com/[USERNAME]/mini-apivore. This project is intended to be a safe, welcoming space for collaboration, and contributors are expected to adhere to the [Contributor Covenant](http://contributor-covenant.org) code of conduct.

## License

The gem is available as open source under the terms of the [MIT License](http://opensource.org/licenses/MIT).
