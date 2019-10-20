---
layout: post
title: Sharing the context with models in Express
image: /assets/posts/context.jpg
description: How to share the request context with models in Express, so the model can use it to filter the data for the current user.
---
Let's imagine that you are building a SaaS platform where companies can manage information about their projects. You need to expose an API where users can see the projects from their company, but your clients don't want to share their projects information with anyone else that is not an employee.

So you start creating a new Express app. First of all, you create a middleware to authenticate the user.
```js
module.exports = (req, res, next) => {
  const authorization = req.get('Authorization');
  if (!authorization) {
    next('No Authorization header');
  }
  let { userId } = decodeToken(authorization);
  let user = UserModel.findById(userId);
  req.context = {
    user,
  };
  next();
};
```
This middleware just verifies the token, extracts the `userId` from it, gets the user from the model and saves the `user` in a **context** object inside the request object. Doing this we are able to access the user from the controllers later on.

Now that we have our API secured, let's create the first endpoints:

```js
router
  .route("/projects")
  .get(projectsController.getProjects)
  .post(projectsController.postProject);
```

Next step, we need to create our controller :)

```js
const getProjects = (req, res) => {
  const { user: currentUser } = req.context;
  const projects = ProjectModel.find(currentUser.company);
  res.json({projects});
}

const getProjectById = (req, res) => {
  const { user: currentUser } = req.context;
  const { id: projectId } = req.params;

  const project = ProjectModel.findById(projectId, currentUser.company);

  if (!project) {
    return res.status(401)
  }

  res.json({project})
};
```
Simple right? We've just created two functions that will call the model to retrieve the required data. As you can see, we are using the `user` from the context to filter the data, so we don't expose projects from other companies.

Let's see the last file, the model:
```js
class Project {
  static find(company) {
    return PROJECTSDATA
      .filter(project => project.company === company)
      .map(projectData => new Project(projectData));
  }

  static findById(id, company) {
    const projectData = PROJECTSDATA.find(project => (
      project.id === id &&
      project.company === company
    ));
    return new Project(projectData)
  }
}
```
Everything looks fine until now, you have the code [here](https://github.com/fernandocalsa/express-context/tree/no-context). The model just exposes two functions to retrieve the projects, filtering by a company. For the sake of simplicity we save all the projects in `PROJECTSDATA`.

So that's it, right? We have an API that exposes the projects from different companies and they are only visible to their employees.

Well, I see a small problem here, developers have to pass down the `company` id of the current user from the controllers to the models all the time. Yeah, it is just an argument, but it can create security issues in the future if a developer forgets to filter the projects by the company. Wouldn't it be nice if the model would have access to the context? so the developer just has to do `ProjectModel.find()` and the model will be responsible for filtering the data for us. This is what I'll try to solve here.

## Getting access to the context

So, the idea is that the model has access to the context, and from here to the current user and his company. The approach I like to take is creating a new set of models for each request, injecting them into the context and injecting the context into the model. I create a new set of models so I make sure that we don't change the context during the execution of one request. If we just add the context to the model at the beginning of the request, whenever a new request starts will update the context for the models, so if the previous request didn't finish it will use a wrong context. With this approach, we keep all the information in the request object.

Let's start, we have to change what the model file is exporting, now we have to export a factory that generates a new model every time, this is as easy as this:

```js
// Before
module.exports = Project;
// Factory
module.exports = () => class Project {
  // all the logic goes here
};
```

Instead of exporting the model, we just export a function that returns the model class, a new one every time we call the function.

Now, we need a new middleware that will inject the model into the context and adds the context into the model, something like this:

```js
const projectFactory = require("../models/project");

module.exports = (req, res, next) => {
  const Project = projectFactory();
  Project.prototype._context = req.context;
  Project._context = req.context;
  req.context.models = { Project };
  next();
};
```

We generate a new model for every request, and inject the context in it, both in the class and in the prototype so we have access to it all the time.

Now the model methods don't need to receive the company id through the arguments, so we can remove it and get it from the context:

```js
static find() {
  const companyId = this._context.user.company;
  const { Project } = this._context.models;
  return PROJECTS
    .filter(project => project.company === companyId)
    .map(projectData => new Project(projectData));
}

static findById(id) {
  const companyId = this._context.user.company;
  const { Project } = this._context.models;
  const projectData PROJECTS.find(project => (
    project.id === parseInt(id) &&
    project.company === companyId
  ));
  return new Project(projectData);
}
```

And finally, as we have now the model in the request, our controller doesn't need to require the model anymore, and can get it from the context, and of course, it doesn't need to pass the company to the model anymore!

```js
const getProjects = (req, res) => {
  const { Project } = req.context.models;

  const projects = Project.find();

  res.json({
    projects
  });
};

const getProjectById = (req, res) => {
  const { id: projectId } = req.params;
  const { Project } = req.context.models;

  const project = Project.findById(projectId);

  if (!project) {
    return res.status(401).json({
      error: "project not found"
    })
  }

  res.json({
    project
  })
};
```

From now on, if the model is well implemented, developers don't have to filter the data anymore, and you'll be sure that everything is filtered by the user's company.

This allows us to move some logic to the model, for example, when we need to create a new project, we would use a post request to `/projects`, the request just needs to send the name of the project, and the model will insert the user who created it and the company. The controller function would be something like this:

```js
const postProject = (req, res) => {
  const { name } = req.body;
  const { Project } = req.context.models;

  const project = new Project({name});
  project.save();

  res.json({
    project
  });
};
```

And the model `save` method would be something like this:

```js
save() {
  this.company = this._context.user.company;
  this.createdBy = this._context.user.id;
  // save to the database
}
```

This approach can be used not only for models but also for many other functions that need access to the context, for example, a logger function that needs to log the request id.

You can see the repository with all the code and a few more endpoints [here](https://github.com/fernandocalsa/express-context)

Thanks for reading this post, and please, let me know in the comments what do you think, or if you found a better approach.
