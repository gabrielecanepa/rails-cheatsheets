# Geocoding

- [Geocoder](#geocoder)
- [Mapbox](#mapbox)
  - [Display a map](#display-a-map)
  - [Search on a map](#search-on-a-map)
  - [Autocomplete address](#autocomplete-address)

## Geocoder

1. Install and configure the [geocoder gem](https://github.com/alexreisner/geocoder):

    ```sh
    # Gemfile

    # ...
    gem "geocoder"
    ```

    ```sh
    bundle install
    rails generate geocoder:config
    ```

    ```rb
    # config/initializers/geocoder.rb

    Geocoder.configure(
      # [...]
      units: :km, # Defaults to miles (:mi)
    )
    ```

2. Add the coordinates columns and update the model:

    ```sh
    rails g migration AddCoordinatesToFlats latitude:float longitude:float
    rails db:migrate
    ```

    ```rb
    # app/models/flat.rb

    class Flat < ApplicationRecord
      geocoded_by :address
      after_validation :geocode, if: :will_save_change_to_address?
    end
    ```

3. Create a geocoded instance and search for nearby instances:

    ```rb
    # rails console
    
    Flat.create(address: "16 Villa Gaudelet, Paris", name: "Le Wagon HQ")
    # => #<Flat:0x007fad2e720898
    # id: 1,
    # name: "Le Wagon HQ",
    # address: "16 Villa Gaudelet, Paris",
    # latitude: 48.8649574,
    # longitude: 2.3800617>

    Flat.near("Tour Eiffel", 10)   # flats within 10 km of Tour Eiffel
    Flat.near([40.71, 100.23], 20) # flats within 20 km of a point
    ```

## Mapbox

### Display a map

See the [Mapbox GL JS](https://docs.mapbox.com/mapbox-gl-js/api) documentation for more details.

1. Add an access token to your `.env` file:

    ```sh
    # .env

    MAPBOX_API_KEY=pk.eyJ1IjoicGR1b****************yZvNpTR_kk1kKqQ
    ```

2. Install [mapbox-gl](https://npmjs.com/package/mapbox-gl):

    ```sh
    importmap pin mapbox-gl
    ```

    ```erb
    <!-- app/views/layouts/application.html.erb -->

    <!-- ... -->
    <link href="https://api.mapbox.com/mapbox-gl-js/v2.11.0/mapbox-gl.css" rel="stylesheet">
    ```

3. Add markup and style for map, markers and info windows:

    ```erb
    <!-- app/views/flats/index.html.erb -->

    <!-- [...] -->
    <div 
      style="width: 100%; height: 600px;"
      data-controller="map"
      data-map-markers-value="<%= @markers.to_json %>"
      data-map-api-key-value="<%= ENV['MAPBOX_API_KEY'] %>"
    >
    </div>
    ```

    ```erb
    <!-- app/views/flats/_info_window.html.erb -->

    <h2><%= flat.name %></h2>
    <p><%= flat.address %></p>
    ```

    ```erb
    <!-- app/views/flats/_marker.html.erb -->

    <%= image_tag "logo.png", height: 30, width: 30, alt: "Logo" %>
    <!-- Or display some data -->
    <div class="badge rounded-pill bg-primary fs-6">
      <%= flat.name %>
    </div>
    ```

    ```scss
    // app/assets/stylesheets/components/_map.scss

    .mapboxgl-popup {
      max-width: 200px;
    }

    .mapboxgl-popup-content {
      text-align: center;
      font-family: "Open Sans", sans-serif;
    }
    ```

4. Define the markers in the Rails controller:

    ```rb
    # app/controllers/flats_controller.rb

    # [...]
    def index
      @flats = Flat.all
      # The `geocoded` scope filters only flats with coordinates
      @markers = @flats.geocoded.map do |flat|
        {
          lat: flat.latitude,
          lng: flat.longitude,
          info_window_html: render_to_string(partial: "info_window", locals: { flat: flat }),
          marker_html: render_to_string(partial: "marker", locals: { flat: flat })
        }
      end
    end
    ```
5. Generate a Stimulus controller for the map (NB: [private properties in JavaScript](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes/Private_class_fields) are prefixed with a `#`):

    ```sh
    rails generate stimulus map
    ```

    ```js
    // app/javascript/controllers/map_controller.js

    import { Controller } from '@hotwired/stimulus'
    import mapboxgl from 'mapbox-gl'

    export default class extends Controller {
      static values = {
        apiKey: String,
        markers: Array
      }

      connect() {
        mapboxgl.accessToken = this.apiKeyValue

        this.map = new mapboxgl.Map({
          container: this.element,
          style: 'mapbox://styles/mapbox/streets-v10' // use Mapbox Studio for more map styles!
        })

        this.#addMarkersToMap()
        this.#fitMapToMarkers()
      }

      #addMarkersToMap() {
        this.markersValue.forEach((marker) => {
          const popup = new mapboxgl.Popup().setHTML(marker.info_window_html)

          // Create a HTML element for your custom marker
          const customMarker = document.createElement("div")
          customMarker.innerHTML = marker.marker_html
          
          // Pass the element as an argument to the new marker
          new mapboxgl.Marker()
            .setLngLat([marker.lng, marker.lat])
            .setPopup(popup)
            .addTo(this.map)
        })
      }

      #fitMapToMarkers() {
        const bounds = new mapboxgl.LngLatBounds()
        this.markersValue.forEach(marker => bounds.extend([ marker.lng, marker.lat ]))
        this.map.fitBounds(bounds, { padding: 70, maxZoom: 15, duration: 0 })
      }
    }
    ```

### Search on a map

1. Install the [mapbox-gl-geocoder](https://npmjs.com/package/@mapbox/mapbox-gl-geocoder) package:

    ```sh
    importmap pin @mapbox/mapbox-gl-geocoder
    ```

    ```erb
    <!-- app/views/layouts/application.html.erb -->

    <!-- ... -->
    <link rel="stylesheet" href="https://api.mapbox.com/mapbox-gl-js/plugins/mapbox-gl-geocoder/v5.0.0/mapbox-gl-geocoder.css" type="text/css">
    ```

2. Initialize the geocoder in the Stimulus controller:

    ```js
    // app/javascript/controllers/map_controller.js
    
    // [...]
    import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder'

    // [...]
      connect() {
        // [...]
        this.#addMarkersToMap()
        this.#fitMapToMarkers()

        const geoCoder = new MapboxGeocoder({ 
          accessToken: mapboxgl.accessToken,                
          mapboxgl: mapboxgl 
        })
        this.map.addControl(geocoder)
      }
    ```

### Autocomplete address

1. Add Mapbox Geocoder to the address input:

    ```erb
    <!-- app/views/flats/_form.html.erb -->
    
    <!-- [...] -->
    <%= f.input :address,
        input_html: { 
          data: { 
            address_autocomplete_target: "address" 
          }
        },
        wrapper_html: { 
          data: { 
            controller: "address-autocomplete", 
            address_autocomplete_api_key_value: ENV["MAPBOX_API_KEY"]
          }
        }
    %>
    ```

2. Generate a new Simulus controller to handle the autocomplete:

    ```sh
    rails generate stimulus address_autocomplete
    ```

    ```js
    // app/javascript/controllers/address_autocomplete_controller.js
    
    import { Controller } from '@hotwired/stimulus'
    import MapboxGeocoder from '@mapbox/mapbox-gl-geocoder'

    export default class extends Controller {
      static values = { 
        apiKey: String 
      }
      static targets = ['address']

      connect() {
        this.geocoder = new MapboxGeocoder({
          accessToken: this.apiKeyValue,
          types: 'country,region,place,postcode,locality,neighborhood,address'
        })
        this.geocoder.addTo(this.element)

        // Listeners
        this.geocoder.on('result', event => this.#setInputValue(event))
        this.geocoder.on('clear', () => this.#clearInputValue())
      }

      // Avoid duplicate instances
      disconnect() {
        this.geocoder.onRemove()
      }

      #setInputValue(event) {
        this.addressTarget.value = event.result['place_name']
      }

      #clearInputValue() {
        this.addressTarget.value = ''
      }
    }
    ```

    You can see which geographic types you can use in the geocoder [here](https://docs.mapbox.com/api/search/geocoding/#data-types).
