# Dynamic Environment Variables for Dockerized React Apps

When Dockerizing a service, such as a NestJS project, it's common to create a Docker image that contains the built NestJS code. When this Docker image is run in different deployment environments like Dev, QA, UAT, and Production, we typically provide the necessary environment values through Environment Variables or a .env file. These values are then used by the NestJS project. However, handling environment variables in a React project is not as straightforward.

Let's start by creating a React project to illustrate the issue and how to solve it. You can create a React project using the following command:

```shell
npm create vite@latest react-env
```

Choose 'React' as the framework and 'JavaScript' as a variant. Once the project is created, ensure that all dependencies are installed by running:

```shell
npm install
```

Now, you can launch the React project with:

```shell
npm run dev
```

Your `App.jsx` file will look like this:

```jsx
import { useState } from 'react';
import reactLogo from './assets/react.svg';
import viteLogo from '/vite.svg';
import './App.css';

function App() {
  const [count, setCount] = useState(0);

  return (
    <>
      <div>
        <a href='https://vitejs.dev' target='_blank'>
          <img src={viteLogo} className='logo' alt='Vite logo' />
        </a>
        <a href='https://react.dev' target='_blank'>
          <img src={reactLogo} className='logo react' alt='React logo' />
        </a>
      </div>
      <h1>Vite + React</h1>
      <div className='card'>
        <button onClick={() => setCount((count) => count + 1)}>count is {count}</button>
        <p>
          Edit <code>src/App.jsx</code> and save to test HMR
        </p>
      </div>
      <p className='read-the-docs'>Click on the Vite and React logos to learn more</p>
    </>
  );
}

export default App;
```

Suppose you want the title `Vite + React`, to be loaded from an environment variable. Create a .env file as follows:

**.env**

```
VITE_TITLE=Dockerization
```

Replace `<h1>Vite + React</h1>` with `<h1>{import.meta.env.VITE_TITLE}</h1>` to make use of the `VITE_TITLE` environment variable. Now, the title is sourced from the `.env` file. Let's create two more environment variables.

**.env**

```
VITE_SUB_TITLE=Multi Stage
VITE_ENVIRONMENT=DEVELOPMENT
```

You can display the sub-title and environment below the heading like this:

```jsx
<h1>{import.meta.env.VITE_TITLE}</h1>
<h4>{import.meta.env.VITE_SUB_TITLE}</h4>
<h1>{import.meta.env.VITE_ENVIRONMENT}</h1>
```

Everything seems to be working well. However, when we move to production, it's important to separate the development environment from the production environment. To do this, create another `.env` file specifically for production, `.env.production`, to avoid mixing development environment variables with production.

**.env.production**

```
VITE_TITLE=Dockerization
VITE_SUB_TITLE=Multi Stage
VITE_ENVIRONMENT=PRODUCTION
```

Here, we've set `VITE_ENVIRONMENT` to `PRODUCTION`. Now, let's create a Dockerfile to build the image and a `.dockerignore` file to exclude the `.env` file from being copied into the Docker image.

**.dockerignore**

```dockerfile
# Versioning and metadata
.git
.gitignore
.dockerignore

# Build dependencies
dist
build
node_modules
coverage

# Environment (contains sensitive data)
.env

# Files not required for production
.editorconfig
Dockerfile
README.md
tslint.json
nodemon.json
```

By including `.env` in the `.dockerignore` file, Docker is instructed to exclude the `.env` file when copying files into the Docker image during the build process. Note that we are only excluding the `.env` file and not the `.env.production` file.

Here's what the `Dockerfile` looks like:

**Dockerfile**

```dockerfile
# Stage 1: Build Image
FROM node:18-alpine as build
RUN apk add git
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2, use the compiled app, ready for production with Nginx
FROM nginx:1.21.6-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY /nginx-custom.conf /etc/nginx/conf.d/default.conf
```

We are using a multi-stage build. The Node image is used as a build image, and `RUN npm run build` is used to build the project. After building, we get a `dist` folder, which we copy into the Nginx image using `COPY --from=build /app/dist /usr/share/nginx/html`. Additionally, we copy the `nginx-custom.conf` file from the project into the Docker image.

Normally, when user visit a URL, Nginx will look for a file at the provided route under `/usr/share/nginx/html` and serve it to the user. However, in the case of a Single Page Application (SPA) like React, routes are usually virtual. For instance, when a user visits the `/login` or `/register` route, there are no physical `/login` or `/register` files within the project. These routes are defined virtually in the project using libraries like React Router. Therefore, we need to instruct Nginx to return `index.html` when a route results in a 404 (Not Found) error. `index.html` serves as the entry point for the React app, and this allows React to handle the route.

Here's what the `nginx-custom.conf` file looks like:

**nginx-custom.conf**

```
server {
  listen 80;
  location / {
    root /usr/share/nginx/html;
    index index.html index.htm;
    try_files $uri $uri/ /index.html =404;
  }
}
```

Nginx is configured to listen on port `80`. Now, let's build and run the project.

To build the Docker image, use the following command:

```shell
docker build -t react-env .
```

With the Docker image in hand, you can run it using the following command:

```shell
docker run -p 3000:80 react-env
```

