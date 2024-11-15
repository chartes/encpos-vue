# encpos-vue

**dev dependencies**:

![Node](https://img.shields.io/badge/node-22.9-blue?style=for-the-badge&logo=Node.js)

[![package - vue](https://img.shields.io/github/package-json/dependency-version/chartes/encpos-vue/vue/master?logo=vue.js&logoColor=white)](https://www.npmjs.com/package/vue)
[![package - vite](https://img.shields.io/github/package-json/dependency-version/chartes/encpos-vue/dev/vite/dev?logo=vite&logoColor=white)](https://www.npmjs.com/package/vite)
[![package - eslint](https://img.shields.io/github/package-json/dependency-version/chartes/encpos-vue/dev/eslint/dev?logo=eslint&logoColor=white)](https://www.npmjs.com/package/eslint)

Frontend application to browse and search the ENC Thesis Abstracts ([Positions de thèses](https://theses.chartes.psl.eu/)).

>:warning: (Install) / Launch first the DTS API and Elasticsearch :  
> See the [API Readme](https://github.com/chartes/encpos-app)

## Project setup
### Clone the GitHub repository:  
in a local folder dedicated to the project
  ```bash
  git clone https://github.com/chartes/encpos-vue.git
  ```

From the app folder (`cd path/to/encpos-vue`)
### Install
```
yarn
```

### Compiles and hot-reloads for development

For a local development server, reads `.env.development` variables:

```
yarn serve
```

### Compiles and minifies for staging

For a staging server (preproduction), reads `.env.staging` variables:

```
yarn build:staging
```

### Compiles and minifies for production

For a production server, reads `.env.production` variables:

```
yarn build:prod
```

### Run preview server

After a `yarn build:prod` or `yarn build:staging` you can run a server to serve the produced `dist` folder with:

```
yarn preview
```

for production or

```
yarn preview --mode=staging
```

for staging.


### Lints and fixes files
```
yarn lint
```

### Customize configuration
See [Configuration Reference](https://cli.vuejs.org/config/).
