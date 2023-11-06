# Express & SQL with PostgreSQL: Show & Create

This build is a continuation after completing seed and read lesson.

## Learning Objectives,

By the end of this lesson, you should be able to:

- Create a model file connected to a database that can return all data from a table, a specific row from a table, or a new row in a table.
- Handle asynchronous errors within a controller file and respond from the server appropriately.
- Use asynchronous code within a controller file to respond to client requests with persisted data.

## Getting Started

This is a continuation of the previous lesson.

- Get your Express server running.
- Make sure Postgres is running.
- In the browser go to the index route and check that it works.

## Show

One thing that will be different with your builds is that instead of using the array position, we will use each item's unique `id` generated by Postgres.

For example, looking at the data, we inserted `Apartment Therapy` should have an `id` of `2`.

![](./assets/id-not-array-index.png)

It would be at array position `1` if we would access it by array position. You used array positions earlier for simplicity. However, there is never a guarantee that an item will be in a particular order/array position, and it can change. So using the `id` now that we have a database is critical.

To start working on the show route, open your `queries/colors.js` file, create and include an async arrow function in `module.exports`.

```js
// queries/colors.js

// ONE Color
const getColor = async () => {};

module.exports = {
  getAllColors,
  getColor,
};
```

You will get the `id` from the `req.params` in the show route (in colorsController - see below).

You will use `db.one()` because we expect one row to be returned.

You are passing two arguments to `db.one()`, one is the SQL query, and the second is the value from the request. In this case, it is the `id` of the color we want to show.

It may be tempting to write the SQL query using a JavaScript template literal like this:

```javascript
`SELECT * FROM colors WHERE id=${id};`;
```

However, this is unsafe. Instead, the SQL string will use its own variables that start with a `$`.

```js
"SELECT * FROM colors WHERE id=$1";
```

And a second argument in the `db.one()` function will be used to pass in the actual data.

```js
const oneColor = await db.one("SELECT * FROM colors WHERE id=$1", id);
```

