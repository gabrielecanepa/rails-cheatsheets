# Authorization

See the [Pundit repository](https://github.com/varvet/pundit) for more information.

1. Add `gem "pundit"` to the Gemfile and run:

    ```sh
    bundle install
    rails generate pundit:install
    ```

2. Configure the `ApplicationController`:

    ```ruby
    # app/controllers/application_controller.rb

    class ApplicationController < ActionController::Base
      before_action :authenticate_user!
      include Pundit

      # White-list approach.
      after_action :verify_authorized, except: :index, unless: :skip_pundit?
      after_action :verify_policy_scoped, only: :index, unless: :skip_pundit?

      # Redirect to root if user is not authorized.
      rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

      def user_not_authorized
        flash[:alert] = "You are not authorized to perform this action."
        redirect_to(root_path)
      end

      private

      def skip_pundit?
        devise_controller? || params[:controller] =~ /(^(rails_)?admin)|(^pages$)/
      end
    end
    ```

3. Generate a policy for each model:

    ```sh
    rails generate pundit:policy restaurant
    ```

4. Implement the policy:

    ```ruby
    # app/policies/restaurant_policy.rb

    class RestaurantPolicy < ApplicationPolicy
      class Scope < Scope
        # Used as index? method
        def resolve
          # All restaurants
          scope.all
          # Only creator's restaurants
          # scope.where(user: user)
        end
      end

      def show?
        true
      end

      def create?
        true
      end

      def update?
        # Only restaurant creator
        record.user == user
        # record: the restaurant passed to the `authorize` method in controller
        # user: the `current_user` signed in with Devise
      end

      def destroy?
        # Only restaurant creator
        record.user == user
      end
    end
    ```

5. Add the `policy_scope` and `authorize` method to the controller actions:

    ```ruby
    # app/controllers/restaurants_controller.rb

    class Restaurants
      # [...]
      def index
        @restaurants = policy_scope(Restaurant)
      end

      def show
        @restaurant = Restaurant.find(params[:id])
        authorize @restaurant
      end

      def create
        @restaurant = Restaurant.new(restaurant_params)
        @restaurant.user = current_user
        authorize @restaurant
        # [...]
      end

      def update
        @restaurant = Restaurant.find(params[:id])
        authorize @restaurant
        # [...]
      end

      def destroy
        @restaurant = Restaurant.find(params[:id])
        authorize @restaurant
        # [...]
      end
    end
    ```

6. Use the `policy` helper in the views:

    ```erb
    <!-- app/views/restaurants/show.html.erb -->

    <% if policy(@restaurant).edit? %>
      <%= link_to "Edit this restaurant", edit_restaurant_path(@restaurant) %> |
    <% end %>
    <%= button_to "Destroy this restaurant", @restaurant, method: :delete if policy(@restaurant).destroy? %>
    <%= link_to "New restaurant", new_restaurant_path if policy(Restaurant).new? %>
    ```

7. Add admin users:

    ```sh
    rails g migration AddAdminToUsers admin:boolean
    rails db:migrate
    ```
  
    ```ruby
    # app/policies/restaurant_policy.rb

    # [...]
    class Scope < Scope
      def resolve
        user.admin? ? scope.all : scope.where(user: user)
      end
    end
    ```

    ```ruby
    # rails console
    User.first.update(admin: true)
    ```


