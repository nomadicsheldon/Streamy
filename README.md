# Streamy
**A Project which includes implementation of CRUD operations, routing, google auth, use of the portal, and redux.**

The main focus of this project is- CRUD and redux implemenation in project.

Point to note-

---

## Implementation description

Folder Structure -

- api
  - db.json
  - package.json
- client
  - public
    - index.html
  - src
    - actions
      - index.js
      - types.js
    - apis
      - streams.js
    - components
      - streams
        - StreamCreate.js
        - StreamDelete.js
        - StreamEdit.js
        - StreamForm.js
        - StreamList.js
        - StreamShow.js
      - App.js
      - GoogleAuth.js
      - Header.js
      - Model.js
    - reducers
      - authReducer.js
      - index.js
      - streamReducer.js
    - history.js
    - index.js
- rtmp server
  - index.js
  
---

### `api/db.js`
This file will store streams data.

### `api/package.json`
using "json-server" module and changing scripts.
Listening in port 3001 for changes in db.json file.
```javascript
"start": "json-server --watch db.json --port 3001"
```

### `client/public/index.html`
Adding googleAPIs to this project.
```
<script src="https://apis.google.com/js/api.js"></script>
```
Adding one more div just below the `root` for showing a model
```javascript
<div id="model"></div>
```

### `client/src/actions/types.js`
This file exists because string should be same in actions and reducers, so for avoiding string to write in multiple places, we write at this file and import in other places, where we want to use.

### `client/src/actions/index.js`
`signIn`
This action is responsible for signing in and storing `userId`

`signOut`
This action is responsible for signout behavior.

`createStream`
This action is responsible for creating stream entry in DB.
Getting data as `formValues` and posting it in `/streams` URL by spreading technique and adding `userId` as well.
`history.push("/");` using for programatic navigation.

`fetchStreams`
fetching all the streams

`fetchStream`
fetching stream with the `id`

`editStream`
using `patch` for editing stream. We have 2 options `put` or `patch`. `put` will replace full object and `patch` will replace perticular `key` only so it's better to use `patch` in our scenario.

`deleteStream`
deleting data with `id` and return `id` as payload.

### `client/src/apis/streams.js`
setting baseURL as `http://localhost:3001` which will get access of `db.json`. And we can easily do our operations here. 

### `client/components/streams/StreamCreate.js`
adding `createStream` action with this component and calling this `onSubmit`.
```javascript
export default connect(null, { createStream })(StreamCreate);
```

### `client/components/streams/StreamDelete.js`
connecting reducers with component as a prop `stream`. `ownProps` give access to component's props into `mapStateToProps`.
```javascript
const mapStateToProps = (state, ownProps) => {
	return { stream: state.streams[ownProps.match.params.id] };
};
```
adding `fetchStream` and `deleteStream` action with this component and calling `fetchStream` on `componentDidMount`, `deleteStream` on button click.
```javascript
export default connect(mapStateToProps, { fetchStream, deleteStream })(
	StreamDelete,
);
```
We are using `Model` for UI.

### `client/components/streams/StreamEdit.js`
using `StreamForm` for showing form. usinf `_.pick` from lodash for making new object which only contains `title` and `description`.
```javascript
<StreamForm
					initialValues={_.pick(this.props.stream, "title", "description")}
					onSubmit={this.onSubmit}
				/>
```

### `client/components/streams/StreamForm.js`
this component is totally responsible for `form` functionality.
`renderError` responsible for showing errors for the particular text field.
`touched` will check for, is this textfield touched by user or not.

`renderInput` responsible for showing textfield
using spreading for all properties of `input` and assigning `autoComplete` as `off`
```javascript
<input {...input} autoComplete="off" />
```
`validate` will take care of validation in form.
we are giving name to `form`. Why is this neccessary? because it is possible to have multiple forms in a project.
`validate` will take care of all the validation.
```javascript
export default reduxForm({
	form: "streamForm",
	validate,
})(StreamForm);
```

