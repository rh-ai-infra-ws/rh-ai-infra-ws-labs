## OpenShift AI Infrastructure Enablement Instructions Repo

The repo holds the content for the OpenShift AI Infrastructure Enablement.

`docs` - contains the student and teacher guides for the technical exercises as well as the classroom
activities. 
The `slides/content` are written in markdown and automatically published to the site when pushed to main.

### 🏃‍♀️ Running the docs & slides site locally

To launch the slides, ensure you have NodeJS installed or run it in a NodeJS container if you prefer.

```shell
npm i -g docsify-cli@4.4.3
docsify serve ./docs
```

* Open the browser to http://localhost:3000 to view the tech exercise.
* Open the browser to http://localhost:3000/slides to view the slides.

## 🎃 Contribution

Pull requests welcome 🎃. Please 🙏, review 👀 the [Contribution Guide](./CONTRIBUTING.md) to become a contributor.

Changes approved and pushed to main will automatically be published to the docs site.