<template>
<main>
  <h1>WordPress Archive Vue</h1>

  <aside>
    <h2>Filters</h2>

    <details v-for='term in terms' :key='term.term_id'>
      <summary>{{ term.name }}</summary>

      <ul>
        <li><button @click='toggle({event: $event, data: {parent: term.slug}})'>Toggle All</button>

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

<script>
import Archive from '../src/archive.vue';

export default {
  extends: Archive,
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
          programs: p => p,

          /**
           * Terms Map
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
     *                              event: $event - The click event
     *                              data: f       - The term object
     */
    change: function(toChange) {
      this.$set(toChange.data, 'checked', !toChange.data.checked);

      this.click(toChange);
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

<style>
  body {
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
      Oxygen-Sans, Ubuntu, Cantarell, "Helvetica Neue", sans-serif;
  }

  summary, label, input, button {
    cursor: pointer;
  }
</style>