### `client/components/streams/StreamList.js`
This component is responsible for showing all the streams added into project.
`renderAdmin` will check for current user and show UI accordingly.
```javascript
stream.userId === this.props.currentUserId
```
`renderList` will render all list.
`renderCreate` will generate a button, which is responsible for navigating to create stream view.
`mapStateToProps` will connect reducer to component and adding currentUserId, isSignedIn as well.
```javascript
const mapStateToProps = (state) => {
	// Object.values with convert dictionary to array
	return {
		streams: Object.values(state.streams),
		currentUserId: state.auth.userId,
		isSignedIn: state.auth.isSignedIn,
	};
};
```

### `client/components/streams/StreamShow.js`
creating `videoRef` here using `React.createRef()`.
```javascript
constructor(props) {
		super(props);

		this.videoRef = React.createRef();
	}
```
Life cycle methods implementation-
first time user comes to this screen we will do API call and buildPlayer
```javascript
componentDidMount() {
		const { id } = this.props.match.params;
		this.props.fetchStream(id);
		this.buildPlayer();
	}
```
everytime state updates buildplayer
```javascript
componentDidUpdate() {
		this.buildPlayer();
	}
```
cleaning up player while moving out of this screen
```javascript
componentWillUnmount() {
		// clean up video player here
		this.player.destroy();
	}
```
building player here, using `flv` for connecting with live streaming.
```javascript
buildPlayer() {
		if (this.player || !this.props.stream) {
			return;
		}

		const { id } = this.props.match.params;
		this.player = flv.createPlayer({
			type: "flv",
			url: `http://localhost:8000/live/${id}.flv`,
		});
		this.player.attachMediaElement(this.videoRef.current);
		this.player.load();
	}
```
This component is responsible for showing stream and details about that stream.
`mapStateToProps` will map `state.streams[ownProps.match.params.id]` as `stream` into component as props.
```javascript
const mapStateToProps = (state, ownProps) => {
	return { stream: state.streams[ownProps.match.params.id] };
};
```

### `client/components/App.js`
This file takes care of routing.
Normally we use `BrowserRouter` but in current project we want to handle Browsing history by ourself so using -
```javascript
<Router history={history}>
...
</Router>
```
`Switch` will take care of two same type of path like - 
`/streams/new`
`/streams/:id`

### `client/components/GoogleAuth.js`
Doing api call and handling callback for it.
```javascript
componentDidMount() {
		window.gapi.load("client:auth2", () => {
			window.gapi.client
				.init({
					clientId:
						"286726074910-jblf6vu62v2nu101giukhok6o8pksock.apps.googleusercontent.com",
					scope: "email",
				})
				.then(() => {
					this.auth = window.gapi.auth2.getAuthInstance();
					// updating state inside redux store
					this.onAuthChange(this.auth.isSignedIn.get());
					this.auth.isSignedIn.listen(this.onAuthChange);
				});
		});
	}
```
will update state in redux.
```javascript
onAuthChange = (isSignedIn) => {
		if (isSignedIn) {
			this.props.signIn(this.auth.currentUser.get().getId());
		} else {
			this.props.signOut();
		}
	};
```
mapping `state.auth.isSignedIn` as `isSignedIn` 
```javascript
const mapStateToProps = (state) => {
	return { isSignedIn: state.auth.isSignedIn };
};
```

### `client/components/Header.js`
Responsible for showing header.

### `client/components/Model.js`
Use `ReactDOM.createPortal` for creating Model.
Use `onClick={(e) => e.stopPropagation()}` will stop event bubbling up.

### `client/reducers/authReducer.js`
setting `INITIAL_STATE`
using spreading syntax for updating.

### `client/reducers/index.js`
combining all reducers here.

### `client/reducers/streamReducer.js`
implementation of `CRUD` operation here
`_.mapKeys(action.payload, "id")` will make dictionary from array.

### `client/history.js`
creating custom browsing history here.

### `client/index.js`
enabling redux dev tool 
```javascript
const composeEnhancers = window.__REDUX_DEVTOOLS_EXTENSION_COMPOSE__ || compose;
```
`Provider` will connect to redux.

### `rtmpserver/index.js`
`node-media-server` using for media server. Code is taken from documentation of `rtmpserver`.

---
