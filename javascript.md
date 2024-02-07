# JavaScript

- [Stimulus](#stimulus)
- [External libraries](#external-libraries)
- [Asynchronous JavaScript](#asynchronous-javascript)

See the [JavaScript in Rails](https://guides.rubyonrails.org/working_with_javascript_in_rails.html) guide for more information.

## Stimulus

Generate Stimulus controllers with:

```sh
rails generate stimulus CONTROLLER_NAME
```

## External libraries

```sh
importmap pin $NAME_OF_THE_LIBRARY
```

For example, to use [flatpickr.js](https://flatpickr.js.org):

1. Install the library and import the stylesheet in your main layout:
  
    ```sh
    importmap pin flatpickr
    ```

    ```erb
    <!-- app/views/layouts/application.html.erb -->

    <!-- ... -->
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/flatpickr/dist/flatpickr.min.css">
    ```

2. Create a specific `datepicker` controller:

    ```sh
    rails generate stimulus datepicker
    ```

    ```js
    // app/javascript/controllers/datepicker_controller.js

    import { Controller } from '@hotwired/stimulus'
    import flatpickr from 'flatpickr';

    export default class extends Controller {
      connect() {
        flatpickr(this.element)
      }
    }
    ```

3. Add the controller to the view:

    ```erb
    <%= f.input :opening_date,
        as: :string,
        input_html: { 
          data: { 
            controller: "datepicker" 
          } 
        }
    %>
    ```

## Asynchronous JavaScript

1. Set up the controller to return a JSON response using `respond_to`:

    ```rb
    # app/controllers/monuments_controller.rb

    # [...]
    def create
      @monument = Monument.new(monument_params)

      respond_to do |format|
        if @monument.save
          format.html { redirect_to monument_path(@monument) }
          format.json # looks for a create.json view
        else
          format.html { render "monuments/new", status: :unprocessable_entity }
          format.json # looks for a create.json view
        end
      end
    end
    ```

2. Make sure that the `jbuilder` gem is installed and create a `create.json.jbuilder` view:

    ```rb
    # app/views/monuments/create.json.jbuilder

    if @monument.persisted?
      json.form render(partial: "monuments/form", formats: :html, locals: { monument: Monument.new })
      json.inserted_item render(partial: "monuments/monument", formats: :html, locals: { monument: @monument })
    else
      json.form render(partial: "monuments/form", formats: :html, locals: { monument: @monument })
    end
    ```

3. Generate a new Stimulus controller to handle the request:
  
    ```sh
    rails generate stimulus insert_in_list
    ```

    ```js
    // app/javascript/controllers/insert_in_list_controller.js

    import { Controller } from '@hotwired/stimulus'

    export default class extends Controller {
      static targets = ['items', 'form']

      send(event) {
        event.preventDefault();

        fetch(this.formTarget.action, {
          method: 'POST',
          headers: { Accept: 'application/json' },
          body: new FormData(this.formTarget)
        })
          .then(response => response.json())
          .then((data) => {
            if (data.inserted_item) {
              this.itemsTarget.insertAdjacentHTML('beforeend', data.inserted_item)
            }
            this.formTarget.outerHTML = data.form
          })
      }
    }
    ```

4. Connect the controller to the DOM:

    ```erb
    <!-- app/views/monument/index.html.erb -->

    <div class="container">
      <div class="row">
        <div class="col" data-controller="insert-in-list">
          <div id="monuments" data-insert-in-list-target="items">
            <!-- ... -->
          </div>
        </div>
      </div>
    </div>
    ```

    ```erb
    <!-- app/views/monument/_form.html.erb -->

    <%= simple_form_for monument,
        data: {
          insert_in_list_target: "form",
          action: "submit->insert-in-list#send"
        } do |f| %>
      <!-- ... -->
    <% end %>
    ```
