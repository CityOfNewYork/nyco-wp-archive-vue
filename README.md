# WordPress Archive Vue

This [Vue.js](https://vuejs.org/) app creates a reactive filterable interface for a post archive on a WordPress site. The app is intended to be configurable for any post type (including custom post types). It uses the browsers URL search parameters to build a query to the [WordPress REST API](https://developer.wordpress.org/rest-api/) for retrieving taxonomy, term, and post data. It supports multilingual sites with [WPML](https://wpml.org/) by retrieving the lang attribute set on the html lang tag and using it with the [languages in directory settings](https://wpml.org/documentation/getting-started-guide/language-setup/language-url-options/#different-languages-in-directories). The app supports customized HTML templates with methods and computed properties. Extend it with a parent component and configure with your own template, scripts, and styles. See the example below for more details.

## Usage

### Install

Install this dependency in your theme:

```shell
$ npm install @nycopportunity/wp-archive-vue
```

The following example is based on the implementation in the [ACCESS NYC WordPress theme](https://github.com/CityOfNewYork/ACCESS-NYC/tree/main/wp-content/themes/access). The live example can be seen on [access.nyc.gov/programs](https://access.nyc.gov/programs/). The example notably uses the [Vue.js createElement](https://vuejs.org/v2/guide/render-function.html#createElement-Arguments) method and pre-rendering features to avoid the use of unsafe `eval()` method (creating code from strings) by the application. However, this implementation method is optional and it can be mounted to a Vue instance in a more traditional fashion.

### Register a REST route for terms

It is recommended to [register a new REST route](https://developer.wordpress.org/reference/functions/register_rest_route/) in your WordPress site that will return the taxonomy and terms data for the post type that will be queried. This example uses the [Transients API](https://developer.wordpress.org/apis/handbook/transients/) to cache requests results for performance.

```php
/**
 * Register REST Route shouldn't be done before the REST api init hook so we
 * will hook into that action.
 */
add_action('rest_api_init', function() {
  /**
   * Returns a list of public taxonomies and their terms. Transients can greatly
   * improve the performance of the REST API so it is recommended to use them.
   *
   * @param  {String} $namespace  Namespace for the route
   * @param  {String} $route      Endpoint for the route
   * @param  {Array}              An array including REST methods and a function
   *                              that processes the request
   *
   * @return {Object}             A WordPress REST Response
   */
  register_rest_route('api/v1', '/terms/', array(
    'methods' => 'GET',
    'callback' => function(WP_REST_Request $request) use ($transients) {
      $lang = (defined('ICL_LANGUAGE_CODE')) ? '_' . ICL_LANGUAGE_CODE : '';

      $data = get_transient('rest_terms_json' . $lang);

      if (false === $data) {
        $data = [];

        // Get public taxonomies and build our initial assoc. array
        foreach (get_taxonomies(array(
          'public' => true,
          '_builtin' => false
        ), 'objects') as $taxonomy) {
            $data[] = array(
              'name' => $taxonomy->name,
              'labels' => $taxonomy->labels,
              'taxonomy' => $taxonomy,
              'terms' => array()
            );
        }

        // Get the terms for each taxonomy
        $data = array_map(function ($tax) {
          $tax['terms'] = get_terms(array(
            'taxonomy' => $tax['name'],
            'hide_empty' => false,
          ));
          return $tax;
        }, $data);

        set_transient('rest_terms_json' . $lang, $data, WEEK_IN_SECONDS);
      }

      $response = new WP_REST_Response($data); // Create the response object

      $response->set_status(200); // Add a custom status code

      return $response;
    }
  ));
});
```

### Configure the component

Create a proxy module that extends the archive methods and passes your desired configuration for the app. For the sake of this example it can be named **my-archive.js**.

```vue
<script>
import Archive from '@nycopportunity/wp-archive-vue/src/archive.vue';

export default {
  extends: Archive, // Extend the Archive app here
  data: function() {
    return {
      /**
       * This is our custom post type to query
       *
       * @type {String}
       */
      type: 'programs',

      /**
       * This is the endpoint list for terms and post requests
       *
       * @type  {Object}
       *
       * @param {String} terms     A required endpoint for the list of filters
       * @param {String} programs  This is based on the 'type' setting above
       */
      endpoints: {
        terms: '/wp-json/api/v1/terms',
        programs: '/wp-json/wp/v2/programs'
      },

      /**
       * This is the domain for our local WordPress installation.
       *
       * @type {String}
       */
      domain: 'http://localhost:8080',

      /**
       * Each endpoint above will access a map to take the data from the request
       * and transform it for the app's display purposes
       *
       * @type   {Function}
       *
       * @return {Object}    Object with a mapping function for each endpoint
       */
      maps: function() {
        return {
          /**
           * Programs endpoint data map
           */
          programs: p => p,

          /**
           * Terms endpoint data map
           */
          terms: terms => ({
            name: terms.labels.archives,
            slug: terms.name,
            filters: terms.terms.map(filters => ({
              id: filters.term_id,
              name: filters.name,
              slug: filters.slug,
              parent: terms.name,
              active: (
                  this.query.hasOwnProperty(terms.name) &&
                  this.query[terms.name].includes(filters.term_id)
                ),
              checked: (
                  this.query.hasOwnProperty(terms.name) &&
                  this.query[terms.name].includes(filters.term_id)
                )
            }))
          })
        };
      }
    };
  },

  /**
   * @type {Object}
   */
  methods: {
    /**
     * Proxy for the click event that toggles the filter.
     *
     * @param   {Object}  toChange  A constructed object containing:
     *                              event - The click event
     *                              data  - The term object
     *
     * @return  {Object}            Vue Instance
     */
    change: function(toChange) {
      this.$set(toChange.data, 'checked', !toChange.data.checked);

      this.click(toChange);

      return this;
    }
  },

  /**
   * The created hook starts the application
   *
   * @url https://vuejs.org/v2/api/#created
   *
   * @type {Function}
   */
  created: function() {
    // Add custom taxonomy queries to the list of safe params
    ['programs', 'populations-served'].map(p => {
      this.params.push(p);
    });

    // Initialize the application
    this.getState()       // Get window.location.search (filter history)
      .queue()            // Initialize the first page request
      .fetch('terms')     // Get the terms from the 'terms' endpoint
      .catch(this.error);
  }
};
</script>
```

#### Configuration

**Data Properties**

Configuration is done through via the following data properties.

Property    | Required     | Description
------------|--------------|-
`type`      | *no*         | The post type endpoint to query. Defaults to `posts`.
`endpoints` | *no*         | Object containing the `posts` and `terms` endpoints. The key for `posts` should match the `type` property.
`domain`    | *no*         | The root domain of the endpoints to query. Only needed if the app will be making cross-origin requests.
`maps`      | **yes**      | A function that contains the data maps for `posts` and `terms` responses. Described below.

The `maps` function allows you to transform the data in the response from the api and map it to your custom Vue.js template or components. It is also possible to pass the raw data, but a map function must be present for each endpoint defined in the `endpoints` data property even if it just returns the same data.

* The {{ endpoint }} key should match the endpoint defined in the endpoints data property
* A single {{ data }} point from the WP REST query is passed to the mapping function at a time
* The returned object {} is wrapped in a grouping operator () to execute scripts on the returned object
* Set your component attributes with data properties from the WP REST response attributes

```javascript
maps: function() {
  return {
    // Example mapping function
    {{ endpoint }}: {{ data }} => ({
      {{ your component attribute }}: {{ data }}.{{ WP REST response attribute }}
    }),

    // A more complete example for the terms endpoint data
    terms: terms => ({
      name: terms.labels.archives,
      slug: terms.name,
      filters: terms.terms.map(filters => ({
        id: filters.term_id,
        slug: filters.slug,
        parent: terms.name
      }))
    })
  };
}
```

**Methods**

`change` - *optional* (see demonstration)

In the demonstration example the `change` method is used as a wrapper to pass the click method and invoke the primary method for adding posts to the archive view. This isn't required but you may add it to your app to change the properties of other components in the app such as the active state of the filters.

```html
<input type='checkbox' :value='filter.slug' :checked='filter.checked' @change='change({event: $event, data: filter})' />
```

```javascript
change: function(toChange) {
  this.$set(toChange.data, 'checked', !toChange.data.checked); // change the active state of the filter data

  this.click(toChange); // invoke the main filtering method

  return this;
}
```

**Hooks**

The `created` hook is required to initialize the application. At a minimum the following block is required create the initial request to the terms object.

```javascript
this.getState()       // Get window.location.search (filter history)
  .queue()            // Initialize the first page request
  .fetch('terms')     // Get the terms from the 'terms' endpoint
  .catch(this.error);
```

### Create a view template

This is where the reactive DOM for your view is added.

```vue
<template>
<main>
  <h1>WordPress Archive Vue</h1>

  <aside>
    <h2>Filters</h2>

    <details v-for='term in terms' :key='term.term_id'>
      <summary>{{ term.name }}</summary>

      <ul>
        <li v-for='filter in term.filters' :key='filter.slug'>
          <label class='checkbox'>
            <input type='checkbox' :value='filter.slug' :checked='filter.checked' @change='change({event: $event, data: filter})' />

            <span v-html='filter.name'>{{ filter.name }}</span>
          </label>
        </li>
      </ul>
    </details>
  </aside>

  <article>
    <h2>Posts</h2>

    <div v-for='page in posts' :key='`page-${posts.indexOf(page)}`'>
      <div v-if='page && page.show'>
        <h3>Page {{ posts.indexOf(page) }}</h3>

        <details v-for='post in page.posts' :key='post.id'>
          <summary v-html='post.title.rendered'>
            {{ post.title.rendered }}
          </summary>

          <pre>{{ post }}</pre>
        </details>
      </div>
    </div>

    <p>
      <button @click='paginate' v-if='next' data-amount='1'>
        Load More Posts
      </button>
    </p>
  </article>
</main>
</template>
```

#### Data

There are two main data objects in the app that are used to display archive results; `terms` and `posts`. The app will query the REST api to retrieve the initial data. Each time a request is made and a response is returned, the data is passed through a mapping function that can be used to customize what is passed to your app or to custom Vue.js components. The example above includes a mapping function for `terms`.

Data    | Description
--------|-
`terms` | An array of WordPress taxonomy objects with nested term objects for each.
`posts` | An array of queried posts organized by page number.

#### Computed Properties

There are several computed properties that can be used in your application to create loading mechanisms and show pagination.

Properties  | Description
------------|-
`filtering` | Wether posts are currently being filtered
`lang`      | The language of the document (and query)
`loading`   | Wether there are no posts to show but a query is being made
`next`      | Wether there is another page with posts to display
`none`      | Wether there posts to display from the modified query
`previous`  | Wether there is a previous page with posts to display

#### Methods

`click` - **required**

The `click` method is the primary method included in the app that will invoke requests for new posts to be added to the archive. It accepts an object as an argument with the click event (`event`) an the term object (`data`). The only required properties to include in the term object are the taxonomy slug (`parent`) and the numerical ID of the WordPress taxonomy term (`id`).

```javascript
this.click({
  event: $event
  data: {
    parent: 'parent' // {String} Slug of the taxonomy
    id: 154          // {Number} ID of the taxonomy term object
    // ...
  }
});
```

`toggle` - *optional*

The `toggle` method will change the state of all the filters to active. If clicked again, it will toggle them to inactive. The main object argument for this method is similar to the `click` method, however, the only attribute that is required for the data object (`data`) is the taxonomy slug (`parent`).

```javascript
this.toggle({
  event: $event
  data: {
    parent: 'parent' // {String} Slug of the taxonomy
    // ...
  }
});
```

There are several other methods available to your application. Take a look at the source code to see them.

### Import and mount the app

Finally, import the application and mount it to your application. The method below uses the `createElement` function because the `MyArchive` application will be pre-rendered when imported.

```javascript
import Vue from 'vue/dist/vue.runtime.min';
import MyArchive from 'my-archive.vue';

new Vue({
  render: createElement => createElement(MyArchive)
}).$mount('[data-js="programs"]');
```

## Contributing

This module uses the [Vue.js CLI](https://cli.vuejs.org/) to run a simple application using a WordPress installation running on `localhost:8080`. Clone the repository and run `npm start` to start the application.

```shell
$ git clone https://github.com/CityOfNewYork/nyco-wp-archive-vue.git
```

```shell
$ cd nyco-wp-archive-vue
```

```shell
$ npm install
```

```shell
$ npm start
```

**Output**

```shell
> @nycopportunity/wp-archive-vue@1.1.3 start /nyco-wp-archive-vue
> vue serve proto/app.vue

INFO  Starting development server...
98% after emitting

DONE  Compiled successfully in 5208ms

  App running at:
  - Local:   http://localhost:8082/
  - Network: http://192.168.1.33:8082/

  Note that the development build is not optimized.
  To create a production build, run npm run build.
```

**http://localhost:8082/**

The application pictured in the screenshot below is querying a local installation of the [ACCESS NYC](https://github.com/CityOfNewYork/ACCESS-NYC) WordPress REST API.

![Prototype Screenshot](proto/screenshot.png)

---

![The Mayor's Office for Economic Opportunity](NYCMOEO_SecondaryBlue256px.png)

[The Mayor's Office for Economic Opportunity](http://nyc.gov/opportunity) (NYC Opportunity) is committed to sharing open source software that we use in our products. Feel free to ask questions and share feedback. **Interested in contributing?** See our open positions on [buildwithnyc.github.io](http://buildwithnyc.github.io/). Follow our team on [Github](https://github.com/orgs/CityOfNewYork/teams/nycopportunity) (if you are part of the [@cityofnewyork](https://github.com/CityOfNewYork/) organization) or [browse our work on Github](https://github.com/search?q=nycopportunity).
