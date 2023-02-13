## MongoDB Collections Viewer 

### This is a simple way to view collections data in the browser 

```bash
mkdir project_directory

cd project_directory

touch app.js
```
Initialise npm
```
npm init
```

Install the packages

```

npm install mongodb ejs
npm install mongoose
npm install express
```

Run the app 

```
node app.js
```

In another terminal, run mongodb

```
mongo
use acebookers_test
```
this should return: 

```
> use acebookers_test
switched to db acebookers_test
```

Since the users collection has already been created, attempting to create a users collection should return an error:
```

> db.createCollection("users")
{
	"ok" : 0,
	"errmsg" : "Collection acebookers_test.users already exists.",
	"code" : 48,
	"codeName" : "NamespaceExists"
}
```
Insert a new document into the users collection

```
> db.users.insertOne({ firstName: "alice", lastName: "smith", email: "alicesmith@email.com", password: "shh", dateOfBirth: '2000-01-01'})
```

The data will be displayed in an unformatted HTML table


```javascript

const express = require('express');
const session = require("express-session");
const mongoose = require('mongoose');
const bodyParser = require('body-parser');
const multer = require('multer');
const upload = multer({ dest: 'uploads/'});

const app = express();

app.use(bodyParser.urlencoded({ extended:true }));

app.use(session({
  secret: 'secret-key',
  resave: false,
  saveUninitialized: true
}));

app.use('/uploads', express.static('uploads'));

mongoose.connect('mongodb://localhost/acebookers_test', { useNewUrlParser: true, useUnifiedTopology: true });

// User schema that creates the users collection in the database
const User = mongoose.model('User', new mongoose.Schema({
  firstName: String,
  lastName: String,
  email: String,
  password: String,
  dateOfBirth: Date
}));

// Post schema that creates a posts collection in the database
const Post = mongoose.model('Post', new mongoose.Schema({
  title: String, 
  content: String, 
  photo: String,
  author: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'User'
  }
}));

// Comment schema that creates the comments collection in the database
const Comment = mongoose.model('Comment', new mongoose.Schema({
  content: String,
  author:{ 
    type: mongoose.Schema.Types.ObjectId, 
    ref: 'User'
  },
  post: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Post'
  }
}));


// get all users
app.get('/', async (req, res) => {
  if (!req.session.user){
    return res.redirect('/login');
  }
  const users = await User.find();
  res.send(`
    <html>
      <head>
        <title>Users</title>
      </head>
      <body>
        <h1>Users</h1>
        <table>
          <tr>
            <th>First Name</th>
            <th>Last Name</th>
            <th>Email</th>
            <th>Password</th>
            <th>Date of Birth</th>
          </tr>
          ${users
            .map(
              (user) => `
            <tr>
              <td>${user.firstName}</td>
              <td>${user.lastName}</td>
              <td>${user.email}</td>
              <td>${user.password}</td>
              <td>${user.dateOfBirth}</td>
            </tr>
          `
            )
            .join("")}
        </table>
        <h2>Add User</h2>
        <form action="/user" method="post">
          <label for="firstName">First Name:</label>
          <input type="text" id="firstName" name="firstName"><br><br>
          <label for="lastName">Last Name:</label>
          <input type="text" id="lastName" name="lastName"><br><br>
          <label for="email">Email:</label>
          <input type="email" id="email" name="email"><br><br>
          <label for="password">Password:</label>
          <input type="password" id="password" name="password"><br><br>
          <label for="dateOfBirth">Date of Birth:</label>
          <input type="date" id="dateOfBirth" name="dateOfBirth"><br><br>
          <input type="submit" value="Submit">
        </form>
      </body>
    </html>
  `);
});

//get user by id 
app.get("/user/:id", async (req, res) => {
  const user = await User.findById(req.params.id);
  res.send(`
    <html>
      <head>
        <title>User</title>
      </head>
      <body>
        <h1>User</h1>
        <table>
          <tr>
            <th>First Name</th>
            <td>${user.firstName}</td>
          </tr>
          <tr>
            <th>Last Name</th>
            <td>${user.lastName}</td>
          </tr>
          <tr>
            <th>Email</th>
            <td>${user.email}</td>
          </tr>
          <tr>
            <th>Password</th>
            <td>${user.password}</td>
          </tr>
          <tr>
            <th>Date of Birth</th>
            <td>${user.dateOfBirth}</td>
          </tr>
        </table>
      </body>
    </html>
  `);
});

app.get('/login', (req, res) => {
  res.send(`
  <html>
      <head>
        <title>Login</title>
      </head>
      <body>
        <h1>Login</h1>
        <form action="/login" method="post">
          <label for="email">Email:</label>
          <input type="email" id="email" name="email"><br><br>
          <label for="password">Password:</label>
          <input type="password" id="password" name="password"><br><br>
          <input type="submit" value="Submit">
        </form>
      </body>
    </html>
  `);
})

app.get('/create-post', (req, res) => {
  if (!req.session.user){
    return res.redirect('/login');
  }
  res.send(`
      <html>
      <head>
        <title>Create Post</title>
      </head>
      <body>
        <h1>Create Post</h1>
        <form action="/create-post" method="post" enctype="multipart/form-data">
          <label for="title">Title:</label>
          <input type="text" id="title" name="title"><br><br>
          <label for="content">Content:</label>
          <textarea id="content" name="content"></textarea><br><br>
          <label for="photo">Photo:</label>
          <input type="file" id="photo" name="photo"><br><br>
          <input type="submit" value="Submit">
        </form>
      </body>
    </html>
  `);
})

app.get('/posts', async (req, res) =>{
  if (!req.session.user){
    return res.redirect('/login');
  }
  const posts = await Post.find().populate('author').populate('comments');
  res.send(`
   <html>
    <head>
      <title>Posts</title>
    </head>
    <body>
      <h1>Posts</h1>
      ${posts.map(post =>`
        <h2>${post.title}</h2>
        <p>${post.content}</p>
        ${post.photo ? `<img src="http://localhost:3000/uploads/${post.photo}" alt="Post photo">`: '' }
				<h3>Comments</h3>
				${post.comments.map(comment => `
					<p>${comment.content}</p>
				`).join('')}
				<form action="/create-comment" method="post">
					<textarea name="content"></textarea>
					<input type="hidden" name="postId" value="${post._id}">
					<button type="submit">comment</button>
				</form>
      `).join('')}
      <button onClick="window.location.href='/logout'">Logout</button>
    </body>
   </html>
  `);
});

app.get('/logout', (req, res) =>{
  req.session.destroy();
  res.redirect('/login');
});



app.post('/create-post', upload.single('photo'), async (req, res) =>{
  if (!req.session.user){
    return res.redirect('/login');
  }
  const post = new Post({
    title: req.body.title, 
    content: req.body.content,
    photo: req.file.filename,
    author: req.session.user._id
  });
  await post.save();
  res.redirect('/');
});

app.post('/create-comment', async(req, res) =>{
	if(!req.session.user) {
		return res.redirect('/login');
	}
	const comment = new Comment({
		content: req.body.content,
		author: req.session.user._id,
		post: req.body.postId
	});
	await comment.save();
	res.redirect('/posts');
});

app.post('/login', async (req, res) =>{
  const user = await User.findOne({ email: req.body.email });
  if (!user){
    return res.send('invalid email or password');
  }
  if (user.password !== req.body.password){
    return res.send('invalid email or password');
  }
  req.session.user = user;
  res.redirect('/');
});

// add new user
app.post('/user', async (req, res)  => {
  const user = new User({
    firstName: req.body.firstName,
    lastName: req.body.lastName,
    email: req.body.email,
    password: req.body.password,
    dateOfBirth: req.body.dateOfBirth
  });
  await user.save();
  res.redirect('/');
});


// Start the Express server
app.listen(3000, () => {
  console.log('Server listening on http://localhost:3000');
});

module.exports = app

```
