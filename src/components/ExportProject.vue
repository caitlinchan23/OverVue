<!--
Description:
  Displays Export Project button and allows users to export project
  Functionality includes: export prompts, creating folders and code, exporting images
  -->

<template>
  <q-btn class="export-btn" color="secondary" label="Export Project" @click="exportProject"/>
</template>

<script>
import { mapState } from 'vuex'
const { remote } = require('electron')
import fs from 'fs-extra'
import path from 'path'

export default {
  name: 'ExportProjectComponent',
  methods: {
    showExportDialog () {
      remote.dialog.showSaveDialog({
        title: 'Choose location to save folder in',
        defaultPath: remote.app.getPath('desktop'),
        message: 'Choose location to save folder in',
        nameFieldLabel: 'Application Name'
      },
      result => {
        this.exportFile(result)
      })
    },

    exportProject: function () {
      this.showExportDialog()
    },

    /**
     * @description creates the router.js file
     * argument: location = path to dir
     * invokes: createRouterImports(this.componentMap['App'].children),
     *          createExport(this.componentMap['App'].children)
     *  */
    createRouter (location) {
      fs.writeFileSync(
        path.join(location, 'src', 'router.js'),
        this.createRouterImports(this.componentMap['App'].children) +
          '\nVue.use(Router);\n' +
          this.createExport(this.componentMap['App'].children)
      )
    },

    /**
     * @description import routed components from the /views/ dir
     * @argument: this.componentMap['App'].children
     */
    createRouterImports (appChildren) {
      let str = "import Vue from 'vue'\nimport Router from 'vue-router'\n"
      appChildren.forEach(child => {
        // console.log(`createRouterImports child: ${child}`);
        str += `import ${
          child
        } from './views/${
          child
        }.vue'\n`
      })
      return str
    },

    /**
     * @description creates the `export default` code in <script>
     */
    createExport (appChildren) {
      let str =
        "export default new Router({\n\tmode: 'history',\n\tbase: process.env.BASE_URL,\n\troutes: [\n"
      appChildren.forEach(child => {
        // console.log(`createExport child: ${child}`);
        if (child === 'HomeView') {
          // console.log(`if createExport addChildren child.componentName${child.componentName}`);
          str += `\t\t{\n\t\t\tpath: '/',\n\t\t\tname:'${child}',\n\t\t\tcomponent:${child}\n\t\t},\n`
        } else {
          // console.log(`else createExport addChildren child.componentName${child}`);
          str += `\t\t{\n\t\t\tpath: '/${child}',\n\t\t\tname:'${child}',\n\t\t\tcomponent: ${child}\n\t\t},\n`
        }
      })
      str += `\t]\n})\n`
      return str
    },

    /**
     * @description: creates component code <template>, <script>, <style>
     * invokes writeTemplate, writeScript, writeStyle
     */
    createComponentCode (componentLocation, componentName, children) {
      if (componentName === 'App') {
        fs.writeFileSync(
          componentLocation + '.vue',
          this.writeTemplate(componentName, children) +
            this.writeStyle(componentName)
        )
      } else {
        fs.writeFileSync(
          componentLocation + '.vue',
          this.writeTemplate(componentName, children) +
            this.writeScript(componentName, children) +
            this.writeStyle(componentName)
        )
      }
    },

    // creates assets folder
    createAssetFile (targetLocation, assetLocation) {
      let saved = remote.nativeImage.createFromPath(assetLocation)
      let urlData = saved.toPNG()
      // console.log('urlData is ', urlData);
      fs.writeFileSync(targetLocation + '.png', urlData)
    },

    /**
     * @description helper function for writeTemplate
     * @name writeTemplateTag
     * takes HTML elements from passed in component and adds them to the template string, then returns the string back to writeTemplate
     * @input: componentMap['component'].htmlList[tag elements]
     */
    writeTemplateTag (componentName) {
      // create reference object
      const htmlElementMap = {
        div: ['<div>', '</div>'],
        button: ['<button>', '</button>'],
        form: ['<form>', '</form>'],
        img: ['<img>', ''],
        link: ['<a href="#"/>', ''],
        list: ['<li>', '</li>'],
        paragraph: ['<p>', '</p>'],
        'list-ol': ['<ol>', '</ol>'],
        'list-ul': ['<ul>', '</ul>'],
        input: ['<input />', ''],
        navbar: ['<nav>', '</nav>']
      }

      // function to loop through nested elements
      function writeNested (childrenArray, indent) {
        if (!childrenArray.length) {
          return ''
        }
        let indented = indent + '  '
        let nestedString = ''

        childrenArray.forEach(child => {
          nestedString += indented
          if (!child.text) {
            nestedString += `<${child}/>\n`
          } else {
            if (child.children.length) {
              nestedString += htmlElementMap[child.text][0]
              nestedString += '\n'
              nestedString += writeNested(child.children, indented)
              nestedString += indented + htmlElementMap[child.text][1]
              nestedString += '\n'
            } else {
              nestedString += htmlElementMap[child.text][0] + htmlElementMap[child.text][1] + '\n'
            }
          }
        })
        return nestedString
      }

      // iterate through component's htmllist
      let htmlArr = this.componentMap[componentName].htmlList
      let outputStr = ``
      for (let el of htmlArr) {
        if (!el.text) {
          outputStr += `    <${el}/>\n`
        } else {
          outputStr += `    `
          if (el.children.length) {
            outputStr += htmlElementMap[el.text][0]
            outputStr += '\n'
            outputStr += writeNested(el.children, `    `)
            outputStr += `    `
            outputStr += htmlElementMap[el.text][1]
            outputStr += `  \n`
          } else {
            outputStr += htmlElementMap[el.text][0] + htmlElementMap[el.text][1] + '\n'
          }
        }
      }
      // console.log(`outputStr from writeTemplateTag: ${outputStr}`)
      return outputStr
    },

    /**
     * @description creates the <router-link> boilerplate for /views/components
     * also creates the <template></template> tag for each component
     */
    writeTemplate (componentName, children) {
      let str = ''

      if (componentName === 'App') {
        str += `<div id="app">\n\t\t<div id="nav">\n`
        children.forEach(name => {
          if (name === 'HomeView') {
            str += `\t\t\t<router-link to="/">${
              name
            }</router-link>\n`
          } else {
            str += `\t\t\t<router-link to="/${
              name
            }">${
              name
            }</router-link>\n`
          }
        })
        str += '\t\t\t<router-view></router-view>\n\t\t</div>\n'
      } else {
        str += `<div>\n`
      }
      // writes the HTML tag boilerplate
      let templateTagStr = this.writeTemplateTag(componentName)
      // console.log(`templateTagStr: ${templateTagStr}`)
      return `<template>\n\t${str}${templateTagStr}\t</div>\n</template>`
    },

    /**
     * @description imports child components into <script>
     */
    writeScript (componentName, children) {
      let str = ''
      children.forEach(name => {
        str += `import ${name} from '@/components/${name}.vue';\n`
      })
      let childrenComponentNames = ''
      children.forEach(name => {
        childrenComponentNames += `\t\t${name},\n`
      })
      return `\n\n<script>\n${str}\nexport default {\n\tname: '${componentName}',\n\tcomponents: {\n${childrenComponentNames}\t}\n};\n<\/script>`
    },

    /**
     * @description writes the <style> in vue component
     * if component is 'App', writes css, else returns <style scoped>
     */
    writeStyle (componentName) {
      let style =
        componentName !== 'App'
          ? ''
          : `#app {\n\tfont-family: 'Avenir', Helvetica, Arial, sans-serif;\n\t-webkit-font-smoothing: antialiased;\n\t-moz-osx-font-smoothing: grayscale;\n\ttext-align: center;\n\tcolor: #2c3e50;\n\tmargin-top: 60px;\n}\n`
      return `\n\n<style scoped>\n${style}</style>`
    },

    // creates index html
    createIndexFile (location) {
      let str = `<!DOCTYPE html>\n<html lang="en">\n\n<head>`
      str += `\n\t<meta charset="utf-8">`
      str += `\n\t<meta http-equiv="X-UA-Compatible" content="IE=edge">`
      str += `\n\t<meta name="viewport" content="width=device-width,initial-scale=1.0">`
      str += `\n\t<link rel="icon" href="<%= BASE_URL %>favicon.ico">`
      str += `\n\t<title>vue-boiler-plate-routes</title>`
      str += `\n</head>\n\n`
      str += `<body>`
      str += `\n\t<noscript>`
      str += `\n\t\t<strong>We're sorry but vue-boiler-plate-routes doesn't work properly without JavaScript enabled. Please enable it
      to continue.</strong>`
      str += `\n\t</noscript>`
      str += `\n\t<div id="app"></div>`
      str += `\n\t<!-- built files will be auto injected -->`
      str += `\n</body>\n\n`
      str += `</html>\n`
      fs.writeFileSync(path.join(location, 'public', 'index.html'), str)
    },

    // creates main.js boilerplate
    createMainFile (location) {
      let str = `import Vue from 'vue'`
      str += `\nimport App from './App.vue'`
      str += `\nimport router from './router'`
      str += `\n\nVue.config.productionTip = false`
      str += `\n\nnew Vue({`
      str += `\n\trouter,`
      str += `\n\trender: h => h(App)`
      str += `\n}).$mount('#app')`
      fs.writeFileSync(path.join(location, 'src', 'main.js'), str)
    },

    // create babel file
    createBabel (location) {
      let str = `module.exports = {`
      str += `\n\tpresets: [`
      str += `\n\t\t'@vue/app'`
      str += `\n\t]`
      str += `\n}`
      fs.writeFileSync(path.join(location, 'babel.config.js'), str)
    },

    // create package.json file
    createPackage (location) {
      let str = `{`
      str += `\n\t"name": "vue-boiler-plate-routes",`
      str += `\n\t"version": "0.1.0",`
      str += `\n\t"private": true,`
      str += `\n\t"scripts": {`
      str += `\n\t\t"start": "vue-cli-service serve",`
      str += `\n\t\t"build": "vue-cli-service build",`
      str += `\n\t\t"lint": "vue-cli-service lint"`
      str += `\n\t},`
      str += `\n\t"dependencies": {`
      str += `\n\t\t"vue": "^2.6.6",`
      str += `\n\t\t"vue-router": "^3.0.1"`
      str += `\n\t},`
      str += `\n\t"devDependencies": {`
      str += `\n\t\t"@vue/cli-plugin-babel": "^3.5.0",`
      str += `\n\t\t"@vue/cli-plugin-eslint": "^3.5.0",`
      str += `\n\t\t"@vue/cli-service": "^3.5.0",`
      str += `\n\t\t"babel-eslint": "^10.0.1",`
      str += `\n\t\t"eslint": "^5.8.0",`
      str += `\n\t\t"eslint-plugin-vue": "^5.0.0",`
      str += `\n\t\t"vue-template-compiler": "^2.5.21"`
      str += `\n\t},`
      str += `\n\t"eslintConfig": {`
      str += `\n\t\t"root": true,`
      str += `\n\t\t"env": {`
      str += `\n\t\t\t"node": true`
      str += `\n\t\t},`
      str += `\n\t\t"extends": [`
      str += `\n\t\t\t"plugin:vue/essential",`
      str += `\n\t\t\t"eslint:recommended"`
      str += `\n\t\t],`
      str += `\n\t\t"rules": {},`
      str += `\n\t\t"parserOptions": {`
      str += `\n\t\t\t"parser": "babel-eslint"`
      str += `\n\t\t}`
      str += `\n\t},`
      str += `\n\t"postcss": {`
      str += `\n\t\t"plugins": {`
      str += `\n\t\t\t"autoprefixer": {}`
      str += `\n\t\t}`
      str += `\n\t},`
      str += `\n\t"browserslist": [`
      str += `\n\t\t"> 1%",`
      str += `\n\t\t"last 2 versions",`
      str += `\n\t\t"not ie <= 8"`
      str += `\n\t]`
      str += `\n}`
      fs.writeFileSync(path.join(location, 'package.json'), str)
    },

    exportFile (data) {
      if (data === undefined) return
      if (!fs.existsSync(data)) {
        fs.mkdirSync(data)
        // console.log('FOLDER CREATED!')
        // console.log(`data: ${data}`); // displays the directory path
        fs.mkdirSync(path.join(data, 'public'))
        fs.mkdirSync(path.join(data, 'src'))
        fs.mkdirSync(path.join(data, 'src', 'assets'))
        fs.mkdirSync(path.join(data, 'src', 'components'))
        fs.mkdirSync(path.join(data, 'src', 'views'))
      }

      // creating basic boilerplate for vue app
      this.createIndexFile(data)
      this.createMainFile(data)
      this.createBabel(data)
      this.createPackage(data)

      // exports images to the /assets folder
      for (let [routeImage, imageLocation] of Object.entries(this.imagePath)) {
        if (imageLocation !== '') {
          this.createAssetFile(path.join(data, 'src', 'assets', routeImage), imageLocation)
        }
      };

      // main logic below for creating components
      this.createRouter(data)

      for (let componentName in this.componentMap) {
        // if componentName is a route:
        if (componentName !== 'App') {
          if (this.$store.state.routes[componentName]) {
            // console.log('THIS IS JUST A ROUTE ', componentName)
            this.createComponentCode(
              path.join(data, 'src', 'views', componentName),
              componentName,
              this.componentMap[componentName].children
            )
            // if componentName is a just a component
          } else {
            // console.log('THIS IS JUST A COMPONENT ', componentName)
            this.createComponentCode(
              path.join(data, 'src', 'components', componentName),
              componentName,
              this.componentMap[componentName].children
            )
          }
          // if componentName is App
        } else {
          this.createComponentCode(
            path.join(data, 'src', componentName),
            componentName,
            this.componentMap[componentName].children
          )
        }
      }
    }
  },
  computed: {
    ...mapState(['componentMap', 'imagePath'])
  }
}

</script>

<style>
.export-btn {
  min-height: 10px !important;
  font-size: 11px;
  text-transform: capitalize;
  padding: 3px 8px;
}
</style>
