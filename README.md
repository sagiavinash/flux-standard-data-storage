Flux Standard Data Storage
==========================

## Introduction

A standard for Flux store objects and compatible action objects

## Problem

1. When multiple actions(same action with different inputs) to data stored in flux stores is prone to query - data mismatch due to race conditions.
2. Need for a data cache and making it navigable through history.
3. Status of data stored in flux stores is often derived by the values(comparing current value with default values) or based on the events occured until now going against reactive programming.


### Design goals

1. Schema of data stored should be progressively enhanced so that all data reads/accessors are unaffected by changes in store.
2. Should be compliant with existing standards and ecosystem (Do no re-invent the wheel).

### Datapoint in a store

a dataPoint MUST

- be a plain javascript object
- have `data` property

a dataPoint MAY

- have `error` property and must be an error object
- have `isLoading` property
- have `query` property
- have `cache` property when `query` property is present
- have `prevQueries` when `query` & `cache` properties are present
- have `nextQueries` when `query`, `cache`& `prevQueries` properties are present

a dataPoint MUST NOT

- have a property which is not listed above

### `data`

The data property MAY be of any value

### `error`

The error property MUST be an error object, data property should not remain same

### `isLoading`

The isLoading property is a boolean, set to true by an action of LOADING type and set to false by the LOADED type.

### `query`

The query property MAY be of any value.

