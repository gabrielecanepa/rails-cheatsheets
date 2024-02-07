# Authentication

See the [Devise repository](https://github.com/heartcombo/devise) for more information.

1. Add `gem "devise"` to the Gemfile and run:

    ```sh
    bundle install
    rails generate devise:install
    ```

    Then follow the instructions displayed in the terminal.

2. Generate a devise model with `rails generate devise User`.

3. Protect every route by default:

    ```ruby
    # app/controllers/application_controller.rb

    class ApplicationController < ActionController::Base
      before_action :authenticate_user!
    end
    ```

4. Skip the login for some pages:

    ```ruby
    # app/controllers/pages_controller.rb

    class PagesController < ApplicationController
      skip_before_action :authenticate_user!, only: :home

      def home
      end
    end
    ```

5. When adding new attributes to a devise model, specify them in the `ApplicationController`:

    ```ruby
    # app/controllers/application_controller.rb

    class ApplicationController < ActionController::Base
      # [...]
      before_action :configure_permitted_parameters, if: :devise_controller?

      def configure_permitted_parameters
        # For additional fields in app/views/devise/registrations/new.html.erb
        devise_parameter_sanitizer.permit(:sign_up, keys: [:first_name, :last_name])

        # For additional in app/views/devise/registrations/edit.html.erb
        devise_parameter_sanitizer.permit(:account_update, keys: [:first_name, :last_name])
      end
    end
    ```

6. Optionally [customize devise routes](https://gist.github.com/JamesChevalier/4703255).