In the above command, we map the Nginx port `80` to the localhost port `3000`. Typically, port `80` and `443` require root access, but by mapping to a different port, you can access the React app at http://localhost:3000. You will see the title `Dockerization`, sub-title `Multi Stage` and environment `PRODUCTION`. These values are sourced from the .env.production file.

However, this setup still leaves us with an issue: how to provide environment values depending on the deployment environment. Let's say we want to see `QA` instead of `PRODUCTION` from the `.env.production` file, when deploying the React app to the QA environment. How can we achieve this?

Providing `VITE_ENVIRONMENT=QA` through the `.env` file or an environment variable `during docker` run won't work. This is because when we build the project, the references to environment variables are replaced with the values provided during the build.

For example, `import.meta.env.VITE_TITLE` is replaced with `Dockerization`, `import.meta.env.VITE_SUB_TITLE` is replaced with `Multi Stage`, and `import.meta.env.VITE_ENVIRONMENT` is replaced with `PRODUCTION`.

After the build, we have hardcoded values: `Dockerization`, `Multi Stage` and `PRODUCTION` because that's what we provided during the build.

## So, what's the solution?

One way is to find all occurrences of the `PRODUCTION` keyword and replace it with the value we want, which is `QA`.

You can use the following command to search through all the files under `/usr/share/nginx/html` (where the project exists). If the `PRODUCTION` keyword is found, it will be replaced with `QA`:

```shell
find /usr/share/nginx/html -type f -exec sed -i "s|PRODUCTION|QA|g" '{}' +
```

However, there's a problem with this approach. If `PRODUCTION` appears anywhere in the project that's not sourced from the `.env.production` file during build, it will also be replaced, which is not the desired outcome.

Additionally, if you have multiple environment variables with the same value during the build, for example:

**.env.production**

```
VITE_TITLE=Dockerization
VITE_SUB_TITLE=Dockerization
VITE_ENVIRONMENT=PRODUCTION
```

You might want the title and sub-title to be `Dockerization` during build, but in QA, you want the title to be `Dockerization for QA` and the sub-title to be `Multi Stage build for QA`.

When you run the following command to replace `Dockerization`:

```shell
find /usr/share/nginx/html -type f -exec sed -i "s|Dockerization|Dockerization for QA|g" '{}' +
```

It will replace both the title and sub-title with `Dockerization for QA`. If you run the next command to replace the sub-title, you'll get `Multi Stage build for QA for QA`, which is not the desired outcome.

**So, what's the solution to these challenges?**

You can use a dummy value during build time. In this case, let's use `VITE_TITLE=MY_APP_TITLE`. This approach not only allows you to replace values based on the environment, but it also makes your solution more flexible for different projects.

Now, your `.env.production` file will look like this:

```
VITE_TITLE=MY_APP_TITLE
VITE_SUB_TITLE=MY_APP_SUB_TITLE
VITE_ENVIRONMENT=MY_APP_ENVIRONMENT
```

Since all environment values now start with `MY_APP_`, you can create a script that looks for all environment variables starting with `MY_APP_` and performs replacements for each of them. Create a shell script called `.env.sh` inside your project:

**env.sh**

```shell
#!/bin/sh
for i in $(env | grep MY_APP_)
do
	key=$(echo $i | cut -d '=' -f 1)
	value=$(echo $i | cut -d '=' -f 2-)
	echo $key=$value
    # sed All files
	# find /usr/share/nginx/html -type f -exec sed -i "s|${key}|${value}|g" '{}' +

    # sed JS and CSS only
    find /usr/share/nginx/html -type f \( -name '*.js' -o -name '*.css' \) -exec sed -i "s|${key}|${value}|g" '{}' +
done
```

To make the script executable, you can update the Dockerfile to run the `env.sh` file when starting the Docker image. Add the following lines at the end of the Dockerfile:

```dockerfile
COPY env.sh /docker-entrypoint.d/env.sh
RUN chmod +x /docker-entrypoint.d/env.sh
```

Nginx Docker images typically look for script files inside the `/docker-entrypoint.d` folder. Any scripts found there are executed before the Nginx service starts. Since you placed your `env.sh` file inside the `/docker-entrypoint.d` folder, your provided environment variables will be replaced as desired before the Nginx service starts, ensuring that your React app functions correctly.

Here's what the final `Dockerfile` looks like:

```dockerfile
# Stage 1: Build Image
FROM node:18-alpine as build
RUN apk add git
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Stage 2, use the compiled app, ready for production with Nginx
FROM nginx:1.21.6-alpine
COPY --from=build /app/dist /usr/share/nginx/html
COPY /nginx-custom.conf /etc/nginx/conf.d/default.conf
COPY env.sh /docker-entrypoint.d/env.sh
RUN chmod +x /docker-entrypoint.d/env.sh
```

Now, when you run the Docker image, you can easily provide the environment variables you need, and they will be used by the React app:

```shell
docker run -p 3000:80 -e MY_APP_TITLE=Dockerization -e MY_APP_ENVIRONMENT=Production react-env
```

This approach allows you to customize your environment variables based on the deployment environment, making your React project more versatile and suitable for various scenarios.
