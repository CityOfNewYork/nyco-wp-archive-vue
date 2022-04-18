<script>
  /**
   * WordPress Archive Vue. Creates a filterable reactive interface using Vue.js
   * for a WordPress Archive. Uses the WordPress REST API for retrieving filters
   * and posts. Fully configurable for any post type (default or custom). Works
   * with multiple languages using the lang attribute set in on the html tag
   * and the multilingual url endpoint provided by WPML.
   *
   * This component does not include a default template but it can be extended
   * by a parent component and configured with the template, script, or style
   * tags.
   */
  export default {
    props: {
      strings: {type: Object}
    },
    data: () => ({
      /**
       * Wether the app has been initialized or not
       *
       * @type {Boolean}
       */
      init: false,

      /**
       * The post type to query. Overide this with a custom post type
       *
       * @type {String}
       */
      type: 'posts',

      /**
       * Post type terms, used to filter visible posts. Terms are a custom built
       * object provided by a custom endpoint set in `register-rest-routes.php`.
       * This endpoint must be present for the app to provide filtering by
       * terms.
       *
       * @type {Array}
       */
      terms: [],

      /**
       * Initial query and current query used to request posts via the WP REST
       * API. This JSON object maps directly to the URL query used by the WP
       * REST API.
       *
       * @type {Object}
       */
      query: {
        page: 1,
        per_page: 5
      },

      /**
       * This is a list of 'safe' params to use with the WP REST API query.
       * Other parameters can create conflicts. This list is based on the
       * WP REST API /posts endpoint arguments;
       * https://developer.wordpress.org/rest-api/reference/posts/#arguments
       *
       * This array can be extended to include custom params such as taxonomies
       *
       * @type {Array}
       */
      params: [
        'context',
        'page',
        'per_page',
        'search',
        'after',
        'author',
        'author_exclude',
        'before',
        'exclude',
        'include',
        'offset',
        'order',
        'orderby',
        'slug',
        'status',
        'tax_relation',
        'categories',
        'categories_exclude',
        'tags',
        'tags_exclude',
        'sticky'
      ],

      /**
       * Initial headers and current headers of visible posts. Headers are used
       * to determine if there are additional pages surrounding a query.
       *
       * @type {Object}
       */
      headers: {
        pages: 0,
        total: 0,
        link: 'rel="next";'
      },

      /**
       * This is the endpoint list for terms and post requests. The terms
       * endpoint does not exist in WordPress by default so it needs to be
       * created via the 'register_rest_route' filter.
       *
       * @type {Object}
       */
      endpoints: {
        terms: '/wp-json/api/v1/terms',
        posts: '/wp-json/wp/v2/posts'
      },

      /**
       * If the domain where the App is hosted is different from the API domain.
       *
       * @type {String}
       */
      domain: '',

      /**
       * Each endpoint above will access a map to take the data from the request
       * and transform it for the app's display purposes.
       *
       * @type   {Function}
       *
       * @return {Object}    Object with a mapping function for each endpoint
       */
      maps: function() {
        return {
          terms: terms => ({
            // checked: false (determined by the initial interaction)
            active: false, // For use with regions and aria-hidden
            name: terms.labels.archives,
            slug: terms.name,
            checkbox: false, // wether the trigger is a checkbox
            toggle: true,
            filters: terms.terms.map(filters => ({
              id: filters.term_id,
              name: filters.name,
              slug: filters.slug,
              parent: terms.name,
              active: ( // for use with setting the tabindex value
                this.query.hasOwnProperty(terms.name) &&
                this.query[terms.name].includes(filters.term_id)
              ),
              checked: (
                this.query.hasOwnProperty(terms.name) &&
                this.query[terms.name].includes(filters.term_id)
              )
            }))
          }),
          posts: p => ({
            title: p.title.rendered,
            link: p.link,
            excerpt: p.excerpt.rendered
          })
        };
      },

      /**
       * Initial history and current history of visible posts. This is used
       * to configure how the URL of the page is rewritten. It can include
       * parameters to omit and a mapping object that will convert WP Query vars
       * vars to WP REST query vars. This prevents conflicts with the original
       * WP Query.
       *
       * filterSafe determines wether to filter the safe params from the history
       * replaceState method.
       *
       * @type {Object}
       */
      history: {
        omit: [
          'page',
          'per_page'
        ],
        map: {
          // ex; 'wordpress-query-var': 'vue-app-query-var'
        },
        filterParams: false
      },

      /**
       * Storage for all Post content, this is set when posts are queried. Post
       * content is organized by pages. Each page is an object that includes a
       * headers object, posts object (actual content), query object (the
       * original query for the page), and show boolean that determines if it
       * should be visible. The first page will always be undefined. When the
       * query is modified by selecting taxonomies to filter on, this entire
       * history is rewritten.
       *
       * @type {Array}
       */
      posts: [
        // undefined
        // {
        //   headers: { ... },
        //   posts: [ ... post content ... ],
        //   query: { ... },
        //   show: true
        // }
      ],
    }),
    computed: {
      /**
       * Wether there are no posts to show but a query is being made
       *
       * @type {Boolean}
       */
      loading: function() {
        if (!this.posts.length) return false;

        let page = this.posts[this.query.page];

        return this.init && !page.posts.length && page.show;
      },

      /**
       * Wether there posts to display from the modified query
       *
       * @type {Boolean}
       */
      none: function() {
        return !this.headers.pages && !this.headers.total;
      },

      /**
       * Wether there is another page or not
       *
       * @type {Boolean}
       */
      next: function() {
        let number = this.query.page;
        let total = this.headers.pages;

        if (!this.posts.length) return false;

        let page = this.posts[number];

        return (number < total) && (page.posts.length && page.show);
      },

      /**
       * Wether there is a previous page or not
       *
       * @type {Boolean}
       */
      previous: function() {
        return (this.query.page > 1);
      },

      /**
       * Wether posts are currently being filtered
       *
       * @type {Boolean}
       */
      filtering: function() {
        if (!this.init) return false;

        return (this.terms.find(t => t.active)) ? true : false;
      },

      /**
       * The language of the document (and query)
       *
       * @type {String}
       */
      lang: () => {
        let lang = document.querySelector('html').lang;

        return (lang !== 'en') ? {
          code: lang,
          path: `/${lang}`
        } : {
          code: 'en',
          path: ''
        };
      },

      /**
       * By default the application returns paged results, this computed property
       * adds all visible posts to a single array to show.
       *
       * @return  {Array}  All visible posts in one array
       */
      postsFlat() {
        let posts = this.posts.filter(page => {
          return (page && page.show) ? page.posts : false;
        }).map(page => {
          return page.posts;
        });

        return posts.flat();
      },

      /**
       * Find the total visible post by mapping each shown page and adding the
       * number of posts.
       *
       * @return  {Number}  Representing the total visible posts
       */
      totalVisible: function() {
        let show = this.posts.map(page => {
          return (page && page.show) ? page.posts.length : 0;
        });

        return (show.length) ?
          show.reduce((acc, cur) => acc + cur, 0) : 0;
      },

      /**
       * Return the total number of checked filters
       *
       * @return  {Number}  Number of checked filters
       */
      totalFilters: function() {
        return this.terms.reduce((acc, cur) => {
          return acc + cur.filters.filter(f => f.checked).length
        }, 0);
      }
    },
    methods: {
      /**
       * Converts a JSON object to URL Query. Opposite of buildJsonQuery.
       *
       * @param  {Object} query  URL Query structured as JSON Object.
       * @param  {Array}  omit   Set of params as flags to not include in
       *                         the returned query string.
       * @param  {Object} rev    Set of parameter maps to replace provided
       *                         param names in the query.
       *
       * @return {String}        The query string.
       */
      buildUrlQuery: function(query, omit = false, rev = false) {
        let q = Object.keys(query)
          .map(k => {
            if (omit && omit.includes(k)) return false;

            let map = (rev && rev.hasOwnProperty(k)) ? rev[k] : k;

            if (Array.isArray(query[k]))
              return query[k].map(a => `${map}[]=${a}`).join('&');
            return `${map}=${query[k]}`;
          }).filter(k => (k)).join('&');

        return (q !== '') ? '?' + q : '';
      },

      /**
       * Converts a URL Query String to a JSON Object. Opposite of buildUrlQuery.
       *
       * @param  {String} query  URL Query String.
       *
       * @return {Object}         URL Query structured as JSON Object.
       */
      buildJsonQuery: function(query) {
        if (query === '') return false;

        let params = new URLSearchParams(query);
        let q = {};

        // Set keys in object and get values, convert to number (!NaN)
        params.forEach(function(value, key) {
          let k = key.replace('[]', '');

          if (!q.hasOwnProperty(k))
            q[k] = params.getAll(key).map(value => {
              return (isNaN(value)) ? value : +value;
            });
        });

        // Reverse map the parameters to the actual query vars
        Object.keys(this.history.map).map(key => {
          if (q.hasOwnProperty(this.history.map[key])) {
            q[key] = q[this.history.map[key]];

            delete q[this.history.map[key]];
          }
        });

        return q;
      },

      /**
       * Set the URL Query
       *
       * @param  {Object} query  URL Query structured as JSON Object.
       *
       * @return {Object}        Vue instance
       */
      replaceState: function(query) {
        let history = {};

        // Filter safe params for the history query
        if (this.history.filterParams) {
          Object.keys(query).map(p => {
            if (this.params.includes(p)) {
              history[p] = query[p];
            }
          });
        } else {
          history = query;
        }

        let state = this.buildUrlQuery(history, this.history.omit, this.history.map);

        window.history.replaceState(null, null, window.location.pathname + state);

        return this;
      },

      /**
       * Basic fetch for retrieving data from an endpoint configured in the
       * data.endpoints property.
       *
       * @param  {Object}  data  A key representing an endpoint configured in
       *                         the data.endpoints property.
       *
       * @return {Promise}       The fetch request for that endpoint.
       */
      fetch: function(data = false) {
        if (!data) return data;

        return (this[data].length) ? this[data] :
          fetch(`${this.domain}${this.lang.path}${this.endpoints[data]}`)
            .then(response => response.json())
            .then(d => {
              this.$set(this, data, d.map(this.maps()[data]));
            });
      },

      /**
       * The click event to begin filtering.
       *
       * @param  {Object} event  The click event on the element that triggers
       *                         the filter.
       *
       * @return {Object}        Vue instance
       */
      click: function(event) {
        let taxonomy = event.data.parent;
        let term = event.data.id || false;

        if (term) {
          // set the individual filter to checked
          this.$set(event.data, 'checked', !event.data.checked);

          // check the parent taxonomy if all filters are checked,
          // set parent to truthy checked state if so
          let tax = this.terms.find(t => t.slug === taxonomy);
          let checked = (tax.filters.filter(t => t.checked).length === tax.filters.length);

          this.$set(tax, 'checked', checked);

          this.filter(taxonomy, term);
        } else {
          this.filterAll(taxonomy);
        }

        return this;
      },

      /**
       * The reset event to toggle all filters.
       *
       * @param  {Object} event  The click event on the element that triggers
       *                         the filter.
       *
       * @return {Object}        Vue instance
       */
      toggle: function(event) {
        let taxonomy = event.data.parent;

        this.filterAll(taxonomy);

        return this;
      },

      /**
       * Single filter function. If the filter is already present in the query
       * it will add the filter to the query.
       *
       * @param  {String} taxonomy  The taxonomy slug of the filter
       * @param  {Number} term      The id of the term to filter on
       *
       * @return {Object}           Vue instance
       */
      filter: function(taxonomy, term) {
        let terms = (this.query.hasOwnProperty(taxonomy)) ?
          this.query[taxonomy] : [term]; // get other query or initialize.

        // Toggle, if the taxonomy exists, filter it out, otherwise add it.
        if (this.query.hasOwnProperty(taxonomy))
          terms = (terms.includes(term)) ?
            terms.filter(el => el !== term) : terms.concat([term]);

        this.updateQuery(taxonomy, terms);

        return this;
      },

      /**
       * A control for filtering all of the terms in a particular taxonomy on
       * or off.
       *
       * @param  {String} taxonomy  The taxonomy slug of the filter
       *
       * @return {Object}           Vue instance
       */
      filterAll: function(taxonomy) {
        let tax = this.terms.find(t => t.slug === taxonomy);

        // The terms may not have the checked state available. If not use the
        // state of all the filters in the taxonomy.
        let checked = (tax.hasOwnProperty('checked'))
          ? !(tax.checked) : !(tax.filters.filter(t => t.checked).length === tax.filters.length);

        this.$set(tax, 'checked', checked);

        let terms = tax.filters.map(term => {
          this.$set(term, 'checked', checked);

          return term.id;
        });

        this.updateQuery(taxonomy, (checked) ? terms : []);

        return this;
      },

      /**
       * This updates the query property with the new filters.
       *
       * @param  {String}  taxonomy  The taxonomy slug of the filter
       * @param  {Array}   terms     Array of term ids
       *
       * @return {Promise}           Resolves when the terms are updated
       */
      updateQuery: function(taxonomy, terms) {
        return new Promise((resolve) => { // eslint-disable-line no-undef
          this.$set(this.query, taxonomy, terms);
          this.$set(this.query, 'page', 1);

          // hide all of the posts
          this.posts.map((value, index) => {
            if (value) this.$set(this.posts[index], 'show', false);

            return value;
          });

          resolve();
        })
        .then(this.wp)
        .catch(message => {
          // eslint-disable-next-line no-undef
          if (process.env.NODE_ENV === 'development') {
            console.dir(message);
          }
        });
      },

      /**
       * A function to reset the filters to "All Posts."
       *
       * @return {Promise}        Resolves after resetting the filter
       */
      reset: function(event) {
        for (let index = 0; index < this.terms.length; index++) {
          this.updateQuery(this.terms[index].slug, []);

          this.$set(this.terms[index], 'checked', false);

          this.terms[index].filters.forEach(f => {
            this.$set(f, 'checked', false);
          });
        }
      },

      /**
       * A function to paginate up or down a post's list based on the change amount
       * assigned to the clicked element.
       *
       * @param  {Object}  event  The click event of the pagination element
       *
       * @return {Promise}        Resolves after updating the pagination in the
       *                          query
       */
      paginate: function(event) {
        // The change is the next page as well as an indication of what
        // direction we are moving in for the queue.
        let change = parseInt(event.target.dataset.amount);
        let page = this.query.page + change;

        event.preventDefault();

        return new Promise(resolve => { // eslint-disable-line no-undef
          this.$set(this.query, 'page', page);
          this.$set(this.posts[this.query.page], 'show', true);

          this.queue([0, change]);

          resolve();
        });
      },

      /**
       * Wrapper for the queue promise
       *
       * @return {Promise} Returns the queue function.
       */
      wp: function() {
        return this.queue();
      },

      /**
       * This queues the current post request and the next request based on the
       * direction of pagination. It uses an Async method to retrieve the
       * requests in order so that we can determine if there are more posts to
       * show after the request for the current view.
       *
       * @param  {Array}  queries  The amount of queries to make and which
       *                           direction to make them in. 0 means the
       *                           current page, 1 means the next page. -1
       *                           would mean the previous page.
       *
       * @return {Object}          Vue instance.
       */
      queue: function(queries = [0, 1]) {
        // Set a benchmark query to compare the upcomming query to.
        let Obj1 = Object.assign({}, this.query); // create copy of object.
        delete Obj1.page; // delete the page attribute because it will be different.
        Object.freeze(Obj1); // prevent changes to our comparison.

        // The function is async because we want to wait until each promise
        // is query is finished before we run the next. We don't want to bother
        // sending a request if there are no previous or next pages. The way we
        // find out if there are previous or next pages relative to the current
        // page query is through the headers of the response provided by the
        // WP REST API.
        (async () => {
          for (let i = 0; i < queries.length; i++) {
            let query = Object.assign({}, this.query);
            // eslint-disable-next-line no-undef
            let promise = new Promise(resolve => resolve());
            let pages = this.headers.pages;
            let page = this.query.page;
            let current = false;
            let next = false;
            let previous = false;

            // Build the query and set its page number.
            Object.defineProperty(query, 'page', {
              value: page + queries[i],
              enumerable: true
            });

            // There will never be a page 0 or below, so skip this query.
            if (query.page <= 0) continue;

            // Check to see if we have the page that we are going to queued
            // and the query structure of that page matches the current query
            // structure (other than the page, which will obviously be
            // different). This will help us determine if we need to make a new
            // request.
            let havePage = (this.posts[query.page]) ? true : false;
            let pageQueryMatches = false;

            if (havePage) {
              let Obj2 = Object.assign({}, this.posts[query.page].query);
              delete Obj2.page;
              pageQueryMatches = (JSON.stringify(Obj1) === JSON.stringify(Obj2));
            }

            if (havePage && pageQueryMatches) continue;

            // If this is the current page we want the query to go through.
            current = (query.page === page);

            // If there is a next or previous page, we'll prefetch them.
            // We'll know there's a next or previous page based on the
            // headers sent by the current page query.
            next = (page < pages && query.page > page);
            previous = (page > 1 && query.page < page);

            if (current || next || previous)
              await promise.then(() => {
                return this.wpQuery(query);
              })
              .then(this.response)
              .then(data => {
                let headers = Object.assign({}, this.headers);

                // If this is the current page, replace the browser history state.
                if (current) this.replaceState(query);

                this.process(data, query, headers);
              }).catch(this.error);
          }
        })();

        return this;
      },

      /**
       * Builds the URL query from the provided query property.
       *
       * @param  {Object}  query  A WordPress query written in JSON format
       *
       * @return {Promise}        The fetch request for the query
       */
      wpQuery: function(query) {
        // Create a safe WP Query using permitted params
        let wpQuery = {};

        Object.keys(query).map(p => {
          if (this.params.includes(p))
            wpQuery[p] = query[p];
        });

        // Build the url query.
        let url = [
          this.domain,
          this.lang.path,
          this.endpoints[this.type],
          this.buildUrlQuery(wpQuery)
        ].join('');

        // Set posts and store a copy of the query for reference.
        this.$set(this.posts, query.page, {
          posts: [],
          query: Object.freeze(query),
          show: (this.query.page >= query.page)
        });

        return fetch(url);
      },

      /**
       * Handles the response, setting the headers of the query and returning
       * the response as JSON.
       *
       * @return {Object} The response object as JSON
       */
      response: function(response) {
        let headers = {
          total: 'X-WP-Total',
          pages: 'X-WP-TotalPages',
          link: 'Link'
        };

        if (response.ok) {
          let keys = Object.keys(headers);

          for (let i = 0; i < keys.length; i++) {
            let header = response.headers.get(headers[keys[i]]);
            let value = (isNaN(header)) ? header : (parseInt(header) || 0);

            headers[keys[i]] = value;
          }

          this.$set(this, 'headers', headers);
        }

        return response.json();
      },

      /**
       * Processes the posts and maps the data to data maps provided by the
       * data.maps property.
       *
       * @param {Object} data     The post data retrieved from the WP REST API
       * @param {Object} query    The the query used to get the data
       * @param {Object} headers  The the headers of the request
       */
      process: function(data, query, headers) {
        // If there are posts for this query, map them to the template.
        let posts = (Array.isArray(data)) ?
          data.map(this.maps()[this.type]) : false;

        // Set posts and store a copy of the query for reference.
        this.$set(this.posts[query.page], 'posts', posts);
        this.$set(this.posts[query.page], 'headers', Object.freeze(headers));

        // If there are no posts, pass along to the error handler.
        if (!Array.isArray(data))
          this.error({error: data, query: query});

        this.$set(this, 'init', true);
      },

      /**
       * Error response thrown when there is an error in the WP REST AP request.
       *
       * @param {Object} response  The error response
       */
      error: function(response) {
        // eslint-disable-next-line no-undef
        if (process.env.NODE_ENV === 'development') {
          console.dir(response);
        }
      },

      /**
       * Convert the current Query or a passed query to JS readable format
       *
       * @param  {String} query  An existing query to parse
       *
       * @return {Object}         Query as a JSON object
       */
      getState: function(query = false) {
        query = (query) ? query : this.buildJsonQuery(window.location.search);

        Object.keys(query).map(key => {
          this.$set(this.query, key, query[key]);
        });

        return this;
      }
    }
  }
</script>