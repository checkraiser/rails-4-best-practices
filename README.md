# Models - Best Practices

## 1. Problems

Suppose we have to publish an item for approving with modified **published_on** field. Here's is our *normal* code:

*app/controllers/users_controller.rb*
```
class ItemController < ApplicationController
  def publish
    if @item.is_approved?
      @item.published_on = Time.now
      if @item.save
        flash[:notice] = "Your item published!"
      else
        flash[:notice] = "There was an error."
      end
      redirect_to @item
    else
      flash[:notice] = "There was an error."
    end
    redirect_to @item
  end
end
```

#### Problems with this code:

- Hard to understand.
- Business logic not encapsulated.
- More conflicts in code.
- New features hard to implement.

#### Add new feature get worse
Suppose we want to check if the item's user needs to have name 'testinguser'.
````
class ItemController < ApplicationController
  def publish
    if @item.is_approved? && @item.user = 'testinguser'
      @item.published_on = Time.now
      if @item.save
        flash[:notice] = "Your item published!"
      else
        flash[:notice] = "There was an error."
      end
      redirect_to @item
    else
      flash[:notice] = "There was an error."
    end
    redirect_to @item
  end
end
````

Code smell: Multiple calls to the same object

==> ***The principle here is TELL, DON'T ASK. You should tell objects what to do, and not ask them questions about their state.***


### 2. Solution:

#### Business logic should go in the models:

````
# app/controllers/item_controller.rb
class ItemsController < ApplicationController
  def publish
    if @item.publish
      flash[:notice] = "Your item published!"
    else
      flash[:notice] = "There was an error."
    end
      redirect_to @item
    end
    redirect_to @item
  end
end
````

````
# app/models/item.rb
class Item < ActiveRecord::Base
  belongs_to :user
  def publish
    if !is_approved? || user == 'Loblaw'
      return false
    end
    self.published_on = Time.now
    self.save
  end
end
````

==> ***Controllers should be responsible for orchestrating the Models.***



#### Avoid calling other domain objects from callbacks

Callbacks are methods that get called at certain moments of an object’s life cycle

*app/models/user.rb*

```

class User < ActiveRecord::Base
  before_create :set_token

  protected
  def set_token
    self.token = TokenGenerator.create(self)
  end
end
```

Code smell: You're referencing other models in a callback (**TokenGenerator**) ==> introduces tight coupling affects the object's database lifecycle.


Solution to this case is to **create custom methods**

*app/models/user.rb*

```
class User < ActiveRecord::Base
  def register
    self.token = TokenGenerator.create(self)
    save
  end
end
```

*app/controllers/users_controller.rb*

```
class UsersController < ApplicationController
  def create
    @user = User.new(user_params)
    if @user.register
      redirect_to @user, notice: 'Success'
    else
      ...
    end
  end
end
```

Callbacks should only be for modifying internal state

*app/models/user.rb*

```
class User < ActiveRecord::Base
  before_create :set_name
  protected
    def set_name
      self.name = self.login.capitalize if name.blank?
    end
end
```

It's a convention to set callback methods as protected.

#### Not all models need to be ActiveRecord

#### *Problem*


*app/controllers/users_controller.rb*

```
class UsersController < ApplicationController
  def suspend
    @user = User.find(params[:id])
    @user.suspend!
    redirect_to @user, notice: 'Successfully suspended.'
  end
end
```

This looks like  a clean API from the outside

Let's see what's in **user** model:

*app/models/user.rb*

```
class User < ActiveRecord::Base
  has_many :items
  has_many :reviews
  def suspend!
    self.class.transaction do #
      self.update!(is_approved: false)
      self.items.each { |item| item.update!(is_approved: false) }
      self.reviews.each { |review| review.update!(is_approved: false) }
    end
  end
end
```
*transactions ensure all enclosing
database operations are atomic*

**...but too much logic cluttered into a single method**

The worst case is when we add more logic businesses to the **User** class:


*app/models/user.rb*
```
class User < ActiveRecord::Base
  ...
  def suspend!
    self.class.transaction do
      disapprove_user!
      disapprove_items!
      disapprove_reviews!
    end
  end
  def disapprove_user!
  def disapprove_items!
  def disapprove_reviews!

end
```

***Code smell: ***

- Too much logic inside of a single class
breaks the Single Responsibility Principle.
- An object that controls too many other
objects in the system is an anti-pattern
known as a God Object


#### *Solution*

- Create new abstractions: Not everything a user needs to go into the User model

*app/models/user_suspension.rb*

```
class UserSuspension
  def initialize(user)
    @user = user
  end
  def create
    @user.class.transaction do
      disapprove_user!
      disapprove_items!
      disapprove_reviews!
    end
  end
  private
    def disapprove_user!
    def disapprove_items!
    def disapprove_reviews!
end
```
This class  is responsible for only one thing. It's a PORO (Plain Old Ruby Object), with regular instance methods.

Now we can use this abstraction from controllers:

*app/controllers/users_controller.rb*

```
class UsersController < ApplicationController
  def suspend
    @user = User.find(params[:id])
    suspension = UserSuspension.new(@user)
    suspension.create!
    redirect_to @user, notice: 'Successfully suspended.'
  end
end
```
Based from this, you can build other frequent (non-ActiveRecord) abstractions: UserRegistration, PlanSubscription, CreditCardChange, ShoppingCart,...

===> **Non ActiveRecord Models are classes which encapsulate unique business logic out of your ActiveRecord models**