This extra step is to prevent SQL injection attacks by hackers. [Bonus Video on SQL Injection](https://www.youtube.com/watch?v=ciNHn38EyRc5).

This cartoon, is a very quick explanation of a SQL injection:

![](https://imgs.xkcd.com/comics/exploits_of_a_mom.png)

The entire function will include error handling:

```js
const getColor = async (id) => {
  try {
    const oneColor = await db.one("SELECT * FROM colors WHERE id=$1", id);
    return oneColor;
  } catch (error) {
    return error;
  }
};
```

> **Note**: You may also pass in arguments to your SQL query using an object with named keys like so:

```js
await db.one("SELECT * FROM colors WHERE id=$[id]", {
  id: id,
});
```

Knowing this alternate syntax can be helpful as you look at other coding examples. When you work on a project, stick with one syntax type for readability and maintainability.

Import the function to your controller.

```js
// controllers/colorsController.js
const { getAllColors, getColor } = require("../queries/color");
```

Create the show route and test it in the browser/Postman.

```js
// controllers/colorsController.js
// SHOW
colors.get("/:id", async (req, res) => {
  const { id } = req.params;
  res.json({ id });
});
```

Add in the query and add some logic if the query returns no results.

```js
// controllers/colorsController.js
// SHOW
colors.get("/:id", async (req, res) => {
  const { id } = req.params;
  const color = await getColor(id);
  if (color) {
    res.json(color);
  } else {
    res.status(404).json({ error: "not found" });
  }
});
```

## Create

Start by creating the query. Create an async arrow function and be sure to include it in `module.exports`.

```js
// queries/colors.js
const createColor = async (color) => {
  try {
  } catch (error) {
    throw error;
  }
};

module.exports = {
  getAllColors,
  createColor,
  getColor,
};
```

Inserting into the database requires two arguments.

We'll use `db.one()` because we expect one row to be returned. When we return `one`, we get an object. If we return `any` or `many`, we get back an array. This will be important as to how we handle accessing our data in the front-end.

You are passing two arguments to `db.one()`. The first is the SQL query, where the values are represented as `$1`, `$2` etc. In the second, we are passing an array for each value.

| SQL Value | SQL Column  |     Array Value     | Array Index |
| :-------: | :---------: | :-----------------: | :---------: |
|    $1     |    name     |    `color.name`     |      0      |
|    $2     | is_favorite | `color.is_favorite` |      1      |

Set up our basic statement:

```js
// queries/color.js
// CREATE
const createColor = async (color) => {
  try {
    const newColor = await db.one(
      "INSERT INTO colors (name, is_favorite) VALUES($1, $2) RETURNING *",
      [color.name, color.is_favorite]
    );
    return newColor;
  } catch (error) {
    return error;
  }
};
```

Import the function

```js
// controllers/colorsController.js
const { getAllColors, getColor, createColor } = require("../queries/color");
```

Create the show route and test it with Postman.

```js
// CREATE
colors.post("/", async (req, res) => {
  const color = await createColor(req.body);
  res.json(color);
});
```

Example Color:

```js
{
 "name":"fuchsia",
 "is_favorite": "false"
}
```

Remember to set up your Postman request to have the following configuration:

- Route `POST` `/colors`.
- Select: `body`, `raw`, `JSON` from the options.
- Valid JSON.

![](./assets/postman-create.png)

## Error Handling/Validating input

Our users can make a bunch of mistakes.

For example they can forget to provide a color name (or accidentally send the request before completing their input):

```js
{
 "is_favorite": "false"
}
```

When you try to make the POST request with the above JSON, you get a hard-to-read error that is Postgres's default error. You can look for the error code and other details, however, you can also create some logic to avoid this error. You can do this by checking if there is a name and then send back an appropriate status code and a more human-readable error.

It can also be important to include error handling, because some errors will crash your server.

You can add this logic to the route, but then our route starts to become a function that is handling more than one thing: it is validating and sending a response. It would be better to write a separate function that validates it. It also would make sense to put the validation function in its file for better organization.

- `mkdir validations`
- `touch validations/checkColors.js`

```js
// validations/checkColors.js
const checkName = (req, res, next) => {
  console.log("checking name...");
};

module.exports = { checkName };
```

```js
//controller/colorsController.js
const { checkName } = require("../validations/checkColors.js");
```

Add this function as middleware for the create route.

```js
// CREATE
colors.post("/", checkName, async (req, res) => {
```

This request will hang because we are not sending a response.

```js
const checkName = (req, res, next) => {
  if (req.body.name) {
    console.log("name is ok");
  } else {
    res.status(400).json({ error: "Name is required" });
  }
};
```

Ok, you get the error message. But how do you return to the route if you enter a name now?

You use the `next()` function.

```js
const checkName = (req, res, next) => {
  if (req.body.name) {
    return next();
  } else {
    res.status(400).json({ error: "Name is required" });
  }
};
```

Let's try again.

```js
{
 "name": "cornflowerblue",
 "is_favorite": "true"
}
```

#### Another User Error

You can end up where the user/front-end app does not give a Boolean value.

```js
{
 "name":"honeydew",
 "is_favorite": "maybe"
}
```

PostgreSQL is strict. It will not accept `maybe`, it will reject the POST request and send back a long error. Again, you can avoid this error by doing some error handling.

```js
// validations/checkColors/js
const checkBoolean = (req, res, next) => {
  if (req.body.is_favorite) {
    next();
  } else {
    res.status(400).json({ error: "is_favorite must be a boolean value" });
  }
};

module.exports = { checkBoolean, checkName };
```

Don't forget to add this to the `colorsController.js`.

```js
const { checkBoolean, checkName } = require("../validations/checkColors.js");


// Further down
colors.post("/", checkBoolean, checkName, async (req, res) => {
```

Hmmm, not quite right. Our `if` statement checks if the value is truthy, not if it is an actual boolean value. How can we check if the `req.body.is_favorite` value is a boolean value?

If you don't know, go ahead and google it.

<details><summary>Possible Solution</summary>

```js
const checkBoolean = (req, res, next) => {
  const { is_favorite } = req.body;
  // account if string or undefined
  // the value false will evaluate to false
  // therefore check if typeof is boolean as well
  if (
    is_favorite == "true" ||
    is_favorite == "false" ||
    is_favorite == undefined ||
    typeof is_favorite == "boolean"
  ) {
    next();
  } else {
    res.status(400).json({ error: "is_favorite must be a boolean value" });
  }
};
```

</details>

## Save it

- `git add -A`
- `git commit -m 'show and create complete'`.

## Reference

[See completed build](https://github.com/pursuit-curriculum-resources/pre-reading-express-sql-seed-read/tree/show-create)