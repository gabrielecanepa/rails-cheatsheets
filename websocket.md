[websocket-img]: .github/assets/websocket.png

# WebSocket

![][websocket-img]

[Action Cable](https://guides.rubyonrails.org/action_cable_overview.html) is an integrated WebSocket framework for Rails.

## Installation

1. Install and start [Redis](https://redis.io):

    ```sh
    brew install redis
    brew services start redis
    ```

2. Install and import the `@rails/actioncable` package:

    ```sh
    importmap pin "@rails/actioncable"
    ```

    ```js
    // app/javascript/application.js

    import '@rails/actioncable'
    ```

## Usage

1. Generate a channel:

    ```sh
    rails g channel Chatroom
    ```

    ```ruby
    # app/channels/chatroom_channel.rb

    class ChatroomChannel < ApplicationCable::Channel
      def subscribed
        # Called once a consumer has become a subscriber of the channel.
        # Usually the place to setup any streams you want this channel to be sending to the subscriber.
      end

      def unsubscribed
        # Called once a consumer has cut its cable connection.
        # Can be used for cleaning up connections or marking users as offline or the like.
      end
    end
    ```

2. Add a stream to the `subscribed` lifecycle method, called when a client requests a subscription to the channel.

    - Static channel:

      ```ruby
      class ChatroomChannel < ApplicationCable::Channel
        def subscribed
          stream_from "world"
        end
      end
      ```

    - Dynamic channel:

      ```ruby
      class ChatroomChannel < ApplicationCable::Channel
        def subscribed
          stream_from "country_#{params[:country]}"
        end
      end
      ```

    - Dynamic channel related to an instance:

      ```ruby
      class ChatroomChannel < ApplicationCable::Channel
        def subscribed
          chatroom = Chatroom.find(params[:id])
          stream_for chatroom
        end
      end
      ```

    - Dynamic channel related to the current user:

      ```ruby
      class ChatroomChannel < ApplicationCable::Channel
        def subscribed
          stream_for current_user
        end
      end
      ```

      By default, `current_user` is not available in an Action Cable channel. To make it available, configure the `app/channels/application_cable/connection.rb` file:

      ```ruby
      module ApplicationCable
        class Connection < ActionCable::Connection::Base
          identified_by :current_user

          def connect
            self.current_user = find_verified_user
          end

          private

          def find_verified_user
            if verified_user = User.find_by(id: cookies.encrypted[:user_id])
              verified_user
            else
              reject_unauthorized_connection
            end
          end
        end
      end
      ```

3. In a controller, broadcast data to the channel after some changes.

    - Generic broadcast:

      ```ruby
      ActionCable.server.broadcast("world", data)
      ```

    - Broadcast in a model-specific channel:

      ```ruby
      ChatroomChannel.broadcast_to(@chatroom, data)
      ```

    - Broadcast in a user-specific channel:

      ```ruby
      ChatroomChannel.broadcast_to(current_user, data)
      ```

4. Subscribe the client to the channel:

    ```sh
    rails g stimulus chatroom
    ```

    ```js
    // app/javascript/controllers/chatroom_controller.js

    export default class extends Controller {
      connect() {
        this.channel = createConsumer().subscriptions.create({ channel: "ChatroomChannel", id: '42' }, {
          received: data => {
            // Do something when the data is broadcasted to the channel
          }
        })
      }

      disconnect() {
        this.channel.unsubscribe()
      }
    }
    ```

5. Use `this.perform` to call a method of a Rails controller:

    ```js
    // app/javascript/controllers/chatroom_controller.js

    // [...]
    received: data => {
      // Call `ChatroomChannel#example` with the `message` parameter
      this.perform('example', { message: 'Hello World' })
    }
    ```

    ```ruby
    # app/channels/chatroom_channel.rb

    class ChatroomChannel < ApplicationCable::Channel
      # [...]
      def example
        # Called when `this.perform('example')` is executed client-side
        # 'Hello World' is accessible via params[:message]
      end
    end
    ```

6. Update the DOM:

   - Change the broadcasted data to an HTML or JSON string using `render_to_string`, e.g.:

      ```ruby
      ChatroomChannel.broadcast_to(@chatroom, render_to_string(partial: "message", locals: { message: @message }))
      ```

   - Use the `received` method in the client-side controller to update the DOM:

      ```js
      // app/javascript/controllers/chatroom_controller.js

      // [...]
      received: data => {
        document.querySelector('ul').insertAdjacentHTML('beforeend', data)
      }
      ```