The value of query should be sent in `payload` property of a LOADING action ([FSA](https://github.com/acdlite/flux-standard-action)).

The value of query should be sent in `meta` property of a LOADED action ([FSA](https://github.com/acdlite/flux-standard-action)).

### `cache`

The cache property is a `set` (unique) of objects with `query`, `data` mandatory properties and optional `error` property.

### `prevQueries`

The prevQueries property is `stack` (LIFO) of queries.

### `nextQueries`

The nextQueries property is `stack` (LIFO) of queries.


### Actions

an action should be [flux-standard-action](https://github.com/acdlite/flux-standard-action) (FSA) compliant.

a datapoint MAY

- be updated by a sync event and the action `type` should be prefixed with `LOAD` keyword
  - the LOAD event should have the `query` as a property of the `meta` object
  - the LOAD event should have the `data` as the value or propery of the `payload`
- be updated by an async event and it should be done via two actions with action `type ` prefixed by `LOADING` and `LOADED` keywords.
  - the LOADING event should have the `query` in the `payload` object
  - the LOADED event should have the `query` as a property of the `meta` object
  - the LOADED event should have the `data` as the value or propery of  the `payload`



### Examples

###### 1. Simple data storage

``` js
/// Flux Actions (FSA Compliant)

// loaded data action
dispatch({
  type: 'LOADED_DATAPOINT',
  payload: 'value',
});

/// dataPoint in a store
datapoint: {
  data: 'value',
},
```


###### 2. Loading & error states

``` js
/// Flux Actions (FSA Compliant)

dispatch({
  type: actions.LOADING_DATAPOINT,
});

loadDataPoint().then((response) => {
  if (!response.error) {
    // loaded data action - success
    dispatch({
      type: actions.LOADED_DATAPOINT,
      payload: 'value',
    });
  } else {
    // loaded data action - failure
    dispatch({
      type: actions.LOADED_DATAPOINT,
      payload: new Error('message'),
      error: true,
    });
  }
});

/// dataPoint in a store (Flux Standard Data Storage Compliant)
reducer(state, action) {
  switch (action.type) {
    case actions.LOADING_DATAPOINT: {
      return {
        ...state,
        isLoading: true,
      };
    }
    case actions.LOADED_DATAPOINT: {
      return !action.error ? {
        ...state,
        datapoint: {
          ...state.datapoint,
          data: action.payload,
          isLoading: false,
        }
      } : {
        ...state,
        dataPoint: {
          ...state.dataPoint,
          data: null,
          error: action.payload,
          isLoading: false,
        },
      };
    }
  }
}
```


###### 3. Datapoint with a corresponding query

``` js
/// Flux Actions (FSA Compliant)

dispatch({
  type: actions.LOADING_DATAPOINT,
  payload: {
    query: query,
  },
});

loadDataPoint(query).then((response) => {
  if (!response.error) {
    dispatch({
      type: actions.LOADED_DATAPOINT,
      meta: {
        query: query,
      },
      payload: 'value',
    });
  } else {
    dispatch({
      type: actions.LOADED_DATAPOINT,
      meta: {
        query: query,
      },
      payload: new Error('message'),
      error: true,
    });
  }
});

/// dataPoint in a store (Flux Standard Data Storage Compliant)
reducer(state, action) {
  switch (action.type) {
    case actions.LOADING_DATAPOINT: {
      return {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: action.payload.query,
          isLoading: true,
        },
      };
    }
    case actions.LOADED_DATAPOINT: {
      if (_.isEqual(state.query, action.payload.query) { 
        // only updates if the LOADED action payload corresponds to the query of latest LOADING action. 
        // no race condition for close simultaneous events
        return !action.error ? {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: action.payload,
            isLoading: false,
          },
        } : {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: null,
            error: action.payload,
            isLoading: false,
          },
        };
      }
      
      return state;
    }
  }
}
```


###### 4. Datapoint with cache

``` js
/// Flux Actions (FSA Compliant)

const cachedValue = _.find(store.getState().datapoint.cache, (entry) => isEqual(query, entry.query));

if (!cachedValue) {
  dispatch({
    type: LOAD_DATAPOINT,
    meta: {
      query: query,
    },
    payload: cachedValue,
  });
} else {
  dispatch({
    type: actions.LOADING_DATAPOINT,
    payload: {
      query: query,
    },
  });

  loadDataPoint(query).then((response) => {
    if (!response.error) {
      dispatch({
        type: actions.LOADED_DATAPOINT,
        meta: {
          query: query,
        },
        payload: 'value',
      });
    } else {
      dispatch({
        type: actions.LOADED_DATAPOINT,
        meta: {
          query: query,
        },
        payload: new Error('message'),
        error: true,
      });
    }
  });
}

/// dataPoint in a store (Flux Standard Data Storage Compliant)
reducer(state, action) {
  switch (action.type) {
    case actions.LOAD_DATAPOINT: {
      // if data is cached then this action is triggered
      return {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: action.meta.query,
          data: action.payload,
        },
      };
    }
    case actions.LOADING_DATAPOINT: {
      return {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: action.payload.query,
          isLoading: true,
        },
      };
    }
    case actions.LOADED_DATAPOINT: {
      // if data is cached then this action is triggered and cache is updated
      if (_.isEqual(state.query, action.payload.query) {
        return !action.error ? {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: action.payload,
            cache: [...state.dataPoint.cache, { query: action.meta.query, data: action.payload }],
            isLoading: false,
          },
        } : {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: null,
            error: action.payload,
      cache: [...state.dataPoint.cache, { query: action.meta.query, error: action.payload }],
            isLoading: false,
          },
        };
      }
      
      return state;
    }
  }
}
```


###### 5. Datapoint with queryHistory

``` js
/// Flux Actions (FSA Compliant)

const cachedValue = _.find(store.getState().datapoint.cache, (entry) => isEqual(query, entry.query));

if (!cachedValue) {
  dispatch({
    type: LOAD_DATAPOINT,
    meta: {
      query: query,
    },
    payload: cachedValue,
  });
} else {
  dispatch({
    type: actions.LOADING_DATAPOINT,
    payload: {
      query: query,
    },
  });

  loadDataPoint(query).then((response) => {
    if (!response.error) {
      dispatch({
        type: actions.LOADED_DATAPOINT,
        meta: {
          query: query,
        },
        payload: 'value',
      });
    } else {
      dispatch({
        type: actions.LOADED_DATAPOINT,
        meta: {
          query: query,
        },
        payload: new Error('message'),
        error: true,
      });
    }
  });
}

/// dataPoint in a store (Flux Standard Data Storage Compliant)
reducer(state, action) {
  switch (action.type) {
    case actions.LOAD_DATAPOINT: {
      return {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: action.meta.query,
          prevQueries: [...state.prevQueries, action.meta.query], // stack updated
          data: action.payload,
        },
      };
    }
    case actions.LOADING_DATAPOINT: {
      return {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: action.payload.query,
          prevQueries: [...state.prevQueries, action.meta.query], // stack updated
          isLoading: true,
        },
      };
    }
    case actions.LOADED_DATAPOINT: {
      if (_.isEqual(state.query, action.payload.query) { 
        // only updates if the LOADED action payload corresponds to the query of latest LOADING action. 
        // no race condition for close simultaneous events
        return !action.error ? {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: action.payload,
            cache: [...state.dataPoint.cache, { query: action.meta.query, error: action.payload }],
            isLoading: false,
          },
        } : {
          ...state,
          datapoint: {
            ...state.dataPoint,
            data: null,
            error: action.payload,
            isLoading: false,
          },
        };
      }
      
      return state;
    }
  }
}
```


###### 6. Datapoint with navigable queryHistory

``` js
/// Flux Actions (FSA Compliant)

const cachedValue = _.find(store.getState().datapoint.cache, (entry) => isEqual(query, entry.query));

dispatch({
  type: LOAD_PREV_QUERY_OF_DATAPOINT,
  meta: {
    query: query,
  },
  payload: cachedValue,
});


/// dataPoint in a store (Flux Standard Data Storage Compliant)
reducer(state, action) {
  switch (action.type) {
    case actions.LOAD_PREV_QUERY_OF_DATAPOINT: {
      // only updates if the LOADED action payload corresponds to the query of latest LOADING action. 
      // no race condition for close simultaneous events
      return !action.error ? {
        ...state,
        datapoint: {
          ...state.dataPoint,
          query: _.last(state.dataPoint.prevQueries), // updated with last query
          prevQueries: _.slice(state.dataPoint.prevQueries, 0, -1)], // stack reduced
          nextQueries: [...state.dataPoint.nextQueries, state.dataPoint.query], // stack increased
          data: _.find(
            store.getState().datapoint.cache,
            (entry) => isEqual(_.last(state.dataPoint.prevQueries), entry.query)
          ), // data retrieved from cache
          cache: [...state.dataPoint.cache, { query: action.meta.query, error: action.payload }],
          isLoading: false,
        },
      } : {
        ...state,
        datapoint: {
          ...state.dataPoint,
          data: null,
          error: action.payload,
          isLoading: false,
        },
      };
    }
  }
}
```
