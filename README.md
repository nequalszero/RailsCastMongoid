## Summary
This is an experiment project testing the waters for using `MongoDB` with `Rails 4`, following along with this video from `RailsCasts`:

https://www.youtube.com/watch?v=L0RqU2MdqXU

There are numerous versioning issues that come up and the steps I took to resolve them are listed below.

## Steps
1. Started new project via `rails _4.2.7_ new MongoidRailsCast --skip-active-record`
2. Added `gem mongoid` to Gemfile, tutorial suggested `~> 3.0.2` which was most recent at that time; I left it off.
3. `rails g mongoid:config`
4. `rails g scaffold product name price:big_decimal`
5. manually added `released_on` property to `Product` model and allowed for extra parameter in `Products` controller, modified appropriate views.
6. Added `attr_accessible :name, :price, :released_on` and `validates_presence_of :name` to `Product` model.  Also added `protected_attributes` gem to the `Gemfile` and `include ActiveModel::MassAssignmentSecurity` to the `Product` model since `attr_accessible` was deprecated from Rails 4 or something.
7. Commented out `include ActiveModel:MassAssignmentSecurity` from `Product` model and added module to `config/application.rb`

  ```ruby
  module Mongoid
    module Document
      include ActiveModel::MassAssignmentSecurity
    end
  end
  ```
8. Sample queries
  ```ruby
    Product.find_by(price: 40)
    Product.where(price: 40) # finds all items matching criteria, returns array-like Mongoid::Criteria object similar to AR relation object
    Product.where(price: 40).first # finds first product matching criteria
    Product.where(:price.lte => 40) # finds products with price <= 40
    Product.lte(price: 40) # does the same as above
    Product.lte(price: 40).gt(released_on: 1.month.ago) # finds products cheaper than 40 released within the last month
  ```

9. `rails g model review content` to generate Reviews model
10. Add `embedded_in :product` to Review model and `embeds_many :reviews` in Product model.  Similar to `has_many` and `belongs_to` except rather than creating a new Reviews document, the information is stored under `reviews` in each Product entry.
11. Created review via `rails c`
  ```ruby
    p = Product.first
    p.reviews.create! content: "great game!"
    p.reviews.size # will be 1
    Review.count # will be 0
    Review.create! # will throw an exception
  ```
12. Add `Mongoid::Timestamps` to Product model to include `created_at` and `updated_at` attributes.
13. Add `Mongoid::Paranoia` to Product model to implement soft-deletes; this gives deleted entries to be restored.  Because Mongoid::Paranoia was deprecated at some point, it was extracted to its own gem `mongoid-paranoia`, and that gem stopped supporting Rails 4 in version 0.3.  Added this to Gemfile to make it work:
  ```ruby
  gem 'mongoid_paranoia', '0.2.1'
  ```
This essentially removes deleted entries from being queried, but allows them to be restored.
  ```ruby
    Product.count # => 4
    p = Product.last # => #<Product _id: 58d32926659c470b2d756e27, created_at: nil, updated_at: nil, deleted_at(deleted_at): nil, name: "Settlers of Catan", price: "34.99", released_on: 2012-01-01 00:00:00 UTC>
    p.destroy
    p # => #<Product _id: 58d32926659c470b2d756e27, created_at: nil, updated_at: nil, deleted_at(deleted_at): 2017-03-24 00:51:09 UTC, name: "Settlers of Catan", price: "34.99", released_on: 2012-01-01 00:00:00 UTC>
    Product.count # => 3
    p.restore
    p # => #<Product _id: 58d32926659c470b2d756e27, created_at: nil, updated_at: nil, deleted_at(deleted_at): nil, name: "Settlers of Catan", price: "34.99", released_on: 2012-01-01 00:00:00 UTC>
    Product.count # => 4
  ```
14. Add `field :_id, type: String, default: ->{ name.to_s.parameterize }` to Product model for more coherent URLs.  Currently show pages for products have URLs like:

  http://localhost:3000/products/58d32926659c470b2d756e27

  With this change, product SHOW pages have more descriptive URLs:

  http://localhost:3000/products/knights-of-catan

  A downside of this is that it requires additional validation to ensure that names are not blank, are unique, etc.  Changing the `name` of a product does not change its `_id`.
