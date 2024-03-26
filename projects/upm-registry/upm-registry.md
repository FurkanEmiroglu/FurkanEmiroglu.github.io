# Creating a package registry for Unity-Package-Manager

## Prerequisites

1. Your root folder must contain a package.json file. You can see an example below.
2. Your root folder must have an .asmdef file. Otherwise UPM will ignore your package after importing.
3. I'ill store my packages on npmjs.com but you can setup your own registry too. I can recommend Verdaccio for custom registries.

### Example package.json file

```json
{
  "author": "Furkan Emiroglu",
  "name": "com.femiroglu.tutorialpackage",
  "displayName": "Tutorial Package",
  "version": "1.0.0",
  "unity": "2021.3",
  "description": "Just a tutorial package",
  "repository": {
    "type": "git",
    "url": "github:furkanemiroglu/nodejs-registry-tutorial-4u"
  },
  "documentationUrl": "https://github.com/FurkanEmiroglu/NodeJS-Registry-Tutorial-4U#readme",
  "changelogUrl": "https://github.com/FurkanEmiroglu/NodeJS-Registry-Tutorial-4U#readme",
  "licensesUrl": "https://github.com/FurkanEmiroglu/NodeJS-Registry-Tutorial-4U#readme",
  "dependencies": {}
}                           
```



- If you want to automate the release process, create an account with a token in npmjs.com
- Create a Github repository for your package and push your root folder.
- Open your repository settings and visit Secrets and Variables/Actions
- Click "New Repository Secret" and add your NPM token a name "NPM_TOKEN"
- Create the directory .github/.workflows under your root folder.
- Create a publish.yml file and fill inside with the code below:
- Don't forget to customize the yml file with your own git configs.



```yml
name: Publish Package to npmjs
on:
  push:
    branches:
      - main
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      # Setup .npmrc file to publish to npm
      - uses: actions/setup-node@v3
        with:
          node-version: '16.x'
          registry-url: 'https://registry.npmjs.org'
      - run: git config --global user.email "emiroglu.furkan@gmail.com"
      - run: git config --global user.name "Furkan Emiroglu[Bot]"
      - run: npm version patch -m "v%s"
      - run: VERSION=$(node -p "require('./package.json').version")
      - run: git tag ${VERSION}
      - run: git push "https://$GITHUB_ACTOR:${{ secrets.GITHUB_TOKEN }}@github.com/$GITHUB_REPOSITORY.git" --follow-tags
      - run: git push "https://$GITHUB_ACTOR:${{ secrets.GITHUB_TOKEN }}@github.com/$GITHUB_REPOSITORY.git" --tags
      - run: npm publish --access public
        env:
          NODE_AUTH_TOKEN: ${{ secrets.NPM_TOKEN }}                  

```





### Result

If everything goes fine, anyone can now add your package scope to their Package Manager registries and access your packages.



- Adding registry to package manager

![Package Manager](https://raw.githubusercontent.com/FurkanEmiroglu/FurkanEmiroglu/main/Package%20Manager.jpg)



- Installing packages

  ![Package Manager](https://github.com/FurkanEmiroglu/FurkanEmiroglu/blob/main/PackageManagerResult.png?raw=true)

  
