# Search

- [Active Record](#active-record)
- [PgSearch](#pgsearch)
  - [Single model](#single-model)
  - [Associated models](#associated-models)
  - [Multisearch](#multisearch)
- [Elasticsearch](#elasticsearch)
  - [Installation](#installation)
  - [Usage](#usage)
- [Algolia](#algolia)


## Active Record

1. Add an input to search for records:

    ```erb
    <!-- app/views/movies/_form.html.erb -->

    <%= form_with url: movies_path do %>
      <%= text_field_tag :query,
            params[:query],
            placeholder: "Type a keyword"
        %>
      <%= submit_tag "Search" %>
    <% end %>
    ```

2. Implement the search logic in the controller:

    ```ruby
    # app/controllers/movies_controller.rb

    # [...]
    def index
      if params[:query].present?
        # WHERE
        @movies = Movie.where(title: params[:query])

        # WHERE ILIKE
        @movies = Movie.where("title ILIKE ?", "%#{params[:query]}%")

        # OR
        sql_subquery = "title ILIKE :query OR synopsis ILIKE :query"
        @movies = Movie.where(sql_subquery, query: "%#{params[:query]}%")

        # JOIN
        sql_subquery = <<~SQL
          movies.title ILIKE :query
          OR movies.synopsis ILIKE :query
          OR directors.first_name ILIKE :query
          OR directors.last_name ILIKE :query
        SQL
        @movies = Movie.joins(:director).where(sql_subquery, query: "%#{params[:query]}%")

        # @@ (multiple terms)
        sql_subquery = <<~SQL
          movies.title @@ :query
          OR movies.synopsis @@ :query
          OR directors.first_name @@ :query
          OR directors.last_name @@ :query
        SQL
        @movies = @movies.joins(:director).where(sql_subquery, query: params[:query])
      end
    end
    ```

## PgSearch

[PgSearch](https://github.com/Casecommons/pg_search) is a gem to build ActiveRecord named scopes that take advantage of PostgreSQL’s full-text search.

To install it, add the `pg_search` gem to your Gemfile and run `bundle install`.

### Single model

```ruby
# app/models/movie.rb

# [...]
include PgSearch::Model

pg_search_scope :search_by_title_and_synopsis,
  against: [:title, :synopsis],
  using: {
    tsearch: { 
      prefix: true 
    }
  }
```

```ruby
# rails console
Movie.search_by_title_and_synopsis("superman batm")
```

### Associated models

```ruby
# app/models/movie.rb

# [...]
pg_search_scope :global_search,
  against: [:title, :synopsis],
  associated_against: {
    director: [:first_name, :last_name]
  },
  using: {
    tsearch: { 
      prefix: true 
    }
  }
```

```ruby
# rails console
Movie.global_search("chris nolan")
```

### Multisearch

1. Generate and run a PgSearch migration:

    ```sh
    rails g pg_search:migration:multisearch
    rails db:migrate
    ```

2. Include PgSearch in all models involved in the multisearch:

    ```ruby
    # app/models/movie.rb

    class Movie < ApplicationRecord
      include PgSearch::Model
      multisearchable against: [:title, :synopsis]
    end
    ```

    ```ruby
    # app/models/tv_show.rb

    class TvShow < ApplicationRecord
      include PgSearch::Model
      multisearchable against: [:title, :synopsis]
    end
    ```

3. Rebuild the multisearch model and search:

    ```ruby
    # rails console

    PgSearch::Multisearch.rebuild(Movie)
    PgSearch::Multisearch.rebuild(TvShow)
    results = PgSearch.multisearch("superman")

    results.map do |result|
      puts result.searchable.title
    end
    ```

## Elasticsearch

[Elasticsearch](https://elastic.co) is a search engine capable of complex features, including:

- Scoring complexity
- Misspelling
- Suggestions
- Highlighting
- Autocomplete
- Geo-aware search

### Installation

1. Download and start Elasticsearch locally:

   - Download the zip file at [elastic.co/downloads/elasticsearch](https://elastic.co/downloads/elasticsearch) and unzip it
   - Open a new terminal tab and run:
     ```sh
     cd ~/Downloads/elasticsearch-??? # change with your version
     bin/elasticsearch -E "xpack.security.enabled=false"
     ```
   - Open [localhost:9200](http://localhost:9200) in your browser to check if is running correctly

2. Install the `elasticsearch` and `searchkick` gems:

    ```rb
    # Gemfile

    gem "elasticsearch"
    gem "searchkick"
    ```

    ```sh
    bundle install
    ```

### Usage

```rb
# app/models/movie.rb

class Movie < ApplicationRecord
  searchkick
  # [...]
end
```

```rb
# rails console

Movie.reindex
results = Movie.search("superman batm")

results.size
results.any?
results.each do |result|
  # [...]
end
```

For more information, see the [Elasticsearch](https://elastic.co/guide/en/elasticsearch/reference/current/index.html) and [Searchkick](https://github.com/ankane/searchkick) documentation.

## Algolia

Algolia is a cloud search engine that provides a real-time search API with rich features.

Use it when:
- You don’t want to deal with complex engine installions
- You want **speed**
- You need typo tolerance and other advanced features
- You mainly work with short texts. If you index large documents (e.g. PDF, web pages), ElasticSearch will do a better job.

Algolia can be used in Rails with the [`algolia-search`](https://github.com/algolia/algoliasearch-rails) gem.
